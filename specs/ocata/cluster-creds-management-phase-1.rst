..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
   License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Cluster creds management, phase 1
=================================

Launchpad blueprint:

https://blueprints.launchpad.net/magnum/+spec/revoke-cluster-cert

Give cluster admins and owners a way to manage credentials/access to
the cluster.

Problem Description
===================

Currently, magnum does not support cluster credential management. This
is a problem for cluster admins and owners as they are unable to
restrict/deny access to an existing cluster once a user has been
granted access.

Use Cases
=========

This feature is required if, for example, a user leaves an
organization and his access to a cluster needs to be revoked.

Proposed change
===============

This blueprint will be implemented in 2 phases.

In the first phase, we'll implement a method to replace the cluster
certificate. This is mainly to provide at least one method of creds
management to cluster admins and owners as soon as possible.

This operation will invalidate all user credentials. Since there is
only one set of credentials for a cluster, once the credentials are
revoked all users' access will be revoked. All users will need to
create new certificates to gain access to the cluster again. This will
allow admins and owners to revoke a user's keystone credentials and
thus deny them access to the cluster.

If the cluster is non-TLS, an error will be returned to the user.

The CA certificate data stored in barbican will be modified when the
CA certificate is rotated.

In phase 2, we'll implement a finer-grained approach to cert
revocation, which will require magnum to start storing a mapping
between a keystone user and the certs they have generated in
magnum. This will allow admins and owners to list users for each
cluster and revoke certs for a specific user. Before this is possible,
we must contribute to Docker/Kubernetes/Mesos in order to support cert
revocation lists.

Implementation
==============

Work Items
----------

Invoke the make-cert.py script and generate a new certificate using
the existing keystone trust that's on the node, and restart Swarm or
Kubernetes and configure it to use the new certificate.

Assignee(s)
-----------

Primary assignee:
 Jason Dunsmore (jasondunsmore)

Milestones
----------

Target Milestone for completion:
 ocata-3

REST API Impact
===============

The /certificates api will need to be modified to support the new
operation. A REST API will be added for::

 PATCH /certificates/{cluster_id}

Rationale for using PATCH:

TLS certificate management has the concept of "re-issue" of a
compromised certificate. This process is used when the private key has
been compromised. In the use case of an employee leaving the
organization, that's technically what has happened.

Technically speaking a new version of the same certificate with the
same CSR is signed by the CA again with a new key, and issued again
with the intent to replace the previous certificate with an equivalent
one that has a different key, but the same CA and expiration
date. This procedure is also known as "rekeying" a certificate.

Given the existing process, a PATCH action is appropriate. The
mechanism is not intented to be used for things like changing the
Certificate Authority, or extending the expiry of a certificate. Those
actions would be deletions followed by re-issuing a completely
different replacement certificate with a different expiration date.

Security Impact
===============

Admins and owners will have the ability to revoke user access to a
cluster, which is needed for certain security protocols.

If an unwanted user has access to magnum's API, he can simply re-issue
a cert. For a block to a user to be successful, his access to magnum's
API needs to be revoked.

Alternatives
============

1) Manage credentials outside of magnum as a manual process.
2) Wait for Kubernetes support for cluster certificate management.

Documentation Impact
====================

Will update docs on the usage of this new feature.

Dependencies
============

None identified.
