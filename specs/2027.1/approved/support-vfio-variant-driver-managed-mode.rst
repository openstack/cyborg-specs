..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Support VFIO variant driver managed mode in Cyborg
==================================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/support-vfio-variant-driver-managed-mode

This spec defines how Cyborg reports PCI managed mode to Nova so that
accelerators using VFIO variant drivers can be assigned to guests without
changing their host-driver binding. Existing deployments retain managed
mode by default.

Problem Description
===================

Overview
--------

Cyborg's compute-host agent uses accelerator drivers to discover devices.
It reports that inventory to the conductor, which persists it and reports
resources to Placement. The generic PCI driver selects devices using
operator-defined inclusion rules. Before this change, operators configure
those rules as JSON entries in ``[pci] passthrough_whitelist``, matching
properties such as PCI address, vendor ID and product ID.

Cyborg returns bound Accelerator Requests (ARQs) containing attach handles.
An attach handle describes how to attach an allocated device. For a PCI
device, it includes the PCI address. Nova's virt driver also needs to know
whether the hypervisor should manage the device's host-driver binding.
Managed mode allows the hypervisor to rebind the device before presenting
it to the guest and restore its host-driver binding afterwards. Cyborg
must report this policy without knowing which Nova virt driver consumes it.

Some PCI accelerator Virtual Functions (VFs) expose a VFIO interface through
a device-specific kernel driver rather than the generic ``vfio-pci`` driver.
These devices must retain their host-driver binding while assigned to a
guest. The operator prepares the device before assignment, and the
hypervisor must leave that binding unchanged. Nova's native PCI passthrough
supports this policy through ``managed`` in ``[pci] device_spec``. The
pre-change Cyborg attach handle lacks an equivalent field.

Existing technical debt
-----------------------

`Bug #2017513`_ tracks reconciliation trying to delete Placement resource
providers for devices that are missing from discovery but still allocated.
Placement rejects those deletions because the resources are in use.
Retaining those devices and their allocations is a prerequisite for this
proposal. A configuration change must not discard a device's persisted
identity or ownership merely because discovery stops reporting it.

The compute-host agent also lacks a general way to read persisted metadata
and allocation state without database access. The separate versioned-object
indirection proposal adds conductor-executed object operations for those
reads and for transactional updates. This feature needs that interface to
validate configuration against the devices already assigned to guests.

`Bug #2167353`_ tracks renaming ``passthrough_whitelist`` to ``device_spec``
as part of Cyborg's effort to remove whitelist terminology. This proposal
includes the rename, following the naming adopted for Cyborg's new driver
proposals.

`Bug #2167145`_ describes the PCI driver silently accepting unrecognized
JSON keys in inclusion rules. A misspelled key can leave an operator's
intended restriction unapplied without any indication of the mistake.
JSON Schema validation of ``device_spec`` will address this bug as part
of the configuration changes.

The existing generic solid-state drive (SSD) driver does not properly
clean devices between users. Its experimental status does not address
the resulting multi-tenancy risks. The driver is planned for deprecation
and replacement by the separate NVMe driver, which will provide cleanup.

Design constraints
------------------

Changing managed mode while a device is allocated is invalid because it
changes how the hypervisor handles a device assigned to a guest. The
compute-host agent must detect this conflict during startup discovery,
before reporting resources. It needs persisted attach-handle metadata and
allocation state to perform that check. This is agent-side configuration
validation, not a decision deferred to conductor resource reporting.
The conductor still executes database reads and transactional updates.
A permitted metadata update must preserve the handle that ARQs reference,
rather than delete and recreate it.

The long-term architecture moves Placement reporting and responsibility
for persisted device state into the agent's resource tracker. The
conductor will provide object indirection so that the agent can manage
that state without direct database access. This proposal keeps PCI
configuration validation in the agent and uses conductor-executed object
operations without adding new conductor-owned policy. Moving the remaining
inventory and Placement responsibilities is a separate refactoring.

Operators also need to exclude exact device addresses from broad inclusion
rules. A host may need one NIC for management or storage traffic while
exposing others with the same vendor and product IDs. Some accelerator
Physical Functions (PFs) and VFs share those IDs, so excluding the PF must
leave matching VFs eligible. Excluding an allocated device must retain its
tracking and allocations through the missing-device fix described above.

A stateful device retains workload data that needs preservation or transfer
when an instance moves. Nova needs to distinguish these devices from
stateless devices before deciding whether resize or cold migration is
safe. The pre-change attach-handle response does not report statefulness.
Non-Volatile Memory Express (NVMe) storage devices are stateful, and the
generic PCI driver lacks the lifecycle handling needed to manage them
safely. Selected NVMe devices must therefore be rejected by default.
A temporary upgrade workaround can allow them at the operator's risk,
but reporting statefulness alone cannot make that use safe.

