===========================
Improve Driver Discovery
===========================

https://blueprints.launchpad.net/magnum/+spec/improve-driver-discovery

Magnum supports Kubernetes, which is a fast evolving project. Due to the fast
evolution, there have been multiple drivers proposed to use different methods
to install Kubernetes, e.g. with ClusterAPI. There is a need to prevent drivers
from clashing when multiple drivers are installed in a deployment.


Problem description
===================

When creating a cluster, Magnum discovers the suitable driver based on a
combination of ('coe', 'os', 'server_type') properties. The properties have the
following meaning:

- ``coe`` - Container Orchestration Engine of the cluster. E.g. kubernetes, swarm
- ``server_type`` - Type of servers that make up thie cluster. E.g. vm for Virtual Machine
- ``os`` - Operating System of the cluster. E.g. fedora-coreos for a Fedora CoreOS image

Very briefly, this is how driver selection works

1. On startup of Magnum, each driver registers itself with information
   specifying what it supports. For example, the Kubernetes Fedora CoreOS
   driver has the following values:::

    {'server_type': 'vm',
     'os': 'fedora-coreos',
     'coe': 'kubernetes'}

2. Cluster Templates are created with similar required properties to specify
   what kind of cluster it is a template for.

3. A new cluster, upon creation, will have its Cluster Template's properties
   matched with the available drivers' properties.
   If a match is found, the driver will be used to provision the
   cluster.

However, a current limitation is that each set of ('coe', 'os', 'server_type')
values can be only used by 1 driver. This can be a problem if there are
multiple drivers that provide the same type of cluster, but differing in
implementation detail.

For example, it may be possible that, for the following properties,::

    {'server_type': 'vm',
     'os': 'ubuntu',
     'coe': 'kubernetes'}

there are two drivers, one using Heat and one using ClusterAPI, to create a
Kubernetes cluster with Ubuntu VMs.

Currently, having both drivers working in a deployment is not possible.

Although such conflicts can be handled by code review for in-tree drivers,
there are now multiple drivers developed out-of-tree.

Magnum's intention is to allow multiple drivers to flourish, so it can be
nimble and handle the fast evolving Kubernetes ecosystem. Therefor, this issue
needs to be resolved urgently.


Proposed change
===============

It will be ideal to have a Cluster Template specify explicitly what driver it
wants to use, instead of having a driver be matched by the discovery mechanism.

One way to achieve that is to use a Glance image property 'magnum_driver' to
specify the driver name. The additional property allows Magnum to select the
preferred driver if there are multiple matching drivers. In the example of two
drivers both matching::

    {'server_type': 'vm',
     'os': 'ubuntu',
     'coe': 'kubernetes'}

the Glance image(s) for each of the drivers will have the following properties.::

  Driver 1:
    {'os_distro': 'ubuntu',
     'magnum_driver': 'k8s_ubuntu_v1'}

  Driver 2:
    {'os_distro': 'ubuntu',
     'magnum_driver': 'k8s_capi_helm_v1'}


For existing drivers that do not conflict, the current way of discovery remains
unchanged.


Alternative 1
-------------

A simple alternative is that each deployment explicitly only enable one driver
matching the tuple. This can be currently done using the configuration option
``[drivers]/disabled_drivers``.

However, there are a few drawbacks.

- There is no ability to run with two drivers with identical properties, so
  that a deployment can move from one to the next.
- Each new driver has to be explicitly disabled.

Alternative 2
-------------

Another alternative will be for every new driver to stick to a naming
convention to avoid clashing. This means using the driver entry point name
(which is unique) for the 'os' property. E.g.::

    {'server_type': 'vm',
     'os': 'k8s_fedora_coreos_v1',
     'coe': 'kubernetes'}

This means Glance images for the cluster will have the 'os_distro' property set
to the magnum driver name.

This can be further extended if multiple operating systems for a driver is
needed. E.g.::

    [{'server_type': 'vm',
      'os': 'k8s_fedora_coreos_v1:ubuntu',
      'coe': 'kubernetes'},
     {'server_type': 'vm',
      'os': 'k8s_fedora_coreos_v1:flatcar',
      'coe': 'kubernetes'}]

The drawback is that this may be a misuse of the 'os_distro' property as
intended by Glance [#f1]_ . However, Magnum has not stuck to this strictly, as it
has been using 'fedora-coreos' and 'fedora-atomic' to distinguish the different
Fedora OS, when the os_distro property for all of them should be 'fedora' [#f2]_.


Additional change
=================

Currently the way of using Glance 'os_distro' property may have unintentional
side effect [#f3]_, as some of the values used in os_distro may be invalid in
libosinfo. For example, the current heat driver uses 'fedora-coreos'. The valid
value in libosinfo is 'fedora' [#f2]_.

We should take this opportunity to deprecate the use of 'os_distro', and
instead replace it with a 'magnum_distro' property.

With this change, the proposed properties for Glance image(s) for the driver will have::

    {'magnum_distro': 'ubuntu',
     'magnum_driver': 'k8s_capi_helm_v1'}

Fallback using os_distro should be maintained.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  waipengyip

Milestones
----------

Target Milestone for completion:
  Caracal

Work Items
----------

- Implement magnum_driver support

- Implement magnum_distro support with os_distro fallback

Dependencies
============

None

Security Impact
===============

None


**Footnotes**

.. [#f1] https://docs.openstack.org/glance/latest/admin/useful-image-properties.html
.. [#f2] https://gitlab.com/libosinfo/osinfo-db/-/blob/v20231027/data/os/fedoraproject.org/coreos-stable.xml.in?ref_type=tags#L10
.. [#f3] https://specs.openstack.org/openstack/nova-specs/specs/mitaka/implemented/libvirt-hardware-policy-from-libosinfo.html
