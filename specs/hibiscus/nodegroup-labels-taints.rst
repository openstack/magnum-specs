..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


===============================
Per-nodegroup labels and taints
===============================

Magnum users need a first-class way to apply Kubernetes node labels and
taints to the nodes that make up a worker nodegroup. This is currently
impossible without post-create ``kubectl`` intervention, and any such
intervention is lost on scale-out because new machines join with the
defaults.

Problem description
===================

A common operational pattern is to dedicate worker pools to specific
workloads, for example a GPU pool, a memory-optimized pool, or a pool
reserved for system add-ons. Two Kubernetes primitives express this:

* **Node labels** allow workloads to target a pool via ``nodeSelector``,
  ``nodeAffinity``, or topology spreading.
* **Node taints** reserve a pool by repelling pods that do not carry a
  matching toleration.

Magnum NodeGroup objects already have a ``labels`` field, but it stores
Magnum-level configuration (for example ``auto_scaling_enabled``,
``availability_zone``) and is consumed by drivers as Magnum config rather
than propagated to Kubernetes. There is no field for Kubernetes-side
labels and no field for taints at all. As a result, operators must:

1. Wait for nodes to join the cluster.
2. Run ``kubectl label`` and ``kubectl taint`` against each node.
3. Repeat the operation manually whenever the nodegroup scales out or rolls.

This makes pool dedication brittle and undoes itself silently during
autoscaling events.

Proposed change
===============

Two new optional fields are added to the NodeGroup object:

* ``node_labels`` — a dict of ``{string: string}``. Each entry is intended
  to be applied to every node in the group as a Kubernetes node label. The
  API rejects labels the kubelet is not permitted to self-apply (restricted
  ``kubernetes.io/`` and ``k8s.io/`` namespace labels outside the
  kubelet-allowed prefixes/set), because such a label would stop the
  kubelet from starting — see Validation below.
* ``node_taints`` — a list of ``{key, value, effect}`` dicts. ``key`` is
  required, ``value`` may be empty, ``effect`` must be one of
  ``NoSchedule``, ``PreferNoSchedule``, ``NoExecute``.

Both are nullable. Existing nodegroups continue to work unchanged.

Scope is limited to **worker** nodegroups. Control plane nodes have their
own kubeadm-managed taint (``node-role.kubernetes.io/control-plane``) and
allowing arbitrary user taints on the control plane risks breaking
cluster bootstrap; that is left to a future spec.

This is enforced by the API: ``node_labels`` and ``node_taints`` are
rejected on any non-worker nodegroup. Creating a control plane (master)
nodegroup is already unsupported, and a ``PATCH`` that sets either field
on the cluster's default control plane nodegroup is rejected with a
``409 Conflict``.

Magnum stores the two fields and passes them through to whichever driver
handles the cluster. Honouring them is the driver's responsibility:
drivers that can apply node labels and taints at node-registration time
should do so; drivers that cannot are free to ignore the fields or to
fail validation explicitly. The reference behaviour is to apply the
labels and taints when each node joins the cluster, so that scale-out
and rolling replacements preserve them without further user action.

Data flow
---------

1. ``openstack coe nodegroup create --node-labels k=v --node-taints
   k=v:NoSchedule …`` sends both fields to the Magnum API. Each taint is
   given as ``KEY[=VALUE]:EFFECT`` (``VALUE`` is optional, e.g.
   ``dedicated:NoSchedule``). Both ``--node-labels`` and ``--node-taints``
   are repeatable, and multiple entries may also be packed into a single
   flag separated by ``,`` or ``;`` — so
   ``--node-taints a=1:NoSchedule,b:NoExecute`` and
   ``--node-taints a=1:NoSchedule --node-taints b:NoExecute`` are
   equivalent. ``node_labels`` follow the existing ``--labels`` syntax.
2. The Magnum API validates and persists them on the NodeGroup row.
3. The conductor passes the NodeGroup to the configured driver.
4. The driver translates the fields into whatever its underlying
   provisioning mechanism supports for node-registration-time label and
   taint application.

REST API
--------

The fields are surfaced symmetrically on the ``NodeGroup`` resource:

* ``POST /clusters/{id}/nodegroups`` accepts both as optional body fields.
* ``PATCH /clusters/{id}/nodegroups/{ng}`` accepts both via JSON Patch on
  ``/node_labels`` and ``/node_taints``.
* ``GET`` returns both.

A ``replace`` on ``/node_labels`` or ``/node_taints`` overwrites the whole
field — the supplied value becomes the complete new set; entries are not
merged with the existing ones. To add to or remove from the current set,
read the nodegroup, modify the value, and send the full desired set. The
``openstack coe nodegroup update`` command sends the complete set in this
way.

Update semantics:

When ``node_labels`` or ``node_taints`` change, the driver is responsible
for converging **all** worker nodes in the group onto the new set — not
just nodes that join afterwards. Magnum stores the desired state and the
driver reconciles existing nodes towards it. For the reference Cluster API
driver this means the affected ``MachineDeployment`` is rolled so every
node ends up with the updated labels and taints; nodes added later by
scale-out naturally pick up the current set as well.

Validation (each failure produces ``409 Conflict`` with a descriptive
message):

* ``node_taints`` entries missing ``key`` or with an unrecognized ``effect``
  are rejected.
* ``node_labels`` keys in the restricted ``kubernetes.io/`` and ``k8s.io/``
  namespaces are rejected unless they use a kubelet-allowed prefix (e.g.
  ``node.kubernetes.io/``) or are one of the kubelet-allowed labels —
  otherwise the kubelet refuses to start and the node group is left
  unschedulable. Label strings outside those namespaces are forwarded
  as-is; Kubernetes itself rejects any remaining malformed labels at
  node-registration time.
* ``node_labels`` or ``node_taints`` on a non-worker nodegroup are rejected
  (see `Proposed change`_).

This is an APIImpact change.

Alternatives
------------

* **Reuse the existing ``labels`` field for Kubernetes node labels.**
  Rejected: ``labels`` already carries Magnum configuration and is
  populated by clients today. Reinterpreting it would either apply
  Magnum config keys (e.g. ``auto_scaling_enabled=true``) as node labels,
  or require an opt-in prefix scheme that is awkward and easy to misuse.

* **Apply labels and taints post-join via a controller-side hook.**
  Rejected: requires running an extra controller against the workload
  cluster, and inherits the same drift problem as manual ``kubectl`` —
  newly joined nodes would briefly exist without labels/taints, which
  defeats the purpose of taints in particular.

* **Encode taints in ``node_labels`` using a sentinel prefix.**
  Rejected: taints are a distinct primitive with three fields. Squeezing
  them into a flat string map sacrifices schema clarity for no benefit.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Sam Morrison <sorrison@gmail.com>

Milestones
----------

Target Milestone for completion:
  hibiscus

Work Items
----------

* Add ``node_labels`` and ``node_taints`` fields to
  ``magnum.objects.NodeGroup`` (object VERSION bump 1.1 → 1.2) and to the
  SQLAlchemy model with an alembic migration.
* Expose both fields on the v1 NodeGroup REST controller, including PATCH
  support and taint-effect validation. Bump the REST API microversion.
* Add ``node_labels`` / ``node_taints`` to ``CREATION_ATTRIBUTES`` in
  ``python-magnumclient`` and add ``--node-labels`` and ``--node-taints``
  CLI flags to ``openstack coe nodegroup create`` and ``update``.
* Update each driver that supports worker nodegroups to honour the new
  fields, or to document explicitly that it does not.
* Documentation: update Magnum admin and user docs to describe the new
  fields and their semantics.

* Magnum change - https://review.opendev.org/c/openstack/magnum/+/988899
* Magnumclient change - https://review.opendev.org/c/openstack/python-magnumclient/+/988900

Dependencies
============

- No new library dependencies.

Security Impact
===============

A user with permission to manage nodegroups already controls the size,
flavor, and image of a worker pool. The ability to taint a pool does not
escalate any privilege but can be used to make pods unschedulable on that
pool. This is a per-cluster effect, scoped to the project that owns the
cluster, and does not affect tenant isolation or the Magnum control plane.

Node labels are user-defined strings exposed on the Kubernetes API. They
are not used by Magnum for authorization decisions, so collisions with
Magnum-internal labels are not a security concern.