A device's host NUMA node describes its locality to host CPUs and memory.
Nova needs that information to model device locality in the guest's
virtual topology. Statefulness and NUMA-locality reporting are stretch
goals for this feature. They can follow managed-mode support in a later
API microversion without delaying it. Neither reporting goal implements
guest topology, resize, cold migration or live migration.

.. note::

   NUMA affinity and NUMA locality have different purposes. NUMA affinity
   policies constrain placement during scheduling. They determine which
   hosts and devices are eligible and how devices must relate to the VM's
   NUMA topology. Required affinity is a hard scheduling constraint.
   Preferred policies can allow placement without that affinity.

   NUMA locality reports topology information to Nova and, through future
   guest-topology support, to the VM. Nova and the guest can then make
   informed decisions about how to use assigned devices. For example, the
   guest kernel can use device locality when scheduling applications that
   interact with a device.

   Reporting locality does not enforce affinity. The locality stretch goal
   supplies host information to Nova. Exposing it in the guest's virtual
   topology remains future work.

Use Cases
---------

* As an operator, I would like to assign accelerator VFs using VFIO variant
  drivers because these devices need to retain their host-driver binding
  while assigned to a guest.
* As an operator, I would like to keep managed mode enabled by default
  because existing deployments must retain their assignment behaviour
  without configuration changes.
* As an operator, I would like to change managed mode for unallocated
  devices without recreating their attach handles, and have Cyborg reject
  changes for allocated devices, because an active guest's device binding
  must not change with the agent's configuration.
* As an operator, I would like to include devices by vendor and product ID
  and exclude individual addresses because host NICs and accelerator PFs
  may share IDs with devices I want Cyborg to manage.
* As an operator, I would like to retain my existing PCI configuration
  during the 2027.1 upgrade because I need time to adopt ``device_spec``
  before the old-name fallback is removed in 2027.2.
* As a Nova developer, I would like to know whether a device is stateful
  because resize and cold-migration safety decisions depend on whether
  its data needs preservation or transfer.
* As a Nova developer, I would like device NUMA locality information
  because future guest topology support needs to describe device locality
  without treating that reporting as an affinity constraint.
* As a Nova developer, I would like one managed-mode field for PCI attach
  handles because its interpretation should not depend on which Cyborg
  driver produced the handle.

Proposed Change
===============

The generic PCI driver will accept an optional managed-mode setting,
persisted in PCI attach-handle information and exposed through ARQ API
microversion 2.4. Device statefulness and host NUMA locality are stretch
goals for all attach-handle types. Their implementation is planned later
in the series, preferably before the 2.4 API change lands. Deferred fields
will use later microversions. A companion Nova change will use managed
mode to control whether the hypervisor manages the device's host-driver
binding. Before reporting discovered PCI resources, the agent will validate
inclusion rules, apply exact-address exclusions and check device classes.
It will compare managed mode with persisted state and reconcile changes
that are safe for unallocated handles.

The PCI configuration changes apply to the generic PCI driver. The
unsafe-device workaround will also gate use of the existing generic SSD
driver. Other drivers that produce PCI attach handles can adopt
managed-mode reporting later. Resize, migration, guest PCIe topology
generation, automatic vendor-device preparation and a generic metadata
schema are outside this proposal's scope.

Configuration and discovery
---------------------------

The ``[pci] passthrough_whitelist`` option will be renamed to
``[pci] device_spec``. Each entry will remain a JSON inclusion rule, with
existing address matching and support for multiple entries preserved.
Code, examples and documentation will use ``device_spec``. Compatibility
with the old name is described under `Upgrade Impact`_.

The driver will validate each ``device_spec`` entry against a JSON Schema
during loading, before discovery or resource reporting. The schema will
cover supported keys and value types, rejecting unrecognized keys and
invalid values with a configuration error that identifies the offending
entry. Existing supported wildcard and regex address matching will remain
valid. The exact-address restriction applies only to ``exclude_list``.

The generic PCI driver will accept ``managed`` in ``device_spec`` entries.
The supported values are ``true`` and ``false``. The driver will normalize
the configured value to a boolean before reporting the attach handle.
For example::

  [pci]
  device_spec = {"vendor_id": "10de",
                 "product_id": "25b6",
                 "address": "0000:25:00.4",
                 "managed": false}

``managed=true`` allows Nova's virt driver to ask the hypervisor to rebind
the device for guest assignment and restore its host-driver binding
afterwards. ``managed=false`` leaves that binding unchanged. The operator
must already have bound the device to an assignment-capable driver and
completed any vendor-specific setup. Neither Cyborg nor Nova will perform
that preparation as part of this feature.

