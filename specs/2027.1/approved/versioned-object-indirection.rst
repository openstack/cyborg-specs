..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Versioned Object Indirection Support
====================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/versioned-object-indirection

We will introduce oslo.versionedobjects indirection so that Cyborg agents
can invoke object operations through the conductor without direct database
access. PCI VFIO variant-driver support and the generic NVMe driver need
this interface for configuration validation, startup recovery and persisted
state updates.
Separating agents from the database model also moves Cyborg closer to
rolling-upgrade support. This change does not itself enable rolling upgrades.

Problem Description
===================

Cyborg agents do not access the database directly. They send discovered
resources to the conductor through ``report_data()``. Other agent operations
receive the data they need as RPC arguments. Agents have no general way to
query or update persisted Cyborg objects.

The PCI VFIO variant-driver feature needs to compare its configured managed
mode with persisted attach-handle metadata and allocation state during
startup. The NVMe driver needs persisted device and allocation state for
startup recovery, and must record whether asynchronous cleanup succeeded or
failed. Sending another discovery report cannot express these checks and
lifecycle operations. Both features need object operations through the
conductor while keeping database access off the compute host.

Cyborg already uses oslo.versionedobjects to represent and serialize objects
across RPC, but does not configure its object indirection interface. Adding
a dedicated conductor RPC method for each new object operation would
duplicate the dispatch and object compatibility mechanisms provided by
oslo.versionedobjects.

Services with direct database access, including the API and conductor, must
use database models compatible with the deployed schema. Indirection keeps
agents independent of that schema while allowing them to request object
operations. It lets agents use older in-memory objects and wire formats
while the conductor uses the current database model. The
oslo.versionedobjects compatibility routines handle these differences.

Cyborg also hard-codes its RPC client version caps. Operators need to pin
these caps to the versions supported by the target services during upgrades.

Use Cases
---------

* Developers can expose selected object operations to agents without adding
  a separate RPC endpoint for each operation.
* Operators need PCI startup validation to reject a managed-mode change
  for a device already allocated to a guest, using persisted state rather
  than discovery data alone.
* Operators need NVMe startup checks to identify interrupted or missed
  cleanup, and cleanup completion to persist ``available`` on success or
  ``error`` on failure without giving agents database access.
* Operators want to upgrade the API, conductor and database first, then
  upgrade compute-node agents in batches. Indirection is one prerequisite
  for supporting this upgrade order.
* Operators can pin outgoing conductor and agent RPC versions to those
  supported by the services in their deployment.

Proposed Change
===============

This change will let Cyborg agents call selected object methods through the
conductor while keeping database access on the conductor. We will configure
object indirection during agent startup and add the conductor RPC operations
that it needs. We will also add manual RPC version caps so operators can
limit calls to the versions supported by their services.

To explain how this works, it helps to distinguish sending an object from
calling one of its methods. Cyborg already uses oslo.versionedobjects to
serialize objects over RPC. Serialization lets one service send object data
to another, but does not itself request method execution. The indirection
API lets the caller ask the receiving service to execute an object method
and return its result. In this proposal, the agent sends the request and
the conductor executes the method.

An object author marks a method with an oslo.versionedobjects decorator.
When code calls that method, the decorator checks whether an
``indirection_api`` is configured. If it is unset, the method body runs
locally. Otherwise, the decorator delegates the call to the configured
implementation. The library defines this interface, but the project decides
how to send the call and which service will execute it.

The caller and that service may understand different object versions. For
class-method calls, the decorator supplies a version manifest, a mapping of
object names to the versions the caller supports, including related objects.
The service can then return compatible object data using the object's
backleveling rules. This allows the call to return objects the caller can
use without requiring both services to have identical object definitions.

Applying Indirection to Cyborg
------------------------------

For Cyborg, the conductor is the service that will execute these methods.
We will set ``CyborgObject.indirection_api`` to a conductor RPC client during
agent startup, before object operations can run. The conductor will leave
indirection unset so that it executes delegated methods locally rather than
sending another RPC request. The API service will also keep its existing
local object execution behaviour.

Object authors will use ``@base.remotable``, exposed through
``cyborg.objects.base``, to mark methods that can use this interface. We
will raise the minimum oslo.versionedobjects version to 3.10.0 so that the
same decorator supports both class and instance methods.

