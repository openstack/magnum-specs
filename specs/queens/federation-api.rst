Magnum Federation API
========================

Launchpad blueprint:

https://blueprints.launchpad.net/magnum/+spec/federation-api

This is a proposal to extend the Magnum API adding support for
federating existing clusters.


Problem Description
-------------------

Magnum provides the means to deploy individual container clusters, supporting
multiple container orchestration engines (COE) - currently Swarm, Kubernetes
and Mesos/DCOS. Each of these clusters is independent from all others, with
its master node(s), worker node(s) and certificate authorities from which client
certificates are issued.

In the case of Kubernetes the COE has the ability to federate independent
clusters, joining them under a single API endpoint and distributing workloads
among all participants. We can envision that other COEs will add similar
functionality in the future.

This proposal adds a new API endpoint to Magnum to setup, persist and manage
federations of local or external clusters.


Use Cases
---------

Below are some of the use cases that this API addresses:

1. A set of Magnum clusters is created in independent projects so that each
   is accounted for its own resources, but a single endpoint is desired where
   policies on workload scheduling can be enforced.

2. A very large container infrastructure is desired, composed of thousands or
   tens of thousands of nodes and growing regularly - as an example, version
   1.7 of Kubernetes supports up to 5000 nodes [3]. A project administrator can
   create a new cluster for each set of new resources and simply add this
   new cluster to an existing federation, easing the management of the overall
   resources and increasing scalability.

3. A set of Magnum clusters is created, each with different characteristics:
   node flavor, storage setup, etc. Federating them together forms a
   heterogeneous cluster.

4. An existing Magnum cluster in an OpenStack environment is to be extended
   using external resources. An external cluster endpoint (deployed in AWS,
   Azure, GKE, another OpenStack or cloud) can be added to an existing
   Magnum federated cluster, including the complex setup and management of
   cluster credentials.

5. A project has several existing clusters which it would like to expose to a
   set of users in a single endpoint, without disrupting existing users of
   each cluster.


Proposed Changes
----------------

The proposed change includes:

* Adding a new '/federation' REST API endpoint to Magnum providing management
  of federated clusters. This includes federation creation, update and
  deletion, as well as adding and removing members from a federation.

* Adding a new entity to the data model (federation) to represent a federated
  cluster and its metadata.

Check sections 'Data Model Impact' and 'REST API Impact' for more details.


Alternatives
------------

A valid alternative would be to manage the setup of the federated control
plane on the client side only. This is possible but has a few drawbacks:

* The logic for the federation setup is in some cases contained in a tool
  supported by the COE. Relying on it would mean adding a client side
  dependency on each of the supported COE clients (hard to manage) or
  replicating the full functionality in the Magnum client code (hard to keep
  up with upstream COE developments).

* Managing the cluster client access context is COE specific and required to
  setup the federation control plane. Asking the user to do this herself
  would greatly reduce the added value of Magnum on managing federations, and
  miss the opportunity to (re)use a lot of the cluster metadata information
  that Magnum already stores.


Data Model Impact
-----------------

A new entity would be added (corresponding tables will be added):

* **federation**

  * uuid
  * name
  * project_id
  * hostcluster_id (the cluster hosting the federation control plane)
  * member_ids (the clusters that are members of the federation)
  * status (the current status of the federation)
  * status_reason (a message with more info regarding the status)
  * properties (additional metadata, coe specific in some cases)

We chose to add a new entity so that we:

* decouple the federation entity from the existing resources
* minimize the impact on the current interfaces
* more easily extend the federation functionality in the future


REST API Impact
---------------

This change leads to a minor version increase in the Magnum API, the
addition of a new REST endpoint and a new set of CLI commands.

Below is a description of the commands to manage federations:

* federation create: creates a new federation, in an existing cluster::

    openstack coe federation create myfederation --hostcluster cluster1

* federation delete: deletes an existing federation::

    openstack coe federation delete myfederation

* federation update: updates the metadata of an existing federation::

    openstack coe federation update --property dns='cluster.local' myfederation