An omitted value means ``true``. Discovery may leave the key absent in that
case. Reconciliation and API serialization will both use the same default.

Exact-address exclusions
------------------------

The new ``[pci] exclude_list`` option will contain PCI address strings
and default to an empty list. Each entry must be a complete PCI address in
``domain:bus:slot.function`` form. Wildcards and regular expressions are
not supported. The driver will reject invalid entries when it loads.

An exclusion takes precedence over any matching ``device_spec`` inclusion.
It does not include a device by itself, and its effect does not depend on
the order of inclusion entries. An address excludes only the device at
that address. Excluding a PF does not exclude its VFs.

For example, an operator can include NICs by vendor and product ID while
keeping one NIC for host traffic::

  [pci]
  device_spec = {"vendor_id": "8086", "product_id": "1533"}
  exclude_list = 0000:03:00.0

Other matching NICs remain eligible. For an accelerator whose PF and VFs
share vendor and product IDs, the operator can exclude the PF while
retaining matching VFs::

  [pci]
  device_spec = {"vendor_id": "10de", "product_id": "1234",
                 "managed": false}
  exclude_list = 0000:06:00.0,0000:07:00.0

The product ID is illustrative. The driver excludes only the two named
devices. Their VFs remain subject to normal inclusion matching. These are
the exclusion use cases discussed for Nova PCI grouping. This proposal
does not add PCI grouping or an ``exclude`` boolean to device specs.

An excluded allocated device will follow the missing-device behaviour
under `Startup and periodic reporting`_. The exclusion will not stop agent
startup or delete ARQs. Removal remains subject to the missing-device
reconciliation checks.

Internal device-class exclusions
--------------------------------

PCI devices report a class code identifying their function, such as an
NVMe storage controller. The generic PCI driver will check selected
devices against an internal set of excluded classes, including NVMe.
This is a reusable driver safety check, not an operator-configurable list.
It is separate from the exact-address ``[pci] exclude_list`` option.

The driver will apply inclusion matching and exact-address exclusions
before checking device classes. By default, selecting a device from an
excluded class will raise a configuration error identifying its PCI
address, class and the generic PCI driver. The agent must not advertise
that device or submit the resource report. Devices outside the selected
set will not cause a class-exclusion error.

The class check can be bypassed only through the temporary option
described under `Unsafe-device workaround`_.

Cyborg will not automatically route a rejected device to another driver.
Existing allocated devices will retain their tracking and allocations as
described under `Startup and periodic reporting`_.

Unsafe-device workaround
------------------------

The ``[workarounds] allow_unsafe_devices`` option will default to ``false``.
Enabling it will bypass the generic PCI driver's excluded-class check and
permit the existing generic SSD driver to load. Enabling the SSD driver
without this workaround will be a startup configuration error, even when
no devices are discovered.

Both drivers will issue the same prominent unsafe-device warning for
each admitted NVMe or SSD device, identifying its address and producing
driver. The warning will explain the lack of safe cleanup and the
operator's responsibility for all multi-tenancy issues arising from that
use. These warnings and startup gates do not depend on either reporting
stretch goal. Statefulness reporting does not provide cleanup or data
protection.

The workaround will not bypass schema validation, inclusion matching,
address exclusions or allocation protection. It will not automatically
migrate devices or allocations to the new NVMe driver. Deprecation and
removal conditions are described under `Upgrade Impact`_.

Device statefulness and NUMA locality
-------------------------------------

The reporting stretch goals will add driver-supplied ``stateful`` and
``numa_node`` values to attach information. Their response types, defaults
and microversion rules are defined under `REST API Impact`_.

``stateful`` identifies devices that retain workload data. Nova can use
this information when deciding whether resize or cold migration is safe.
A stateful device requires preservation or transfer of that data; a
stateless device does not require device-data transfer.

Once statefulness reporting is implemented, NVMe devices admitted by the
generic PCI driver through ``[workarounds] allow_unsafe_devices`` and every
device reported by the existing generic SSD driver must report
``stateful=true``. The separate NVMe driver must use the same value in API
versions that expose the field.

Drivers will report known host locality through ``numa_node`` in attach
information. This describes the device's host NUMA node, not a guest NUMA
node assignment.

Persisted state and reconciliation
----------------------------------

The conductor owns database access. Its inventory includes devices,
their control paths and deployables, and attach handles. A control path
identifies how Cyborg manages a device. A deployable represents a resource
that Cyborg can assign from that device.

Agents will use the object-indirection interface from the separate
versioned-object specification to read those records for their host.
This feature will provide the required remotable object operations,
executed on the conductor. Inventory collections will use versioned list
objects so the conductor can return lists compatible with the agent's
object versions. This feature will not add direct agent database access
or a separate conductor RPC endpoint for each lookup.

