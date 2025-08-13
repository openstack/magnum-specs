..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


========
Reloader
========

Problem description
===================

Some workloads that rely on up-to-date Secrets and ConfigMaps may fail when
these resources are updated or changed. One such example is the Cinder CSI
plugin which mounts the `cloud-config` secret and uses the contained
application credential for authentication which is required for most, if not
all, volume management operations. If the application credential is manually
rotated, the Cinder CSI plugin would need to be reloaded to prevent
authentication failure.

In this spec, we propose adding Reloader[1] as an optional cluster addon
to help circumvent these cases.

Proposed change
===============

The proposed change includes:

1. The addition of Reloader as a cluster addon within capi-helm-charts.
2. Annotation of OpenStack services that rely on Secrets and ConfigMaps
   deployed alongside them.
3. The addition of a configuration option within the magnum-capi-helm driver
   that enables the deployment of Reloader to workload clusters.

Alternatives
--------------

The alternative is to continue requiring an administrator to manually reload
affected workloads when mounted Secrets and ConfigMaps are changed.

Implementation
==============

The implementation involves the addition of Reloader into capi-helm-charts,
with a set of default values appropriate for workload clusters initialised with
Magnum. Additionally, pods that should be reloaded with Reloader must be
annotated appropriately.

capi-helm-charts
----------------

Reloader will be added to capi-helm-charts as a cluster addon and toggled,
similarly to other addons, using the `reloader.enabled` value. Enabling
Reloader will install the upstream Helm chart from the Stakater repository[2]
to workload clusters and annotate affected OpenStack pods appropriately for
Reloader's named resource reload mechanism[3]. These pods are namely
cinder-csi and openstack-cloud-controller-manager. Initially, this spec
proposes that pod restarts should only be invoked when the cloud-config Secret
changes.

Considering that end-users may want to use Reloader for their own workloads,
we can reduce the scope of resources this release of Reloader has access to
by setting `reloader.watchGlobally` to `false` and deploying Reloader to the
`openstack-system` namespace as all affected pods with the annotation exist
there. This options creates a `Role` and associated `RoleBinding` for Reloader
instead of a `ClusterRole` and `ClusterRoleBinding`.

magnum-capi-helm
----------------

Reloader should not be enabled in the chart by default, but be configurable via
the magnum-capi-helm driver if desired. A small change to the driver to add
this option will be required.

Assignee(s)
-----------

Primary assignee:

* northcottmt

With support from:

* dalees

Milestones
----------

* Add Reloader as a cluster addon within capi-helm-charts
* Modify existing OpenStack releases with the appropriate annotations
* Add configuration options to the magnum-capi-helm driver to enable Reloader
  in workload clusters

Target milestone for completion:
  Flamingo

Work Items
----------

Complete all the above milestones.

Dependencies
============

None

Security Impact
===============

This spec proposes that an externally maintained service is added to workload
clusters which may have security implications that should be taken into
consideration. Attention must be given to what changes are introduced in each
update, particularly in what resources Reloader has access to. Reloader should
not be granted access to namespaces beyond `openstack-system`, which it would
have if installed with default values. Appropriaoate RBAC mitigates the
potential for undesirable access to other namespaces.

References
==========

[1] https://github.com/stakater/Reloader
[2] https://github.com/stakater/Reloader/tree/master/deployments/kubernetes/chart/reloader
[3] https://github.com/stakater/Reloader?tab=readme-ov-file#2--named-resource-reload-specific-resource-annotations