* federation list: list existing federations, with metadata and membership::

    openstack coe federation list

    +------+--------------+---------+-----------------+
    | uuid | name         | members | status          |
    +------+--------------+---------+-----------------+
    | ...  | myfederation | 3       | UPDATE_COMPLETE |
    +------+--------------+---------+-----------------+


* federation show: show details of an existing federation::

    openstack coe federation show myfederation

    +---------------------+-------------------------------------------+
    | Property            | Value                                     |
    +---------------------+-------------------------------------------+
    | uuid                | 5b2ee3b5-2f85-4917-be7c-11a2c82031ad      |
    | name                | myfederation                              |
    | hostcluster         | <uuid-cluster1>                           |
    | members             | ['<uuid-cluster2>', '<uuid-cluster3>']    |
    | project_id          | ae72a4f3-30cf-4406-83b4-40b16bb480d6      |
    | properties          | dns=cluster.local                         |
    | status              | UPDATE_COMPLETE                           |
    | status_reason       | Federation UPDATE completed successfully  |
    +---------------------+-------------------------------------------+

* federation join: adds an existing cluster to a federation::

    openstack coe federation join myfederation cluster2 cluster3

* federation unjoin: parts a cluster from a federation::

    openstack coe federation unjoin myfederation cluster3


Other Implementation Options
----------------------------

See Alternatives.


Security Impact
---------------

Management of federations implies the same level of security as for the
existing cluster management functionality - creation, deletion, update based
on a policy.

Joining/unjoining a federation requires special care and a special policy
evaluating permissions on both source and destination clusters - the host
and the additional members. An example policy is shown below:


Notifications Impact
--------------------

New notifications will be added for:
* federation creation
* federation deletion
* federation update (including metadata and members)


Other End User Impact
---------------------

Users will not be able to delete a cluster that is part of a federation, and
will have to unjoin first.

New subcommands will be added to the openstack client as described above.


Developer Impact
----------------

This functionality will be added via a separate endpoint and metadata stored
in a new entity. Impact in the existing Magnum code will be minimal.


Implementation
--------------

The implementation will be done in 5 phases.

1. Add the new API endpoint and data model entity, and the corresponding
   controller implementation linked to each driver. At this point we will
   have all drivers declaring this functionality as 'Not Implemented'.

2. Implement the federation functionality for the Kubernetes Atomic driver.
   This includes all the described setup in [2] (dns, context, credentials).
   The initial implementation will only support federating Magnum clusters
   in the same OpenStack deployment.

3. Add the new command line tools to the openstack client.

4. Add support for including external Kubernetes clusters (deployed in GKE,
   ACS, AWS, ...) in a Magnum federation. Given this is all set at the level
   of Kubernetes the source environment of the cluster should not be
   relevant, but a way for the user to provide the cluster context to Magnum
   will be added to the command line tools and API.

5. Implement the Magnum federation notifications, for creation, deletion and
   update.

Implementation of this functionality for drivers other than Kubernetes will be
left for future specs, as there is currently no support in other COEs.


Assignee(s)
-----------

See blueprint:

https://blueprints.launchpad.net/magnum/+spec/federation-api


Work Items
----------

See blueprint:

https://blueprints.launchpad.net/magnum/+spec/federation-api


Dependencies
------------

Setting up federations is often done by a COE specific tool - kubefed in the
case of Kubernetes. We will rely on these tools for the driver specific
implementation, thus adding new dependencies on them.


Testing
-------

A new set of tests covering creation, deletion, update of federations and
join/unjoin of clusters needs to be added to both unit and functional tests.


Documentation Impact
--------------------

New documentation will be added to describe the new API endpoint and its
functionality. The quickstart guide needs an update to include information
regarding cluster federation.


References
----------

[1] - Magnum Federation API Blueprint:
https://blueprints.launchpad.net/magnum/+spec/federation-api

[2] - Tutorial on Cluster Federation setup with Kubefed
https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed

[3] - Kubernetes - Building Large Cluster:
https://kubernetes.io/docs/admin/cluster-large/