The PCI VFIO variant-driver managed-mode feature needs this access before
the agent sends a normal discovery report. At startup, the agent will
compare the configured and discovered ``managed`` value with persisted
attach-handle metadata. If the value would change for an allocated handle,
startup will fail rather than accept configuration that changes how an
assigned device is managed. For an unallocated handle, the agent will
request a conductor-executed metadata update that preserves the handle's
UUID and checks that the persisted state has not changed since the agent
read it.

The generic NVMe driver needs similar reads for configuration validation
and startup recovery. It needs to identify devices left in cleanup states
and allocations whose cleanup request was missed. It also needs remote
updates after cleanup. The conductor's request starts asynchronous work on
the compute host, and cleanup can continue after the RPC dispatch returns.
Once the agent confirms cleanup succeeded, it will invoke an object
operation through the conductor to record ``available``. Failure or timeout
will instead record ``error`` so that an unclean device cannot be reused.
The agent performs the hardware cleanup and requests the state transition.
The conductor executes the database operation.

These examples require more than writing a field value. The dependent
features will define their object methods, validation rules, concurrency
checks and Placement reservation coordination. This spec supplies the
indirection they need, not those driver-specific operations.

Both examples start with looking up persisted objects for the agent's host.
For collection results, we will follow Nova's pattern of returning dedicated
versioned list objects rather than plain Python lists. For the PCI feature,
a ``DeviceList`` could provide that lookup. A simplified example is:

.. code-block:: python

    from cyborg.objects import base, fields
    from cyborg.objects.device import Device


    @base.CyborgObjectRegistry.register
    class DeviceList(base.ObjectListBase, base.CyborgObject):
        VERSION = '1.0'
        fields = {'objects': fields.ListOfObjectsField('Device')}

        @classmethod
        @base.remotable
        def get_by_hostname(cls, context, hostname):
            devices = Device.list(context, {'hostname': hostname})
            result = cls(context=context, objects=devices)
            result.obj_reset_changes()
            return result

Python applies ``@base.remotable`` first, wrapping the method with the
local-or-remote behaviour described above. The outer ``@classmethod``
supplies the class as ``cls`` when the method is called. Since there is no
instance to hold a request context, the caller passes it explicitly. The
agent does not need to construct an RPC message itself. It calls:

.. code-block:: python

    devices = DeviceList.get_by_hostname(context, hostname)

On the agent, the decorator sends the object name, method name, version
manifest and arguments through ``object_class_action_versions``. The
conductor resolves the registered ``DeviceList`` class and invokes the
method. Since conductor indirection is unset, its decorator runs the body
locally. ``Device.list()`` reads the database there and supplies the members
of the versioned list. The conductor serializes the ``DeviceList`` using
the caller's target version and version manifest. OVO's compatibility
routines handle its contained ``Device`` objects before the response is
sent, without an additional RPC round trip for each member. The caller
receives a ``DeviceList`` that it can iterate over. The same call made
locally in the conductor does not need an RPC round trip.

An instance method uses the same ``@base.remotable`` decorator without
``@classmethod``. In that case, the decorator takes the context stored on
the object and delegates through ``object_action``. The remote method may
change the object as well as return a result. The decorator therefore
applies returned field updates and change-tracking state to the caller's
instance before returning the result.

In either case, the decorator is what makes the method use indirection.
An undecorated method still runs locally, so an agent must not call an
undecorated method that accesses the database. The device lookup above is
an example of how a dependent feature can expose an operation. Adding the
production list classes and their methods belongs to those features, not
this infrastructure spec.

To support these calls, we will add three synchronous operations to the
conductor RPC interface at version 1.1, using the existing transport and
Cyborg object serializer:

* ``object_class_action_versions`` invokes an object class method using the
  caller's object version manifest and returns a compatible result.
* ``object_action`` invokes an instance method and returns its result,
  changed field values and change-tracking state. The caller uses these
  values to update its instance as oslo.versionedobjects requires.
* ``object_backport_versions`` converts an object to the versions supported
  by the caller, including its nested objects, using a version manifest.

Once these endpoints exist, adding another remotable object method will
normally reuse RPC version 1.1. It will not require a new conductor RPC
version unless the generic dispatch contract itself changes. The new
method will instead form part of its object's versioned contract.

