..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


================================================
Explicit Driver Selection in Cluster Templates
================================================

https://blueprints.launchpad.net/magnum/+spec/cluster-template-driver-selection

The Caracal `improve-driver-discovery` spec introduced the ``magnum_driver``
Glance image property, allowing deployments with multiple drivers that cover
the same (``coe``, ``os``, ``server_type``) tuple to disambiguate via image
metadata. This spec extends that work by giving operators and users two
additional, higher-level controls: a ``driver`` field on the cluster template
API and a deployment-wide ``default_driver`` configuration option.


Problem description
===================

After the Caracal work, the driver used by a cluster template is still
determined implicitly — either from the ``magnum_driver`` image property or,
if that property is absent, by whichever driver happens to match the tuple
first. This creates two pain points:

1. **No user-side control.** A user who wants to target a specific driver
   (e.g. to test a new out-of-tree driver alongside the existing one) cannot
   express that preference in the API. They must rely on the image carrying
   the right ``magnum_driver`` property, which may not be under their control.

2. **No operator-side default.** An operator who wants to ensure a consistent
   driver is used across templates — regardless of what image metadata is
   present — has no clean mechanism to express that. The only workaround is
   to disable all other drivers via ``[drivers]/disabled_drivers``, which is
   cumbersome and prevents running more than one driver simultaneously.


Proposed change
===============

Two complementary changes are proposed:

**1. ``driver`` field on the Cluster Template API**

A ``driver`` field is added to the Cluster Template create and show APIs.
When a user supplies ``driver`` at creation time, that value is stored and
used as-is, bypassing any image metadata lookup for driver resolution.

The field is optional. Existing behaviour is preserved when it is omitted.

**2. ``[drivers]/default_driver`` configuration option**

A new string option ``default_driver`` is added to the ``[drivers]`` section
of ``magnum.conf``. When set, it is used as the driver for any cluster
template whose driver is not provided by the user or the image metadata.

**Driver resolution priority order**

At cluster template creation time the driver is resolved using the following
priority, stopping at the first value that is non-empty:

1. ``driver`` explicitly provided by the user in the API request.
2. ``magnum_driver`` property of the Glance image referenced by the cluster
   template.
3. ``[drivers] default_driver`` configuration option in ``magnum.conf``.
4. The name of the first alphabetically sorted driver among all drivers
   registered via the ``magnum.drivers`` entry-point group.

This ordering ensures backwards compatibility: deployments that do not set
``default_driver`` and do not tag images with ``magnum_driver`` continue to
work exactly as before (fallback 4 preserves the previous implicit behaviour).

**API microversion**

Because the ``driver`` field is new input/output on an existing resource, a
new API microversion is required. Clients that negotiate the new microversion
may send and receive the ``driver`` field; older clients see the field silently
absent.

REST API impact
---------------

*POST /v1/clustertemplates*

Request body change (new microversion)::

    {
        "driver": "k8s_capi_helm_v1",   # optional
        ...
    }

Response body change::

    {
        "driver": "k8s_capi_helm_v1",
        ...
    }

*GET /v1/clustertemplates/{cluster_template_ident}*

Response body change::

    {
        "driver": "k8s_capi_helm_v1",
        ...
    }

Configuration impact
--------------------

New option in ``magnum.conf``::

    [drivers]
    default_driver = <driver-entry-point-name>

If unset, Magnum falls back to the alphabetically first available driver
(existing behaviour).

Alternatives
------------

**Alt 1 – Operator disables conflicting drivers**

The existing ``[drivers]/disabled_drivers`` list can be used to ensure only
one driver matches a given tuple, effectively forcing a default. This is
cumbersome for operators who want more than one driver in production and does
not give end users any way to select a driver.

**Alt 2 – Image-only approach (status quo)**

Requiring all images to carry a ``magnum_driver`` property is workable but
shifts the burden entirely onto image management. An operator who imports a
public image without the property has no recourse short of modifying the image
metadata. A config default is a lower-friction alternative.

**Alt 3 – Driver as a mandatory field**

Making ``driver`` a required field at cluster template creation would be the
most explicit approach, but would break backwards compatibility for all
existing clients and automation tooling.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mnasiadka

Milestones
----------

Target milestone for completion:
  Hibiscus(2026.2)

Work Items
----------

- Add ``driver`` field to the Cluster Template object and DB model.
- Add ``driver`` field to the Cluster Template API controller (create/show).
- Implement driver resolution priority chain in the controller.
- Add ``Driver.get_default_driver()`` class method.
- Add ``[drivers]/default_driver`` config option.
- Update ``enforce_driver_supported`` validation to use the resolution chain.
- Add a new API microversion.
- Update ``python-magnumclient`` to expose the ``driver`` field.
- Add unit tests for each priority-chain scenario.
- Update API reference documentation.

Dependencies
============

- `improve-driver-discovery spec (Caracal)
  <https://opendev.org/openstack/magnum-specs/src/branch/master/specs/caracal/improve-driver-discovery.rst>`_
  — this spec is a direct extension of that work. The ``magnum_driver`` image
  property introduced there is priority level 2 in the resolution chain above.

Security Impact
===============

The ``driver`` field allows a user to name any driver that is registered on
the deployment. Validation already rejects unknown driver names via the
existing ``get_driver`` path, so no additional authorization surface is
introduced beyond what is already present.

Operators who wish to restrict which drivers users may choose can continue to
use ``[drivers]/disabled_drivers`` to remove unwanted drivers from the
registered set.