The agent will use the control path, deployable and normalized PCI address
to match discovered handles with persisted handles. The ``managed`` value
is metadata, not part of that identity. Reconciliation will compare effective
boolean values, so an absent ``managed`` key and an explicit ``true`` do
not cause handle replacement. Adding or refreshing ``stateful`` and
``numa_node`` must also preserve handle identity rather than remove and
recreate the handle. Metadata reconciliation will support those updates
without authorizing new lifecycle operations on an allocated device.
Other attach-handle metadata will retain its existing discovery
reconciliation behaviour.

Whenever effective managed mode changes, including alongside other
metadata, the agent will check persisted allocation state. A handle remains
allocated for this check if ``in_use`` is set or an ARQ still references
it. If the handle is allocated, the agent will reject the change and report
the host, PCI address and conflicting values.
The operator must restore the previous configuration or release the
affected allocation before changing managed mode.

For an unallocated handle, the agent will request an in-place metadata
update through the conductor, preserving the handle's UUID and topology.
Within a transaction, the update will compare the expected topology,
metadata and allocation state with the current database state, including
ARQ references. If that state changed after the lookup, the update will
fail rather than overwrite it. The agent will read the current state and
retry once. If another request allocated the handle in the meantime, the
agent will reject the managed-mode change.

For example, a persisted handle for ``0000:25:00.4`` with no managed marker
has an effective value of ``true``. If discovery reports ``false`` and the
handle is unallocated, the conductor will update that handle's metadata
without changing its UUID. If the handle is allocated, Cyborg will retain
the persisted value and reject the report. A later report with no effective
managed-mode change will retain the existing metadata representation.

Startup and periodic reporting
------------------------------

An RPC version cap is an operator-set limit on the versions a service can
use for outgoing RPC calls. The object-indirection prerequisite adds
``[upgrade_levels] conductor`` to control calls to the conductor.

The generic PCI driver's ``init_host()`` will perform the first phase of
startup validation using local configuration only. If any entry in
``[pci] device_spec`` explicitly specifies ``managed``, whether ``true`` or
``false``, it will call ``ConductorAPI.supports_object_indirection()``.
Entries supplied through the deprecated option name will receive the same
check.

The helper checks whether the configured conductor RPC cap permits version
1.1. If it does not, startup will raise a configuration error identifying
managed-mode support, required RPC version 1.1, the configured cap and
``[upgrade_levels] conductor``. This check sends no RPC request and does
not discover remote service versions or establish that all services are
upgraded. Conductors must already support the required object operations.

Only after driver startup validation passes will the agent run initial
discovery and persisted-state reconciliation synchronously, before normal
periodic reporting. This second phase checks for changes affecting
allocated handles. A managed-mode conflict on an allocated handle will
prevent startup. Invalid PCI attach-handle data will also fail validation
rather than silently change assignment behaviour. Periodic reporting uses
the configuration loaded at startup. A configuration conflict will reject
that report and produce an error log without stopping the running agent.
Selecting a device from an internally excluded PCI class, with the class
check enabled, will follow the same startup and periodic-reporting error
policy. Transient initial reporting failures will be logged and retried
through periodic reporting.

A missing allocated device is a different condition from a managed-mode
change on a device still being reported. The fix for bug #2017513 will
retain missing allocated resources, reserve their Placement inventory
against new allocations and log their absence. Their absence alone must
not prevent agent startup. This proposal uses that behaviour rather than
redefining device removal or deleting ARQs automatically.

ARQ information and Nova consumption
------------------------------------

The generic PCI driver will supply the configured ``managed`` value in
its reported attach information. The conductor will persist it as JSON in
``AttachHandle.attach_info``. API serialization will expose attachment
details through ``attach_handle_info`` using the contract under
`REST API Impact`_.

Nova will request the new Cyborg microversion when using this feature.
Its virt driver will read ``attach_handle_info.managed`` for PCI ARQs and
use it to control hypervisor management of the host-driver binding. This
applies to any Cyborg driver that supplies a PCI attach handle. An absent
key retains the existing default of managed mode enabled.

For example, Nova's libvirt driver will translate ``managed=false`` into
XML equivalent to::

  <hostdev mode='subsystem' type='pci' managed='no'>
    <driver name='vfio'/>
    <source>
      <address domain='0x0000' bus='0x25' slot='0x00' function='0x4'/>
    </source>
  </hostdev>

An explicit ``true`` or missing marker will preserve ``managed='yes'`` for
KVM/QEMU. Enablement ordering is described under `Upgrade Impact`_. This
proposal does not add automatic detection of old Nova consumers.