Adding a remotable method, making an existing method remotable or changing
its signature changes the object's fingerprint. Authors will bump the
object's ``VERSION``, normally its minor version for a compatible addition,
and update the expected fingerprint in the object tests. Changes to method
behaviour or return values may also require a version bump even when the
fingerprint cannot detect them. A method-only addition does not necessarily
need field backleveling code, and neither the decorator nor the tests
increments the version automatically. Authors will also maintain list-object
versions and their member-version relationships as the contained objects
evolve.

These operations must preserve the request context, supported exception
propagation and object serialization over RPC. Collection results will use
versioned list objects with ``ListOfObjectsField`` members. The conductor
will use OVO's existing manifest-aware serialization for individual objects
and these lists, rather than add custom plain-container traversal or rely
on per-member compatibility RPCs. Unsupported versions must fail rather
than silently discarding information outside the object's compatibility
rules.

Expected unsupported operations will raise serializable Cyborg exceptions
rather than generic ``NotImplementedError``. This preserves a meaningful
failure on the caller without triggering OVO's fallback to legacy
indirection methods. We will not add legacy endpoints or retry the failed
operation through them.

Object compatibility does not tell an agent which RPC operations a
conductor supports. An old conductor may understand the object data but
lack the endpoint needed to execute the call. We therefore also need
operator-controlled RPC version caps. We will add two numeric options under
``[upgrade_levels]``:

* ``conductor`` defaults to ``1.1`` and applies to every ``ConductorAPI``
  client, including the agent's object indirection client.
* ``agent`` defaults to ``1.0`` and applies to every ``AgentAPI`` client.
  The agent RPC interface remains at version 1.0.

Each RPC client will pass its configured cap to oslo.messaging as
``version_cap``. Methods will still specify the RPC version they require.
The cap is an upper limit, not a request to send every call at that version,
so existing version 1.0 calls will stay at 1.0. With a conductor cap of 1.0,
the client must reject indirection calls requiring 1.1 before sending them.
It must not relabel them as version 1.0 or fall back to direct database
access. Configuration validation will reject malformed caps and versions
the local client does not support.

oslo.messaging enforces the supplied cap and offers ``can_send_version()``
to check it. It does not discover remote service versions or choose a
compatible method implementation. Object version manifests likewise do not
describe RPC endpoint availability. Automatic cap discovery through an
``auto`` setting is future work and is not supported by this change.

We will add ``ConductorAPI.supports_object_indirection()`` as a client-side
helper that returns ``self.client.can_send_version('1.1')``. It checks the
client's locally configured version cap without sending an RPC request or
querying a conductor. The name describes what the local client is allowed
to send, not a capability discovered on the server.

Dependent features that need indirection will call this helper during agent
startup, before initial discovery, reporting or other feature activity. If
an enabled feature needs indirection but the configured cap prevents it,
startup will raise a configuration exception identifying the feature,
required RPC version, configured cap and ``[upgrade_levels] conductor``
option. An agent with no
enabled feature requiring indirection can still run with a cap of 1.0.
For example, the PCI managed-mode feature will reject ``managed=false``
when the conductor cap is 1.0 because that configuration requires
indirection at version 1.1. Each dependent feature will implement and test
its own configuration consistency checks. Those checks are outside this
infrastructure spec's implementation and testing scope. Generic runtime
cap enforcement will remain in place as a safeguard.

This startup check only confirms that the local cap permits the generic
RPC operations. It cannot establish whether remote conductors implement a
particular object method. Dependent features still need compatible object
versions and conductor-first deployment ordering. Object backleveling
cannot supply methods missing from an older conductor.

This spec covers the indirection infrastructure and manual RPC caps.
Dependent features will select and decorate production object methods and
add any inventory APIs, metadata updates, PCI drivers or reconciliation
that they need. Existing purpose-built conductor RPC methods will remain
unchanged. Consolidating resource objects such as ``Device`` and
``DriverDevice`` into a single representation is outside the scope of this
change.

Alternatives
------------

One alternative is to add purpose-built RPC endpoints for each dependent
feature. This would duplicate object dispatch and compatibility handling.
Giving agents direct database access would instead expand their deployment
requirements and bypass the existing conductor boundary.

Data Model Impact
-----------------

This change requires no new object fields, object version changes, database
schema changes or data migrations. Agents use compatible object
representations rather than the database model.

REST API Impact
---------------

None.

Security Impact
---------------

