..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Cluster Driver Encapsulation and Lifecycle Hooks
================================================

Launchpad blueprint:

https://blueprints.launchpad.net/magnum/+spec/bp-driver-consolodation

By encapsulating all operations and monitoring performed on a cluster into the
driver interface, operators have a broader choice in which services they chose
to install and maintain in order to most efficiently operate Magnum according
to their own constraints and service expertise.

Also, extending this driver interface to include the definition of pre and post
operation hooks can allow operators to more easily integrate Magnum with other
parts of their infrastructure which could possibly exist outside of the
OpenStack cloud.

Problem Description
===================

Magnum currently only supports drivers that leverage Heat as the underlying
cluster orchestration engine. Because Magnum makes direct calls to Heat during
creation and deletion of a cluster, operators and other contributors who wish
to leverage alternative orchestration engines are unable to do so.

Additionally, should the operator need to customise this orchestration, they
are limited to only those mechanisms directly supported by Heat.

Use Cases
=========

1. Leverage one or more alternative OpenStack services for cluster
   orchestration such as Senlin or Mistral.

2. Allow the operator to define complex lifecycle management hooks outside of
   those provided by Heat directly in their driver implementation.

3. Drivers that leverage Heat should be even simpler to implement and only
   require the implementor to define a Heat template with little or no driver
   code.

Proposed Change
===============

1. The driver class hierarchy shall be extended to include a base driver class
   for defining the (mostly existing) contract for a driver independent of
   orchestration engine while adding methods to cover those operations for
   which Magnum currently calls Heat directly.

2. Magnum code shall be refactored to defer any cluster operations to this new
   driver interface.

3. An abstract implementation of this interface based on common Heat operations
   will be created and the current driver code refactored to be based on the
   new implementation.

Proposed Driver Interface
-------------------------
Attributes and methods of the existing classes keep their current definitions
unless specifically called out below. The ``osc`` argument has been removed
from the applicable driver methods to allow for non-OpenStack drivers to manage
their own clients.

.. code-block:: python

    @six.add_metaclass(abc.ABCMeta)
    class Driver(object):
        # magnum.drivers.common.driver.Driver

        @abc.abstractproperty
        def provides(self):
            # return a list of (server_type, os, coe) tuples supported by this
            # driver

        @abc.abstractmethod
        def create_cluster(self, context, cluster, cluster_create_timeout):
            # ... renamed create_stack method

        @abc.abstractmethod
        def update_cluster(self, context, cluster, scale_manager=None,
                           rollback=False):
            # ... renamed update_stack method; For now, this will do what it
            # does today. The intent is to avoid too many disruptive changes in
            # the current interface and defer extensions and/or new features
            # to subsequent specifications. In the future, this method should
            # probably be broken down into more granular operations like scale,
            # upgrade, rename, etc.

        @abc.abstractmethod
        def delete_cluster(self, context, cluster):
            # ... add this method to the driver

Proposed Heat Driver
--------------------
These classes essentially consolidate existing Heat-based driver logic into
easily extensible classes that most existing drivers can implement.

.. code-block:: python

    @six.add_metaclass(abc.ABCMeta)
    class HeatDriver(magnum.drivers.common.driver.Driver):
        # magnum.drivers.common.driver.heat.HeatDriver

        @abc.abstractmethod
        def get_template_definition(self):
            # return an implementation of
            # magnum.drivers.common.driver.heat.TemplateDefinition

        def create_cluster(self, context, osc, cluster, cluster_create_timeout):
            # per the current driver implementation

        def update_cluster(self, context, osc, cluster, scale_manager=None,
                           rollback=False):
            # per the current driver implementation

        def delete_cluster(self, context, cluster):
            # ... move the existing delete logic into this driver

    @six.add_metaclass(abc.ABCMeta)
    class TemplateDefinition(object):
        # move magnum.drivers.common.template_def.TemplateDefinition to
        # magnum.drivers.common.driver.heat.TemplateDefinition


    class HeatPoller(object):
        # move existing
        # magnum.conductor.handlers.cluster_conductor.HeatPoller to
        # magnum.drivers.common.driver.heat.HeatPoller

Design Principles
=================

Magnum should never have to make assumptions about the underlying orchestration
system and should defer all calls for information to the specified driver for a
cluster. To do this the driver interface should be designed to sufficiently
allow Magnum to enact all aspects of a cluster's lifecycle.

A sub-class of driver should be created such that a user or operator wishing to
continue to use Heat as the underlying orchestration engine would only need to
extend this class to specify a template and at best some basic mappings between
driver parameters and parameters in the underlying template.

Alternatives
============

1. Force users that wish to use other OpenStack services for orchestration to
   do so via Heat resources for those services.

       a. This forces operators into deploying, supporting, and maintaining
          Heat in addition to whichever other services they may actually
          require.

       b. This assumes 100% coverage by Heat of other applicable services and/or
          limits the aspects of the target service that a provider can take
          advantage of to those exposed by the Heat resources.

       c. This requires users who wish to integrate custom or existing services
          and infrastructure outside of OpenStack to implement custom Heat
          resources in addition to having to implement a custom Magnum driver.

2. Force users wishing to apply something other than Heat alone for cluster
   orchestration to fork or manage a patched version of Magnum. This would mean
   patching or overriding the conductor and driver mechanisms.

       a. Patching leads to anger. Anger leads to hate. Hate leads to
          suffering.

       b. Leaving operators the only reasonable option to fork the project
          should be avoided if at all possible.

Data Model Impact
=================

Refactoring the data model to rename or move the ``stack_id`` column from the
cluster data model could be considered, however it is probably ok to leave this
for now and just make it nullable to remove the need for data migration and
upgrade downtime.

REST API Impact
===============

This proposal does not anticipate requiring any changes to the current v1 api
nor to the proposed v2 API allowing for heterogeneous clusters.

Security Impact
===============

None identified.

Notifications Impact
====================

None identified.

Other End-user Impact
=====================

None identified.

Performance Impact
==================

No change in current performance is anticipated as this is largely a refactor
of existing code.

Other Deployer Impact
=====================

By default, the drivers and dependencies won't change as this proposal would
only refactor code and keep the existing drivers functionally intact.

Deployers wishing to customize cluster orchestraion will have significantly
increased flexibility in how to implement and a broader choice in what they can
implement without having to patch Magnum directly.

Developer Impact
================

Developers wishing to create customised drivers based on Heat should have a
much easier time given that in many cases, all they need to do is subclass the
base Heat driver and define a template location.

Developers who wish to leverage a service other than Heat will be able to
develop those drivers without having to touch or patch other parts of Magnum.

Implementation
==============

Assignee(s)
-----------

TBD

Work Items
----------

1. Refactor current driver code into an abstract Driver class that defines a
   driver's contract

2. Create a base HeatDriver class that implements most common methods for
   leveraging Heat for cluster operations independent of a specific template

3. Identify and refactor references to the template_def object in favor of new
   driver methods

4. Refactor the current drivers to leverage the generic Heat driver

5. Refactor Magnum code to defer all orchestration operations to the driver


Dependencies
============

None identified.

Testing
=======

1. Add unit tests to validate each new class in the driver hierarchy

2. All other functional unit tests should pass without modification.

Documentation Impact
====================

1. The cluster drivers section of the users guide will need to be updated to
   include both the new interface as well as implementing a driver via the Heat
   interface.
   (http://docs.openstack.org/developer/magnum/userguide.html#cluster-drivers)

2. The documentation will need to make it clear that existing diagnostic
   techniques apply to Heat-based drivers, and that other driver types will
   have a driver-specific troubleshooting process.

References
==========

None identified.
