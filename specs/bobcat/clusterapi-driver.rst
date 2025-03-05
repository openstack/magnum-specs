..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================
Cluster API driver
==================

https://storyboard.openstack.org/#!/story/2009780

Problem description
===================

The existing Magnum support for Kubernetes has proven hard to
maintain, particularly around: upgrade [#]_, auto healing, auto scaling,
and keeping track of host operating system changes.

At the same time, Kubernetes Cluster API [#]_ has gained a lot of
traction as a way to create, update and upgrade Kubernetes clusters.
In particular, there is an active community keeping the OpenStack
connector working:
https://github.com/kubernetes-sigs/cluster-api-provider-openstack

In addition, the community maintains a set of image build pipelines
to create kubeadm powered clusters across a range of operating
systems. These have been shown to work well with OpenStack:
https://image-builder.sigs.k8s.io/capi/providers/openstack.html

There is a strong case for a native OpenStack API that
gives you a multi-tenant cluster as a service that is well
inegrated with keystone.
Currently that isn't expected to be Cluster API CRDs directly,
although that may change. See the Cluster API book:
https://cluster-api.sigs.k8s.io/user/personas.html#service-provider-kubernetes-as-a-service

In this spec, we propose adding a new Magnum driver that allows
operators to offer Cluster API managed Kubernetes clusters, via
the existing Magnum API.

Proposed change
===============

For details on Cluster API terminology please see:
https://cluster-api.sigs.k8s.io/user/concepts.html

The new Cluster API driver will not make use of Heat, instead it
will create clusters by interacting with a Kubernetes Cluster that
has been configured as a Cluster API Management Cluster.

For CI jobs, the Magnum devstack plugin will need to create a k8s
cluster with all the required Cluster API components installed.
Then configuring the new Magnum driver to have access to that cluster.
Then we need the Tempest plugin to create the correct sort of template
that makes use of the new driver.

The intention for the new driver is to make it compatible with the
Magnum REST API. No changes to the REST API are anticipated.
For example, Terraform's Magnum support should just work without
any changes.

Bootstrapping and managing the external management cluster and the
Cluster API service is out of scope.  This driver would depend on
a Cluster API being available in an external kubernetes cluster.
For example, the devstack plugin could create a single node k3s
cluster and install what is required in there. Those details should
also be docuemented, such that operators and installs know exactly
what is required.
Longer term, we might choose to do something different, but this
seems like a pragmatic way to move forward quickly.

Feature comparison
------------------

A feature comparison shows that Cluster API has much in common with Magnum,
but introduces some innovations that would be beneficial to Magnum.

+--------------------------+----------------------+---------------------------+
| Feature                  | Magnum               | Cluster-API               |
+==========================+======================+===========================+
| Cloud Provider OpenStack | Installed by default | Installed via Helm charts |
+--------------------------+----------------------+---------------------------+
| Host OS support          | Fedora Core OS       | Typically Ubuntu 20.04    |
|                          |                      | LTS, various choices      |
|                          |                      | supported by the image    |
|                          |                      | builder [#]_.             |
+--------------------------+----------------------+---------------------------+
| External dependencies    | Heat, Keystone,      | External Kubernetes,      |
|                          | others.              | Keystone, others.         |
+--------------------------+----------------------+---------------------------+
| Supported CNIs           | Flannel, Calico.     | Further options available,|
|                          |                      | eg Cilium.                |
+--------------------------+----------------------+---------------------------+
| Cinder CSI               | Managed in Magnum    | Installed via Helm charts.|
+--------------------------+----------------------+---------------------------+
| Prometheus monitoring    | Installed by default.| Installed via Helm charts.|
+--------------------------+----------------------+---------------------------+
| Ingress controllers      | Octavia, Traefik,    | Installed via Helm charts.|
|                          | Nginx.               |                           |
+--------------------------+----------------------+---------------------------+
| Delegated authorisation  | Keystone trust.      | Application credential.   |
+--------------------------+----------------------+---------------------------+
| Kubernetes CRD API       | None.                | Helm, flux, argo, etc.    |
+--------------------------+----------------------+---------------------------+
| In-place upgrades        | Partial - depends on | Various supported         |
|                          | driver.              | strategies, build in      |
|                          |                      | infrastrucutre agnostic   |
|                          |                      | code. Defaults to rolling |
|                          |                      | upgrade (like Magnum).    |
+--------------------------+----------------------+---------------------------+
| Self-healing             | Partial / uncertain. | Supported with            |
|                          |                      | infrastrcuture agnostic   |
|                          |                      | code via reconciliation   |
|                          |                      | loop.                     |
+--------------------------+----------------------+---------------------------+
| Auto-scaling             | Supported for        | Supported with            |
|                          | default node group.  | infrastructure agnostic   |
|                          |                      | code.                     |
|                          |                      |                           |
|                          |                      | Auto-scaling to zero      |
|                          |                      | supported in CAPI, but    |
|                          |                      | CAPO only supports        |
|                          |                      | scaling to one node per   |
|                          |                      | group currently.          |
+--------------------------+----------------------+---------------------------+
| Multiple node groups     | Supported.           | Supported, with no        |
|                          |                      | default group.            |
+--------------------------+----------------------+---------------------------+
| Additional networks      | Supported (review    | Supported                 |
|                          | pending)             |                           |
+--------------------------+----------------------+---------------------------+
| New Kubernetes versions  | Test burden on Magnum| Test burden split between |
|                          | entirely.            | Cluster API for           |
|                          |                      | Kubernetes, Magnum for    |
|                          |                      | infrastructure provision. |
+--------------------------+----------------------+---------------------------+

Alternatives
------------

We could attempt to maintain and fix the existing k8s support, but
its getting harder and harder.

Implementation
==============

The initial target for the driver is to support the following operations,
likely a new patch adding a new operation in roughly the following order:

* define templates that map to different k8s versions
  (i.e. image_id from template)
* create a cluster and delete a cluster
  (flavor_id and node counts from default node group)
* coe credentials to help users create a valid kubeconfig
* Devstack install (manually) passing sonoboy conformance tests
* Sonoboy conformance tests passing in Zuul CI
* support for resizing the default node group
* upgrade by moving to a newer template (with updated image)
* add/remove/resize node groups
* customize internal and external network uuids

Initial POC
-----------

An initial POC making this set of assumptions can be found here:
https://review.opendev.org/c/openstack/magnum/+/851076

The POC established a few ground rules for cluster templates:

* image_id in the template the key part of the template
* kube_tag i.e. the k8s version, is in the template and important

For clusters we found:

* using uuid stops any change issues around duplicate names
* default node group users: cluster.flavor_id, cluster.node_count
* Other node groups map in a similar way, using uuid as the name
* control plane size comes from cluster.master_flavor_id

Communicating with K8s
----------------------

The Cluster API driver will make use of helm to template the
Kubernetes resources needed to construct the cluster.

When you create a cluster, helm values will be derived from
a combination of the template and the cluster. 
A cluster will take the image_uuid from the template and
flavor_ids and node counts from the cluster, to create a
set of helm values to be used with the above chart.

Operators can customise a set of "standard" Helm
charts as needed for their specific Cloud.

For an example of what we are using as a starting point,
please see:
https://github.com/stackhpc/capi-helm-charts

OpenStack creds
---------------

Within the Cluster API driver, we want to move towards using
Application Credentials, rather than legacy Keystone trusts.

In part, this is because Cloud Provider OpenStack can't
currently be used with trusts, but application credentials
within a clouds.yaml work perfectly.

We can create per cluster application
credentials that are registered with each cluster on creation,
and can be deleted when the cluster is removed.

We need to ensure these credentials can be rotated when
different users in the same project make changes to a cluster.
Also, when a user is either removed from a project, or has their
roles changed, application credentials will be invalidated.

Assignee(s)
-----------

Primary assignee:

* Matt Pryor (StackHPC)

With support from:

* John Garbutt (StackHPC)

Milestones
----------

* Initial driver with create and delete
* CI passing for create and delete cluster
* Sonoboy conformance tests passing in CI
* Complete work items e.g. node groups

Work Items
----------

Complete all the above milestones.

Dependencies
============

None

Security Impact
===============

A new driver built upon Cluster API has the potential to improve
security for Magnum, due to wider scrutiny of the open source
implementation, a smaller code base for the Magnum team to maintain
and a larger community focussing on the security of Cluster API's
managed clusters.

References
==========

.. [#] https://docs.openstack.org/magnum/latest/user/#rolling-upgrade
.. [#] https://cluster-api.sigs.k8s.io
.. [#] https://github.com/kubernetes-sigs/image-builder/tree/master/images/capi/packer/qemu 