Cyborg's message bus is private, and access to it is trusted in the same
way as root access to a service host. The conductor will resolve registered
object classes and invoke the requested method by name, following Nova's
trusted internal dispatch model. We will not add a method allowlist to
defend against callers who already have message-bus access. The remotable
decorator controls normal client-side indirection, not authorization.

Dispatched calls will preserve the request context and existing
authorization checks. This change adds no public API, credentials or
elevated privileges.

Other End User Impact
---------------------

None. This change requires no SDK or CLI changes.

Performance Impact
------------------

An agent's remotable call adds a synchronous conductor RPC round trip and
serialization overhead. Dependent features should prefer bounded, batched
operations over per-device calls in loops. Existing undecorated operations
are unaffected.

Other Deployer Impact
---------------------

Agents continue to use the existing messaging transport without database
access. No new service is required. Operators can set
``[upgrade_levels] conductor`` and ``[upgrade_levels] agent`` to numeric
RPC version caps. Their defaults are 1.1 and 1.0 respectively. There is no
``auto`` mode.

Operators will need to choose a cap supported by every service that the
client can target. All conductors will need to support the new indirection
endpoints before agents can use them. Pinning the conductor cap to 1.0
preserves existing calls but prevents features that require indirection
from working.

Developer Impact
----------------

Future features can mark selected object methods as remotable and use the
same object interface locally or from agents. Authors must preserve object
version compatibility and ensure that methods execute safely on the
conductor. Authors will update object versions and expected fingerprints
when they change the remotable contract. The new contract tests will make
unreviewed changes visible, but authors still need to assess compatibility
when updating those expectations. Features that depend on indirection will
also validate their required RPC version during agent startup. This change
does not alter the accelerator driver interface.

Upgrade Impact
--------------

Conductor RPC version 1.1 adds endpoints and keeps existing version 1.0
operations for older agents. All conductors will need to support the new
endpoints before agents use them. A configured cap does not discover
service capabilities or make unsupported calls compatible.

Cyborg currently requires coordinated upgrades of the API, conductor and
agents. This spec does not change that supported upgrade procedure. Older
agents do not contain code that calls the new indirection endpoints, so
they continue using the existing version 1.0 operations against an upgraded
conductor. That case needs no new RPC-cap override.

The caps limit outgoing client calls rather than change what a server
accepts. If an upgraded client could otherwise call endpoints unsupported
by its targets, an explicit conductor cap of 1.0 will prevent those calls.
It will also prevent features requiring indirection from starting. The
default cap of 1.1 permits indirection once all target conductors support
it. Pinning alone does not establish support for mixed service versions.
The agent cap remains at 1.0, matching its unchanged RPC interface.

Object version manifests and compatibility routines let newer conductors
communicate with older agent objects where the object's compatibility rules
allow it. The API and conductor still need database models compatible with
the deployed schema. Agents remain isolated from the database, and
indirection extends that isolation to new object operations. This helps
prepare Cyborg for upgrading central services and the database
first, then upgrading compute-node agents in batches.

This change does not enable rolling upgrades. Cyborg must also define and
test which service versions can run together, including their RPC
interfaces and object operations. Indirection cannot add endpoints to old
conductors, make arbitrary new object methods compatible with old agents,
or add indirection support to agents that predate this feature.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sean-k-mooney

Other contributors:
  None

Work Items
----------

* Add generic conductor RPC client and server operations at version 1.1.
* Raise the oslo.versionedobjects minimum version to 3.10.0.
* Wire agent startup and expose the object-layer remotable decorator.
* Add configurable conductor and agent RPC caps with numeric validation.
* Add the client-side ``supports_object_indirection()`` helper.
* Add Nova-style object contract fingerprint tests.
* Add hardware-independent dispatch and compatibility unit tests.
* Document object versioning, feature checks and upgrade ordering.

Dependencies
============

This change uses the existing oslo.versionedobjects and oslo.messaging
libraries and adds no new libraries. We will raise the oslo.versionedobjects
minimum version from 1.31.2 to 3.10.0 because it adds support for combining
``@classmethod`` with ``@remotable``. This lets us use the same remotable
decorator for class and instance methods rather than the deprecated
``@remotable_classmethod`` form.

The following features depend on this infrastructure, rather than being
prerequisites for it:

* `PCI VFIO variant-driver managed mode`_ needs persisted metadata and
  allocation state for configuration validation and startup reconciliation.