Alternatives
------------

Discovery-only metadata updates would leave the agent unable to validate
configuration against persisted allocation state. Treating a metadata
change as handle removal and recreation would also discard stable identity.
Conductor-executed object operations and stale-state checks avoid those
problems.

Top-level ARQ fields would put device properties on the request rather
than its attach handle. Dedicated database columns or a new metadata table
are unnecessary because the existing attach information already stores
JSON. Extending that mapping avoids introducing a separate storage schema.

An ``exclude`` boolean inside device specs would mix inclusion and
exclusion rules in the same list. A separate list of exact addresses meets
the host-device and PF exclusion use cases without adding wildcard or
regular-expression exclusion rules.

Implementing every PCI-producing driver in this cycle would broaden the
work without changing the Nova contract. Other drivers can adopt that
contract independently.

Data Model Impact
-----------------

No database schema change or data migration is required. Managed mode and
the reporting stretch goals will use the existing attach-handle JSON.
Existing rows remain valid. Exclusions and the configuration rename require
no new persisted fields.

The ARQ object mapping must preserve JSON booleans for managed mode and,
as the stretch goals are implemented, statefulness booleans and NUMA node
integers and nulls. Values must not be converted to strings. Object
versions will be bumped with conversion rules for older representations
as these changes land.
Remotable method additions and versioned list objects will follow the
indirection specification's versioning rules, including member-version
relationships and tests that detect object-contract changes.

REST API Impact
---------------

Microversion 2.4 changes the ARQ representation returned by:

* ``GET /accelerator/v2/accelerator_requests/{uuid}``
* ``GET /accelerator/v2/accelerator_requests``, including existing
  ``instance`` and ``bind_state`` filters.

Requests retain their existing structure. Successful HTTP 200 responses
add the nested keys described below. There are no new URLs, request
parameters, request bodies or policy rules. ARQ creation does not assign
a PCI attach handle, so this feature adds no creation-time input.

The required change for a bound PCI ARQ in microversion 2.4 is::

  {
    "attach_handle_type": "PCI",
    "attach_handle_info": {
      "domain": "0000",
      "bus": "25",
      "device": "00",
      "function": "4",
      "managed": false
    }
  }

``managed`` is required for PCI attach handles and is a JSON boolean.
It is ``true`` if the stored value is absent. It is not added to non-PCI
attach handles.

The target is to include the stretch-goal fields in the same API change.
If either is deferred until after 2.4 lands, that field requires a later
microversion. An already-published microversion must not gain new fields.
From its introducing microversion, ``stateful`` will be required for all
attach-handle types as a JSON boolean, defaulting to ``false``.
``numa_node`` will likewise be required for all attach-handle types from
its introducing microversion. It will be a non-negative integer or JSON
``null``, defaulting to ``null`` when stored locality is missing or unknown.
Host NUMA node IDs are zero-indexed, so ``0`` is a valid node ID.

If both stretch goals land before the 2.4 API change, the target response
for a stateless PCI device on host NUMA node 0 is::

  {
    "attach_handle_type": "PCI",
    "attach_handle_info": {
      "domain": "0000",
      "bus": "25",
      "device": "00",
      "function": "4",
      "managed": false,
      "stateful": false,
      "numa_node": 0
    }
  }

ARQs without an attach handle gain no fields. Other keys in each mapping
retain their current contract. Tenants cannot supply these properties in
ARQ requests.

Microversions before 2.4 omit ``managed``. Each stretch-goal field is
omitted before its own introducing microversion, including from 2.4 if
deferred. Filtering must not mutate the underlying ARQ object or persisted
metadata.

Security Impact
---------------

Only operators can configure managed mode. Tenants cannot override it
through device profiles or ARQ requests. Incorrect configuration can make
a device unusable or expose bugs in its host driver. Operators remain
responsible for preparing devices and choosing a mode their drivers support.

An operator enabling ``[workarounds] allow_unsafe_devices`` accepts
responsibility for all multi-tenancy issues, including isolation and
cleanup of data between users. Neither driver's warnings nor
``stateful=true`` reporting provide those protections.

Address exclusions are operator-controlled configuration. Statefulness and
NUMA locality are driver-reported properties, not tenant overrides. Incorrect
statefulness reporting can cause a consumer to treat a device containing
workload data as stateless. Consumers must respect an explicit
``stateful=true`` value when deciding which lifecycle operations are safe.

Agents continue to use the existing trusted message bus rather than database
credentials. Reconciliation adds no privileged host operation. Vendor setup
and driver binding retain their existing privilege requirements.

Other End User Impact
---------------------

End users will continue to request accelerators through device profiles.
They will not select managed mode themselves.

