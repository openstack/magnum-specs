Magnum Control Plane resize
===========================

This is a proposal to implement support for resizing the number
of control plane nodes.


Problem Description
-------------------

Currently, no driver in Magnum supports resizing the number of control plane nodes.

Cluster API supports modifying the control plane node count to any odd number such
that etcd's raft consensus can be established and isn't wasteful of nodes.

In practice, this means 1, 3, 5 and 7 nodes are supported, though any odd number
may be acceptable[1].


Use Cases
---------

Below are some of the use cases that this change addresses:

1. User creates a cluster with a single Control Plane node, and then
   updates their requirement to have an HA control plane.
2. User has a HA cluster and wants to save resources by reducing to 1.
3. Moving between 3 and 5 control plane nodes may be a good choice for users
   as their availability needs change.

All of these actions currently require re-creation of the cluster.


Proposed Change
---------------

The proposed change includes:

1. Move the logic of `default-master` resize away from the Conductor and into the
   driver.
2. The driver may choose to validate the new size, and only support odd numbered
   sizes such as 1, 3, 5 and 7.
3. No existing drivers in Magnum will support this resize action. The first drivers
   to use this will be out of tree Cluster API drivers.

Check sections 'Data Model Impact' and 'REST API Impact' for more details.


Alternatives
------------

The alternative is to continue requiring users to re-create clusters when
they have a need for differing control plane sizes.


Data Model Impact
-----------------

There is no data model impact.

The database already stores the number of control plane nodes.


REST API Impact
---------------

The REST API does not need to change.

This feature will be supported and implemented using the existing resize API
under `POST /v1/clusters/{cluster_ident}/actions/resize` by providing a
`nodegroup` parameter of `default-master` and the requested size.

This resize API currently does not permit any changes to the `default-master`
nodegroup.

In the CLI environment, this endpoint is accessible with the following command::

    openstack coe cluster resize --nodegroup default-master <cluster_ident> <node_count>

Security Impact
---------------

None


Notifications Impact
--------------------

None


Developer Impact
----------------

Driver developers will have the option of supporting this resize action.

The default will be backwards compatible in that it is not supported.


Implementation
--------------

Assignee(s)
-----------

dalees


Milestones
----------
Target Milestone for completion:
  Caracal


Work Items
----------

* Move resize validation logic from Conductor and move into driver.
* Define a default implementation for all drivers. The default will be Not Supported, which
  is consistent with the current behaviour.
* Implement the resize action in one or more drivers. The assignee intends to implement this in
  the CAPI Helm driver, and inform other out-of-tree developers of this optional feature.

Dependencies
------------

None

References
----------

[1] https://etcd.io/docs/v3.5/faq/#what-is-failure-tolerance