* `Generic NVMe driver with secure cleanup`_ needs persisted state for
  validation and startup recovery, and remote lifecycle updates when
  asynchronous cleanup completes or fails.

The NVMe feature also adds a separate conductor-to-agent ``cleanup_device``
RPC at agent RPC version 1.1. That is distinct from conductor indirection
RPC version 1.1 and is not introduced here. The base agent interface and
its default cap remain at 1.0 in this spec. The NVMe feature will define
its additional agent RPC requirement.

.. _PCI VFIO variant-driver managed mode:
   https://blueprints.launchpad.net/openstack-cyborg/+spec/support-vfio-variant-driver-managed-mode

.. _Generic NVMe driver with secure cleanup:
   https://blueprints.launchpad.net/openstack-cyborg/+spec/generic-nvme-driver-with-secure-cleanup

Testing
=======

Unit tests will cover local execution without indirection, agent startup
wiring, RPC version selection, class and instance dispatch, exception
propagation, instance field updates and change tracking. Exception tests
will verify that unsupported operations return serializable Cyborg errors
without triggering legacy fallbacks or retrying the operation.
Compatibility tests
will cover version manifests, nested objects, dedicated versioned lists
and unsupported versions. List tests will exercise members whose versions
differ between caller and conductor, verifying that OVO produces compatible
members without per-member compatibility RPCs.

Cyborg currently has no equivalent to Nova's object-version fingerprint
tests. We will add a suite using oslo.versionedobjects'
``fixture.ObjectVersionChecker`` and a checked-in map of expected
fingerprints for registered production Cyborg objects. Test-only object
registrations will not form part of that baseline.

The checker fingerprints fields and remotable method names and signatures.
A contract change will fail the comparison until an author reviews the
change, updates the object's version as required and records the new
expected fingerprint. We will follow the versioning rules described above.
The comparison detects changes, but does not itself prevent an author from
updating an expected hash without the required version bump. Review and
compatibility tests remain necessary for that, and for behaviour changes
that do not affect a fingerprint.

RPC client tests will cover both cap defaults and explicit overrides. They
will verify that a conductor cap of 1.0 allows existing calls but rejects
indirection calls before sending them, and that 1.1 allows indirection.
They will also check that agent calls stay at version 1.0, that malformed
caps, unsupported caps and ``auto`` are rejected, and that blocked calls
never fall back to direct agent database access.

Unit tests for ``supports_object_indirection()`` will verify that it returns
``False`` with a conductor cap of 1.0 and ``True`` with a cap of 1.1. They
will also verify that the helper sends no RPC request. Tests of dependent
features' startup validation remain outside this spec's scope.

This internal infrastructure will have unit coverage that needs no
accelerator hardware or full OpenStack deployment. We will adjust the
existing Grenade jobs as needed to verify RPC pinning when the configurable
caps are implemented. We will not add new Grenade tests or standalone
Tempest or other API-driven integration tests for indirection. Dependent
features will exercise it through their own API-driven scenarios.

Documentation Impact
====================

We will add a developer technical reference to Cyborg's contributor
documentation. It will explain indirection in more detail, using PCI
startup validation and NVMe cleanup completion to illustrate the need for
remote object operations. It will cover startup wiring, decorators, context,
serialization, object backleveling and tests. It will explain when authors
need to bump object versions and update expected fingerprints, and how
dependent features validate their RPC requirements at startup. It will also
distinguish object compatibility from RPC version caps and remote method
availability.

The configuration documentation will describe both ``[upgrade_levels]``
options, their defaults and limits, and the manual steps for changing caps
during upgrades. We will also add a release note covering indirection, RPC
caps and the conductor-first upgrade requirement.

References
==========

* `oslo.versionedobjects usage and indirection
  <https://docs.openstack.org/oslo.versionedobjects/latest/user/usage.html>`_
* `oslo.versionedobjects base interfaces`_
* `oslo.versionedobjects object fingerprint guidance`_
* `Nova object-version tests`_

.. _oslo.versionedobjects object fingerprint guidance:
   https://docs.openstack.org/oslo.versionedobjects/latest/reference/fixture.html

.. _Nova object-version tests:
   https://opendev.org/openstack/nova/src/branch/master/nova/tests/unit/objects/test_objects.py

.. _oslo.versionedobjects base interfaces:
   https://docs.openstack.org/oslo.versionedobjects/latest/reference/base.html

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2027.1
     - Introduced