OpenStack SDK and the official OpenStackClient accelerator commands will
support microversion 2.4 and display managed mode in ARQ list and show
output. Support for statefulness and NUMA locality will follow whichever
microversions introduce them. Clients will preserve the fields' JSON
types in ``attach_handle_info``. No new command or option is required to
set these driver-reported properties. Tests and documentation will cover
the response changes as they are introduced.

This proposal adds no new functionality to ``python-cyborgclient``.
Client changes will use OpenStack SDK and official OpenStackClient.

Performance Impact
------------------

PCI reconciliation adds synchronous conductor RPC calls and database reads
before initial and periodic resource reports. These lookups cover the
host's persisted device topology and attach handles. Implementations should
avoid unnecessary per-handle calls where collection operations suffice.
A managed-mode change also adds a transactional metadata update and may
require one refresh and retry after a conflict.

Exclusion matching adds an exact-address membership check during PCI
discovery. Known NUMA locality can be read with the device's other discovery
information. ARQ serialization adds metadata defaults and microversion
filtering. There are no new periodic tasks or scheduler filters. The fix
for missing devices adds Placement reservations. Ordinary managed-mode
metadata updates do not.

Other Deployer Impact
---------------------

Agent reconfiguration requires a restart. Configuration is not reloaded
while the agent runs.

Operators must prepare devices requiring unmanaged assignment before the
agent reports them, including host-driver binding and per-VF setup.
Operators must not configure a device for both Cyborg management and Nova
native PCI passthrough.

A managed-mode conflict on restart requires the operator to restore the
previous configuration or release the affected allocation before changing
its mode. Other configuration corrections and compatibility requirements
are described under `Upgrade Impact`_.

The reporting stretch goals add no operator configuration options.

Developer Impact
----------------

Driver developers can emit managed mode in PCI attach information and,
as the stretch goals are implemented, statefulness and NUMA locality for
any attach-handle type. The common response contract applies independently
of the producing driver. No new accelerator driver method is required.

Upgrade Impact
--------------

Renaming the inclusion option requires no configuration change for the
2027.1 upgrade. oslo.config will accept ``passthrough_whitelist`` as a
deprecated name for ``device_spec`` throughout that release. Deprecating it
in the 2027.1 SLURP release gives operators time to migrate before its
fallback is removed in 2027.2. At that later upgrade, configuration still
using the old name must be changed to ``device_spec``.

The renamed option will use oslo.config's native deprecated-name support,
not a custom fallback parser. Entries supplied through either name will
undergo the same JSON Schema validation. Previously tolerated unknown keys
and invalid values must be corrected before restarting an upgraded agent.

Previously accepted inclusion rules selecting NVMe devices will cause a
startup error by default. Operators must narrow those rules, exclude the
device addresses or temporarily enable
``[workarounds] allow_unsafe_devices=true`` at their own risk.

The existing generic SSD driver will move from experimental to deprecated
because it does not properly clean devices. Deployments enabling it must
set ``[workarounds] allow_unsafe_devices=true`` before restarting an
upgraded agent, or startup will fail. Its replacement is the separate
NVMe driver with cleanup support.

The workaround will be deprecated for removal when introduced in 2027.1.
Removal requires both the separate NVMe feature to be available and at
least one release to have passed since the workaround was introduced.
The earliest removal is 2027.2, not a fixed removal deadline.

Managed-mode support requires the object-indirection API and is supported
only after all Cyborg services are fully upgraded. Conductors must provide
the required indirection endpoints and object operations before agents use
them. Object compatibility does not enable rolling upgrades or make mixed
service versions safe.

An explicit ``managed`` value in ``[pci] device_spec`` will trigger the
startup helper check described under `Startup and periodic reporting`_.
The configured conductor RPC cap must permit version 1.1 for either
``managed=true`` or ``managed=false``. Passing this local check does not
verify service versions or establish that the deployment is fully upgraded.

Before operators enable ``managed=false``, Cyborg API services must support
microversion 2.4 and Nova compute services consuming the affected ARQs must
request that version and apply the marker. Old clients and older API
microversions omit it. Nova's missing-value default preserves ordinary
managed devices but is unsafe for devices that require unmanaged assignment.

Existing attach handles require no bulk migration or recreation to receive
the response defaults. Discovering ``managed=true`` for an old handle
without the marker must preserve its identity, including while allocated.

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

* Rename the PCI inclusion option to ``device_spec`` with native
  oslo.config deprecated-name support, and apply the compatibility and
  deprecation schedule under `Upgrade Impact`_.
* Add JSON Schema validation and exact-address exclusions to PCI discovery.
* Validate and normalize managed configuration and report it in PCI
  attach-handle JSON.
