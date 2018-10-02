Magnum Nodegroups
=================

Launchpad blueprint:

https://blueprints.launchpad.net/magnum/+spec/magnum-nodegroups

This is a proposal to extend the Magnum API adding support for nodegroups.

Problem Description
-------------------

Currently Magnum supports the creation of clusters with all the nodes in the
same availability zone. At the same time, the user has the ability to choose
one flavor for master nodes and one for worker nodes.

The concept of nodegroups provides users with the ability to specify groups of
nodes with different properties. Within the scope of a group users are able to
define labels, used image, flavor, etc depending on the purpose these nodes are
going to be used for.

This proposal tries to address the changes needed to support nodegroups with
Magnum.

Use Cases
---------

1. As a user, I want to deploy heterogeneous workloads in the same cluster.
   These can include sql databases with high iops requirements, caches
   requiring large amounts of memory and batch jobs requiring a larger number
   of cpus or even gpus.

2. As a user I want to create higly available clusters with Magnum.

Proposed Changes
----------------

The proposed change includes:

* Add a new '/clusters/{cluster_id}/nodegroups' REST API endpoint to Magnum
  providing management of the given cluster's nodegroups. This includes
  nodegroup creation, update and deletion.

* Add a new object to the data model to represent a nodegroup.

* Change the cluster create procedure to create two default nodegroups, one
  containing the master node(s) of the cluster and one containing the worker
  node(s).

* Adapt the cluster delete procedure to delete also the nodegroups associated
  with the cluster being deleted.

Check sections `Data Model Impact`_ and `REST API Impact`_ for more details.

    NOTE::
    As a first step, users will be able to create nodegroups containing only
    worker nodes. This is because the scripts used for scaling up do not
    support adding new master nodes to the cluster. This change is left as
    future work and will be handled by another spec.

Alternatives
------------

As an alternative to the proposed solution, a user could create multiple
independent clusters and connect them in one single federated control plane,
acting as one heterogeneous cluster.

The problem is that there is no feature parity between the cluster and the
federation APIs and for the time being, cluster federation is supported only by
the Kubernetes COE.

It seems that the concept of nodegroups takes care of the matter at hand, in a
more complete way.

Data Model Impact
-----------------

A new entity would be added (corresponding tables will be added):

* **nodegroup**

  * uuid
  * name
  * cluster_uuid (the uuid of the cluster where the nodegroup belongs)
  * project_id
  * docker_volume_size
  * labels
  * flavor_id
  * image_id
  * node_addresses
  * node_count
  * role (shows if the nodegroup contains master or worker nodes for now)

The project id could be fetched by the cluster, but we add it here also for
future use. This is the scenario where the master nodes belong to an operator
tenant and the cluster nodegroups belong to different projects.

Adding the nodegroup entity means that some information currently stored in the
the cluster, should be moved to nodegroup table. The cluster columns that need
to be dropped are the following:

* node_count
* master_count
* node_addresses
* master_addresses

    NOTE::
    It is really important to point out that moving information from the
    cluster to the nodegroup table will NOT result in changing the output of
    the existing CLIs. The only thing that will change is the way this
    information is stored and subsequently fetched from the database.
    e.g. The cluster show output will contain the node_count information but it
         will be calculated at the API level by summing the node_count of all
         the associated worker nodegroups.

REST API Impact
---------------

This change leads to a minor version increase in the Magnum API, the
addition of a new REST endpoint and a new set of CLI commands.

Below is a description of the commands to manage nodegroups:

* add a new nodegroup, in an existing cluster::

    openstack coe node-group create <params> <cluster> <nodegroup>

* delete an existing nodegroup::

    openstack coe node-group delete <cluster> <nodegroup>

* update an existing nodegroup::

    openstack coe node-group update <params> <cluster> <nodegroup>

* list existing nodegroups given an existing cluster::

    openstack coe node-group list <cluster>

    +------+-------------+-------------+------------+-----------+
    | uuid | name        |  flavor id  | node count |   role    |
    +------+-------------+-------------+------------+-----------+
    | ...  | nodegroup1  |  flavor-1   |      3     |   master  |
    +------+-------------+-------------+------------+-----------+
    | ...  | nodegroup2  |  flavor-2   |      5     |   worker  |
    +------+-------------+-------------+------------+-----------+

* show details of an existing nodegroup::

    openstack coe node-group show <cluster> <nodegroup>

    +---------------------+-------------------------------------------+
    | Property            | Value                                     |
    +---------------------+-------------------------------------------+
    | uuid                | 5b2ee3b5-2f85-4917-be7c-11a2c82031ad      |
    | name                | nodegroup1                                |
    | cluster uuid        | <uuid-cluster1>                           |
    | project id          | <uuid-project1>                           |
    | docker volume size  | 5                                         |
    | labels              | <label1>, <label2>, <label3>              |
    | flavor id           | flavor1                                   |
    | node count          | 3                                         |
    | node addresses      | <ip-node1>, <ip-node2>, <ip-node3>        |
    | role                | master                                    |
    +---------------------+-------------------------------------------+

Backward Compatibility
----------------------

In this section we refer to the clusters created before the introduction of
Magnum Nodegroups as "old clusters".

During the upgrade, the existing stacks will not be modified. This is the
reason that adding as well as deleting nodegroups to/from old clusters will be
not permitted.

Showing details for a nodegroup in an old cluster should work correctly.

Security Impact
---------------

There is no keypair added in the nodegroup object as all nodegroups will
inherit the one set to the cluster. This approach was chosen, in order to not
propagate the use of keypairs to the level of nodegroups and complicate further
their removal in the future.

Notifications Impact
--------------------

New notifications will be added for:
* nodegroup creation
* nodegroup deletion
* nodegroup update

Other End User Impact
---------------------

New subcommands will be added to the openstack client as described above.

At the same time, some of the existing commands for managing clusters have to
be adapted:

### Cluster Create ###
The existing create cluster cli will result in a cluster with two default
nodegroups, one for the master node(s) and one for the worker(s).

### Cluster Delete ###
When the user deletes a cluster, all the associated nodegroups will be deleted
as well. There is no point of making the user delete all the nodegroups
separately before deleting the cluster.

### Cluster Update ###
Cluster update should continue working for the already existing clusters and it
should be deprecated for the new ones. All scaling operations for new clusters
should be done using the "node-group update" command.

### Cluster Show ###
Firstly, the node count of the cluster should reflect the sum of the node count
fields of all its nodegroups.
Another thing that has to be handled is showing the status of the cluster. The
show cluster cli should summarize the status of its nodegroups since each stack
has its own status.

Developer Impact
----------------

None.

Implementation
--------------

The implementation will be done in 4 phases.

1. Add the new API endpoint and data model entity, and the corresponding
   controller implementation linked to each driver. At this point we will
   have all drivers declaring every operation regarding nodegroups as
   'Not Implemented'. At the same step, we need to adapt all the operations
   for cluster management.

2. Implement the nodegroup functionality for all drivers.

3. Add the new command line tools to the openstack client.

4. Implement the Magnum nodegroup notifications, for creation, deletion and
   update.

Assignee(s)
-----------

Primary assignee:
  <ttsiouts>

Work Items
----------

See `Implementation`_.

Testing
-------

A new set of unit and functional tests covering creation, deletion and update
of nodegroups is needed. At the same time, the existing tests for cluster
creation, deletion and update should be adapted.

Documentation Impact
--------------------

New documentation will be added to describe the new API endpoint and its
functionality as well as the changes in the existing cluster API.

References
----------

Magnum Nodegroups Blueprint:
https://blueprints.launchpad.net/magnum/+spec/magnum-nodegroups
