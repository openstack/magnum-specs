..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


===================
Credential rotation
===================

Problem description
===================

Cluster administrators may wish to rotate the application credentials used to
create their clusters. This may be to regularly rotate them as a security
measure, to migrate their cluster ownership to a different user account, or to
recreate credentials that have become invalidated by removal of roles from a
user.

In this spec, we propose a new Magnum API endpoint that actions the rotation
of the application credential bound to the user who made the request.

Proposed change
===============

The proposed change introduces the following procedure to rotate credentials,
the nature of which is dictated by the underlying driver.

1. Cluster operator invokes credential rotation via the Magnum client:

    $ openstack coe credential rotate

2. Magnum client initiates PATCH /v1/credential/{cluster_ident}.
3. Magnum API performs validation and invokes conductor via RPC.
   This returns HTTP 202 Accepted on successful validation, or HTTP 400 Bad
   Request on failed validation.
4. Conductor invokes underlying driver (e.g. magnum-capi-helm) to perform
   credential rotation.
5. Driver issues a new application credential bound to the user who made the
   request.
6. Driver deploys application credential to cluster as a `Secret`.

The driver will manage the reporting of cluster state throughout this process.

Alternatives
--------------

The alternative is to continue requiring an administrator to manually change
the credentials in a cluster to change ownership.

Implementation
==============

The implementation will involve development in the Magnum API, conductor,
magnum-capi-helm driver, client, and Horizon. Changes to the data model are
dependent on how effectively the magnum-capi-helm driver can leverage the
state management capabilities of Keystone and Kubernetes itself. We can, for
example, check the existing secret for attributes of the application credential
we expect to find in Keystone.

REST API
-----------

A new credential API will be introduced with one endpoint. This will require a
new API microversion.

The `/v1/credential/{cluster_ident}` endpoint will initiate the rotation of a
cluster's credential. The exact definition of "credential" will vary between
drivers and may not be implemented in all cases.

Assuming the requester is authorized, has the correct privileges, and provides
a valid request; the API will return HTTP 202 Accepted. To accommodate this,
the `credential:rotate` policy will be introduced.

Additionally, if the underlying driver does not implement the credential
rotation functionality, the request will fail and detail that the operation is
not supported.

Conductor
-----------

The only change to the conductor will be to invoke the underlying driver to
perform the credential update. All cluster state operations and the rotation
of credentials is delegated to this driver.

Driver
-----------

The magnum-capi-helm driver is used here as an example implementation and will
be the initial driver to support this feature. It will follow this process to
rotate application credentials in a cluster:

1. Change the cluster status to `UPDATE_IN_PROGRESS`.
2. Check cluster `Secret` for the name of the application credential currently
   in-use.
3. Request a new application credential from the Keystone client, using a name
   template.
4. Update the existing `Secret` with the new application credential.
5. Delete the old application credential.
6. Change the cluster status to `UPDATE_COMPLETE` on success, or
   `UPDATE_FAILED` on failure.

(3) would benefit from a more sophisticated name template for application
credentials. The current template is `magnum-{cluster.uuid}` which could be
changed to simply iterate (i.e. `magnum-{cluster.uuid}-{n+1}`) or have a random
nonce value (i.e. `magnum-{cluster.uuid}-{nonce}`). Either would work, and
could be made to support migration from the existing name template.

(4) only updates the `cloud-config` secret containing the application
credential on the management cluster. It does not update corresponding workload
cluster secrets, nor does it restart containers dependent on them as these
should be handled by other systems. Some workloads, such as the Cinder CSI and
other OCCM integrations, may see authentication failures due to the subsequent
deletion of the application credential in (5) until they pick up the updated
`cloud-config` secret. For example, when using the capi-helm-charts stack
including the cluster-api-addon-provider, the management cluster's secret is
watched for changes [1] and updates workload cluster secret [2]. Changes to
these secrets may require dependent workloads to be restarted but this will
be developed separately to this proposal which is focused on the Magnum
changes.

Client
-----------

The Magnum client will be updated to introduce a new command to action the
credential rotation via the API endpoint. This will not require any
additional arguments beyond the ID/name of the cluster as above.

   $ openstack coe credential rotate my-cluster

   # In future
   $ openstack coe credential show my-cluster
   $ openstack coe credential create --secret my-secret my-cluster
   $ openstack coe credential set my-credential my-cluster

Horizon
-----------

Horizon will be updated to include 'Rotate Application Credential' as an
action within the drop-down menu of each cluster.

Assignee(s)
-----------

Primary assignee:

* northcottmt

With support from:

* dalees

Milestones
----------

* Add credential API and associated policies to Magnum.
* Invoke driver from Magnum conductor.
* Update status, and rotate credentials in magnum-capi-helm driver.
* Add command to support credential rotation in Magnum client.
* Add drop-down menu item to rotate credentials per cluster in Horizon.

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

An application credential rotation action can improve security for Magnum and
managed clusters when utilised properly by cluster administrators. The
introduced changes include an easier method to rotate application credentials,
promoting more regular credential rotation. When invoked regularly, secrets
containing the application credential are less likely to become stale or
compromised.

References
==========

[1] https://github.com/azimuth-cloud/cluster-api-addon-provider/pull/49
[2] https://review.opendev.org/c/openstack/magnum-specs/+/955448/comment/f175de4e_30d3a8a6/