* Add remotable inventory operations, versioned list objects and the
  transactional, stale-checked metadata update through object indirection.
  Reconcile managed mode before resource reporting without replacing
  handles or changing the mode of allocated devices.
* Add synchronous startup validation and the conductor RPC-cap check.
* Add internal device-class exclusions, the unsafe-device workaround,
  SSD-driver startup gating and shared per-device warnings. Deprecate the
  generic SSD driver in favour of the separate NVMe driver.
* Add API microversion 2.4, response filtering and API reference updates,
  preserving JSON field types and backleveling older object versions.
* Update Nova microversion negotiation and PCI managed-mode consumption.
* Extend existing pci-sim DevStack integration and PCI Tempest lifecycle
  scenarios for ``device_spec``, explicit API 2.4 requests and boolean
  assertions. Update the jobs to use ``managed=false`` for UART VFs.
* Add Cyborg configuration, allocation-safety, startup and API regression
  coverage, and update SDK and OpenStackClient accelerator support and
  tests.
* Later in the series, implement statefulness and NUMA locality as stretch
  goals, preferably before the API change lands. Preserve handle identity
  and apply the common defaults for all attach-handle types. Include
  ``stateful=true`` for admitted PCI NVMe and all generic SSD devices.
  Introduce deferred fields through later microversions.
* Update administrator documentation and release notes.

Dependencies
============

* The 2027.1 `versioned object indirection specification`_ supplies remote
  object execution, manual RPC caps and the startup RPC-cap check helper.
  PCI inventory operations and reconciliation belong to this feature.
* The fix for `bug #2017513`_ retains allocated resources missing from
  discovery and prevents new allocations from their Placement inventory.
* The companion `Nova blueprint`_ supplies microversion negotiation and
  managed-mode consumption for Cyborg PCI attach handles. It requires no
  Nova REST API change.
* OpenStack SDK and official OpenStackClient need support and tests for the
  new Cyborg microversion.
* This proposal addresses `bug #2167353`_ through the configuration rename
  and `bug #2167145`_ through JSON Schema validation. Neither fix is an
  external prerequisite.
* The separate NVMe driver and Nova lifecycle work are not prerequisites
  for managed-mode support. The driver will report statefulness when that
  field is supported; lifecycle work can consume it.
* Nova's PCI grouping proposal supplies related exclusion use cases, not a
  runtime dependency or a Cyborg grouping requirement.
* pci-sim, its DevStack setup and PCI Tempest lifecycle scenarios already
  provide the software-backed assignment test infrastructure.
* JSON Schema validation requires a validator. The implementation will
  identify the library and declare any additional runtime requirement.

.. _versioned object indirection specification:
   versioned-object-indirection.html
.. _Nova blueprint:
   https://blueprints.launchpad.net/nova/+spec/support-vfio-variant-driver-managed-mode-via-cyborg

Testing
=======

Cyborg unit, database, object and API tests will cover:

* Configuration loading through both ``device_spec`` and its deprecated
  name, including supported wildcard and regex inclusion matching,
  misspelled or unsupported keys and invalid value types. Exclusion tests
  will cover empty and multiple-address lists, precedence, malformed
  addresses, unsupported patterns and PF/VF independence with shared IDs.
* Managed-mode discovery and reconciliation, including equivalent omitted
  and explicit ``true`` values, stable UUIDs for unallocated updates and
  rejection while allocated. Allocation tests will include surviving ARQ
  references with ``in_use=false`` and references appearing between lookup
  and update. Database tests will exercise stale topology, metadata and
  allocation state; agent tests will cover refresh, one retry and rejection
  if the refreshed state is allocated.
* Startup and periodic-reporting behaviour for configuration conflicts,
  transient failures and missing or newly excluded allocated devices.
  RPC-cap tests will cover both explicit managed values, both option names
  and caps 1.0 and 1.1. They will verify that the PCI driver's ``init_host()``
  uses the helper and rejects a blocking cap before discovery or
  persisted-state reconciliation.
* Internal device-class checks for selected NVMe devices, unselected NVMe
  devices and supported classes. Tests will verify default rejection,
  startup failure versus periodic-report rejection, and admission with
  shared per-device warnings when the workaround is enabled. SSD tests
  will cover startup rejection without the workaround even with no devices,
  and successful loading with warnings when enabled. Configuration tests
  will verify that the class set is not operator-configurable and that the
  workaround is registered under ``[workarounds]``. It must not bypass
  schema validation, inclusion matching, address exclusions or allocation
  protection.
* Versioned inventory collections, JSON boolean preservation and object
  backleveling. The object-indirection prerequisite tests remote dispatch.
  API tests will cover ARQ show and list responses at 2.4 and earlier
  versions, defaults for old handles, invalid persisted values and filtering
  without source-object mutation.

Reporting tests will accompany the stretch goals when implemented. They
will cover PCI and non-PCI handles, stateful producers, host NUMA node 0,
unknown locality, JSON types and identity-preserving updates. Tests will
check each field's presence and defaults from its introducing microversion
and absence before it. Admitted PCI NVMe and every generic SSD device must
report ``stateful=true`` once that field is supported. Safety gates and
warnings must remain covered independently of these goals.

Existing pci-sim DevStack integration and PCI Tempest lifecycle scenarios
will be extended rather than replaced. Jobs will configure the UART VFs
with ``managed=false``. The scenarios will request API 2.4 explicitly and
assert boolean managed mode, reusing their ARQ binding, guest-device
visibility, lifecycle and ARQ-deletion checks. Older-release jobs will
retain version-appropriate assertions. Exact-address exclusion edge cases
will be covered by the targeted Cyborg tests above.

The NVMe and mdev feature series add other VF types and their
``managed=true`` paths. That work remains in those series rather than
being a prerequisite for this proposal's UART scenarios.

End-to-end compatibility with Nova will be qualified by the existing
Cyborg integration jobs, which run against both Cyborg and Nova changes.

The scenarios require a full deployment with pci-sim, VFIO and QEMU, but
no physical accelerator. Software-backed assignment does not establish
compatibility with every vendor driver or firmware combination. Real
VFIO variant-driver testing can supplement first-party CI.

Documentation Impact
====================

Cyborg administrator documentation will cover managed-mode configuration,
host preparation, allocation restrictions, startup errors and recovery.
It will include PCI selection and exclusion examples for host NICs and PFs,
and explain JSON Schema errors and internal device-class restrictions.
The unsafe-device workaround, SSD deprecation and NVMe replacement must
make the cleanup deficiency, warnings and operator responsibility for all
multi-tenancy issues explicit.

Configuration references, generated samples and deployment examples will
use ``device_spec``. Upgrade documentation and release notes will cover
the compatibility, deployment ordering and removal conditions under
`Upgrade Impact`_. DevStack and PCI scenario documentation will describe
the managed-mode additions to the existing pci-sim setup.

The API reference, version history, SDK and OpenStackClient documentation
will describe fields, types and defaults as each microversion introduces
them. Nova's Cyborg administrator documentation will cover consumption and
enablement ordering. Driver documentation will explain the reporting
stretch goals without implying cleanup, data-transfer or guest-topology
support. Release notes will identify which reporting goals were delivered.

References
==========

* `2026.2 managed-mode proposal
  <../../2026.2/approved/support-vfio-variant-driver-managed-mode.html>`_
* `Kernel VFIO PCI variant driver documentation
  <https://docs.kernel.org/driver-api/vfio-pci-device-specific-driver-acceptance.html>`_
* `Nova VFIO variant driver managed-mode specification
  <https://specs.openstack.org/openstack/nova-specs/specs/2025.1/implemented/enable-vfio-devices-with-kernel-variant-drivers.html>`_
* `Nova/Cyborg interaction
  <https://specs.openstack.org/openstack/cyborg-specs/specs/ussuri/implemented/nova-cyborg-interaction.html>`_
* `Libvirt host device assignment
  <https://libvirt.org/formatdomain.html#host-device-assignment>`_
* `Bug #2017513`_, retain allocated devices missing from discovery rather
  than trying to delete their in-use Placement resource providers.
* `Bug #2167353`_, rename the PCI inclusion configuration option from
  ``passthrough_whitelist`` to ``device_spec``.
* `Bug #2167145`_, validate PCI inclusion rules rather than silently accept
  unrecognized JSON keys.
* `Cyborg 2027.1 PTG notes
  <https://etherpad.opendev.org/p/r.42ca3bf2adcaa246f111a7b92b1f9c52>`_
* `Nova PCI grouping proposal
  <https://specs.openstack.org/openstack/nova-specs/specs/2027.1/approved/pci-passthrough-groups.html>`_

.. _bug #2017513: https://bugs.launchpad.net/openstack-cyborg/+bug/2017513
.. _bug #2167353: https://bugs.launchpad.net/openstack-cyborg/+bug/2167353
.. _bug #2167145: https://bugs.launchpad.net/openstack-cyborg/+bug/2167145

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2026.2
     - Introduced.
   * - 2027.1
     - Reproposed after the unmerged 2026.2 prototype exposed the need for
       persisted-state validation and identity-preserving metadata updates.
       Added exact-address exclusions, internal device-class checks and
       the PCI configuration rename and deprecation. Statefulness and
       NUMA reporting are stretch goals.
