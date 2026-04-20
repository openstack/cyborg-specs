..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Generic NVMe driver with secure cleanup
=======================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/generic-nvme-driver-with-secure-cleanup

In OpenStack there are several ways to pass-through generic PCI devices
including a dedicated Inspur NVMe Cyborg driver, Nova PCI pass-through
and the Cyborg generic PCI driver. None of these existing mechanisms
support multi-tenancy when used with NVMe devices nor are suitable for
use in a public cloud as a result. Once the instance gets deleted, there
is no way to securely erase data from an NVMe device with OpenStack.
The same NVMe device gets allocated to another instance without cleanup.
This can leak sensitive information between tenants and may cause issues
with workloads by hijacking the boot process and compromising the guest.
Basically there is no way to manage the life cycle of an NVMe device in
OpenStack.

This blueprint proposes a generic Cyborg NVMe driver that manages the
whole lifecycle of NVMe devices. It includes always binding a clean NVMe
device to a new instance and securely cleaning up the NVMe device after
deallocation for reuse.

Problem description
===================

OpenStack lacks automated lifecycle management for NVMe devices. When a
Cyborg-managed NVMe device is detached from an instance, tenant data
remains on the device. Without automated sanitization, operators must
manually track device allocation, run ``nvme sanitize`` or
``nvme write-zeroes`` commands via SSH after instance deletion, verify cleanup
completion, and ensure devices are not reallocated during cleanup. This
manual process is time-consuming, error-prone, and does not scale.

Additionally, Cyborg's existing Inspur NVMe driver locks operators into
a single hardware vendor. Operators cannot manage NVMe devices from
other vendors through Cyborg without vendor-specific driver
implementations for each manufacturer.

Cyborg needs a generic NVMe driver that discovers devices from any
vendor, manages allocation and binding, and performs automated secure
cleanup using NVMe sanitize/zero commands before devices are returned
to the available pool.

Use Cases
---------

* As an operator, I want Cyborg to discover and manage NVMe devices
  from any vendor without requiring vendor-specific drivers, so I am
  not locked into a single hardware manufacturer.

* As an operator, I want NVMe devices automatically sanitized after
  instance deletion, so tenant data cannot leak to subsequent instances
  without manual intervention.

* As an operator, I want failed cleanups to block device reallocation,
  so devices with residual data are never assigned to new tenants.

* As an operator, I want visibility into cleanup status and manual
  recovery tools, so I can diagnose and resolve stuck cleanup
  operations.

* As a cloud user, I want assurance that my NVMe device contains no
  data from previous tenants, so my application does not encounter
  unexpected data corruption or security violations.

* As a Cyborg developer, I want a base driver cleanup interface that
  vendor-specific drivers can override, so cleanup behavior remains
  extensible without changing the conductor unbind contract.

Proposed change
===============

This blueprint adds a generic NVMe driver (``NVMeDriver``) as a
standalone driver. Two new methods are added to the ``GenericDriver``
base class (``cyborg/accelerator/drivers/driver.py``): ``init_host()``
and ``cleanup(device)``, both as no-op (``pass``) defaults so that
existing drivers are unaffected. The Cyborg agent remains driver-type agnostic.
Vendor-specific drivers can be further created by inheriting from
``NVMeDriver``::

        GenericDriver (cyborg/accelerator/drivers/driver.py)
        ├── init_host()             → no-op (pass); subclasses override
        ├── discover()              → subclasses override
        └── cleanup(device)         → no-op (pass); subclasses override

        NVMeDriver (new, cyborg/accelerator/drivers/nvme.py,
                    standalone driver)
        ├── init_host()             → validates nvme-cli is installed
        ├── discover()              → sysfs PCI enumeration filtered by device_spec,
        │                              NVMe capability detection via nvme id-ctrl
        └── cleanup(device)         → nvme-cli sanitize/zero

        VendorNVMeDriver (future vendor-specific drivers)
        └── cleanup(device)         → override with vendor tools

In the ``NVMeDriver`` class, the ``init_host()`` method validates that
``nvme-cli`` is installed on the cyborg-agent node. The ``discover()``
method filters out NVMe devices and reports NVMe device capabilities to
Placement. The ``cleanup()`` method performs cleanup of NVMe devices
using ``nvme-cli``. Bind and unbind use the common Cyborg conductor
process for all device types.

NVMe Device Lifecycle Flow
--------------------------

::

    ┌─────────────────────────────────────────────────┐
    │  Operator configures cyborg.conf                │
    │  [agent] enabled_drivers = nvme_driver          │
    │  [nvme]  device_spec = {vendor_id, product_id,  │
    │          address, clear_action, clear_strategy}  │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │  cyborg-agent starts                            │
    │  Driver.init_host() called for each driver      │
    │  NVMeDriver validates nvme-cli is installed     │
    │  (see Timeout and Crash Recovery for agent      │
    │   restart behavior)                             │
    │  NVMeDriver.discover() runs, matches device_spec│
    │  Creates RP with resource class                 │
    │    CUSTOM_NVME_<VENDOR_ID>_<PRODUCT_ID>         │
    │  Reports NVMe capability traits to Placement    │
    │    (CES, BES, WZS)                              │
    │  Resolves cleanup action from policy matrix     │
    │  device_state → available                       │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │  User creates instance with NVMe device profile │
    │  Nova schedules → Cyborg allocates device       │
    │  Cyborg conductor binds device to instance      │
    │  Placement: reserved = total  (set at BIND)     │
    │  Guard: reject bind if device_state != available│
    │  device_state → allocated                       │
    └─────────────────┬───────────────────────────────┘
                      │  User deletes instance
                      ▼
    ┌─────────────────────────────────────────────────┐
    │  Nova deletes instance                          │
    │  libvirt rebinds device to host NVMe driver     │
    │  Nova calls DELETE /v2/accelerator_requests     │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │  Cyborg conductor (unbind):                     │
    │  dispatch cleanup_device RPC to agent (async)   │
    └─────────────────┬───────────────────────────────┘
                      │
              (async — Nova does not wait,
               instance deletion is complete)
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │  Cyborg agent:                                  │
    │  device_state → pending_cleaning                │
    │  device_state → cleaning                        │
    │  resolve NVMe controller from PCI address       │
    │  execute locked-in cleanup action               │
    │    (sanitize or zero-write)                      │
    │  bounded by [nvme] cleanup_timeout              │
    └──────────┬──────────────────────┬───────────────┘
               │ SUCCESS              │ FAILURE/TIMEOUT
               ▼                      ▼
    ┌─────────────────────┐  ┌────────────────────────┐
    │  reserved = 0       │  │  reserved = total      │
    │  device_state →     │  │  device_state → error  │
    │    available         │  └──────────┬─────────────┘
    └─────────────────────┘             │ operator calls
                                        │ POST /v2/devices
                                        │   /{uuid}/clean
                                        ▼
                             ┌────────────────────────┐
                             │  Re-triggers cleanup   │
                             │  (same locked-in action)│
                             │  device_state →        │
                             │    pending_cleaning    │
                             └────────────────────────┘

Scope
-----

This spec covers local NVMe PCI controllers present on the host PCI
bus.

The following items are explicitly out of scope for this spec:

* **NVMe-oF/TCP/RDMA** (fabric-attached storage) would require its own
  driver. Cinder already provides NVMe-oF capabilities.
* **Instance resize and migration** for instances with Cyborg-managed
  NVMe devices. NVMe devices are stateful and do not support data
  transfer.
* **Cleanup guarantee traits** describing the configured policy (as
  opposed to hardware capability traits). Currently only hardware
  traits (CES, BES, WZS) are reported. A future spec could add traits
  that reflect the operator-selected cleanup guarantee.
* **Cleanup policy as an API attribute** per device. Currently
  cleanup policy is operator-only configuration in ``device_spec``.
* **Encryption key cleanup for the zero path**. Sanitize CES handles
  key rotation inherently; ensuring no stale keys remain for the zero
  path is deferred to the implementation.
* **Minimal privsep context**. Currently all NVMe operations require
  ``CAP_SYS_ADMIN``. Investigating a lesser capability set is a future
  improvement.
* **Driver-independent capability model**. Currently cleanup support
  is determined by ``device.type == 'NVME'``. A general capability
  model — either a ``device_metadata`` JSON blob
  (e.g. ``{"capabilities": ["SUPPORTS_CLEANING",
  "SUPPORTS_PROGRAMMING"]}``) or device attributes
  (``CAP_SUPPORTS_CLEANING=True``) — would allow driver-independent
  decisions. The interaction with the existing attributes API needs
  design work and is deferred to the driver framework spec.

init_host
---------

The Cyborg agent startup follows this order: the agent's ``init_host()``
runs first, then the ``update_available_resource`` periodic task calls
``discover()`` on each enabled driver. This spec adds a driver-level
``init_host()`` hook that the agent will call for each enabled driver
before discovery runs. The ``GenericDriver`` base classes provide
no-op (``pass``) defaults. ``NVMeDriver`` overrides
``init_host()`` to validate that ``nvme-cli`` is present on the host,
following the same pattern as Nova's libvirt driver which validates
minimum libvirt and QEMU versions at startup. If ``nvme-cli`` is not
found and the NVMe driver is configured in ``[agent] enabled_drivers``,
``init_host()`` raises an exception and prevents the agent from
starting. If no NVMe devices are found during discovery,
the driver returns an empty list without error.

Device Discovery
----------------

The NVMe driver is enabled by adding ``nvme_driver`` to
``[agent] enabled_drivers`` and configuring ``device_spec`` under the
``[nvme]`` section in ``cyborg.conf``. The NVMe driver uses
``device_spec`` as the configuration option name. The existing PCI
driver currently uses ``passthrough_whitelist`` under ``[pci]`` and
will move to ``device_spec`` in a future release.

The ``device_spec`` supports all parameters available in the existing
PCI driver including ``vendor_id``, ``product_id``, and ``address``
with glob and regex matching. Two optional cleanup policy keys are
also accepted: ``clear_action`` and ``clear_strategy`` (see the
Discovery and Cleanup Resolution section below for the full policy
matrix). Discovery reads ``/sys/bus/pci/devices/`` and filters
devices based on ``device_spec``::

    [nvme]
    device_spec = {"vendor_id": "8086", "product_id": "0001",
                   "clear_action": "auto", "clear_strategy": "auto"}
    # OR
    device_spec = {"address": "*:0a:00.*"}
    # OR combined
    device_spec = {"vendor_id": "8086", "address": "0000:0a:00.*"}

``clear_action`` and ``clear_strategy`` default to ``auto`` when omitted.

Discovery and Cleanup Resolution
--------------------------------

During discovery, the NVMe driver resolves the NVMe device path from the
PCI address via sysfs and runs ``nvme id-ctrl`` under privsep for each
discovered device. The output is parsed to determine hardware
capabilities. The following fields are checked:

* ``sanicap`` bit 0 — Crypto Erase Sanitize (CES)
* ``sanicap`` bit 1 — Block Erase Sanitize (BES)
* ``ONCS`` bit 3 — Write Zeroes (WZS)
* ``OACS`` bit 3 — Namespace Management (used internally, not a trait)

Hardware capabilities are reported as standard traits to Placement,
following the pattern established by ``os_traits/hw/nic/__init__.py``.
The traits are defined in a new ``os_traits/hw/nvme/__init__.py``
module::

    TRAITS = [
        'CES',   # Crypto Erase Sanitize Supported (sanicap bit 0)
        'BES',   # Block Erase Sanitize Supported (sanicap bit 1)
        'WZS',   # Write Zeroes Supported (ONCS bit 3)
    ]

All hardware capability traits are reported to Placement regardless of
the operator-selected cleanup policy. Traits describe what the device
supports, not what the configured policy will use.

Cleanup policy is configured per device via the ``clear_action`` and
``clear_strategy`` keys in ``device_spec``. ``clear_action`` selects
the operation type (sanitize, zero); ``clear_strategy`` constrains the
erase mechanism (crypto, block).

``clear_action`` selects the cleanup operation:

* ``auto`` (default): select an operation during discovery based on
  ``clear_strategy`` and the device capabilities.
* ``sanitize``: require NVMe sanitize.
* ``zero``: require block-device clearing by writing zeroes to the
  full device.

``clear_strategy`` selects the erase approach within the action:

* ``auto`` (default): choose the strongest supported approach for the
  selected action.
* ``crypto``: require cryptographic erase.
* ``block``: require a non-cryptographic block/media erase or block
  clear approach.

The policy matrix is::

    +--------------+----------------+---------------------------------------+
    | clear_action | clear_strategy | selected cleanup operation             |
    +==============+================+=======================================+
    | auto         | auto           | CES sanitize, else BES sanitize, else |
    |              |                | write zeroes, else shred               |
    +--------------+----------------+---------------------------------------+
    | auto         | crypto         | CES sanitize only                     |
    +--------------+----------------+---------------------------------------+
    | auto         | block          | BES sanitize, else write zeroes, else |
    |              |                | shred                                 |
    +--------------+----------------+---------------------------------------+
    | sanitize     | auto           | CES sanitize, else BES sanitize       |
    +--------------+----------------+---------------------------------------+
    | sanitize     | crypto         | CES sanitize only                     |
    +--------------+----------------+---------------------------------------+
    | sanitize     | block          | BES sanitize only                     |
    +--------------+----------------+---------------------------------------+
    | zero         | auto/block     | write zeroes, else shred              |
    +--------------+----------------+---------------------------------------+
    | zero         | crypto         | invalid configuration                 |
    +--------------+----------------+---------------------------------------+

The resolution chain within the ``zero`` method prefers ``nvme
write-zeroes`` (controller-side, if ``WZS`` is supported) over
host-side zeroing via ``shred``, following Nova's ``volume_clear``
pattern for LVM volumes.

The resolved action is stored in ``std_board_info`` alongside existing
device metadata. The selected cleanup action is fixed for the device
until the next discovery cycle or agent restart. There is no runtime
fallback. The cleanup action portion of ``std_board_info`` must not be
overwritten by ``discover()`` while ``device_state`` is not
``available``; in ``error`` state the action may be updated to allow
operators to fall back to a different strategy. If the locked-in
cleanup action fails, times out, or cannot be completed, the device
moves to ``error`` state.

If the configured policy cannot be satisfied by the discovered device
capabilities, the device is not reported as available. The driver logs
an error and excludes the device from the available pool until the
configuration is corrected or the device is replaced.

Driver Deprecation
------------------

The existing SSD and Inspur drivers are deprecated in this release in
favour of the generic NVMe driver. The deprecated drivers and the new
generic NVMe driver can coexist on the same system as long as they
manage independent devices. For any given device, managing it with more
than one driver simultaneously is considered invalid. The Cyborg agent
will fail to start up if more than one driver returns the same device,
raising a new ``InvalidConfiguration`` exception. See the Upgrade
impact section for deprecation timeline and removal criteria.

NVMe Device Type
----------------

A new ``DEVICE_NVME = 'NVME'`` constant will be added to
``cyborg/common/constants.py`` alongside the existing device types
(``GPU``, ``FPGA``, ``AICHIP``, ``QAT``, ``NIC``, ``SSD``). The NVMe
driver's ``discover()`` method sets ``type=NVME`` on each
``DriverDevice`` object. Adding ``NVME`` to the type field does not
require an API microversion because the ``type`` field is
``wtypes.text`` (free-form string) at the API layer.

Placement Integration
---------------------

Device type and vendor/product identity are encoded in the resource
class name as ``CUSTOM_NVME_<VENDOR_ID>_<PRODUCT_ID>``, following
the pattern used by Nova's PCI placement translator which generates
``CUSTOM_PCI_<VENDOR_ID>_<PRODUCT_ID>``. The existing Cyborg PCI
driver incorrectly encodes vendor and product identity as traits;
instead the NVMe driver follows Nova's PCI placement translator by
encoding them in the resource class. Traits are reserved for NVMe
capabilities as described in the NVMe Device Capabilities section
above.

The NVMe driver's ``discover()`` method builds ``DriverDevice`` objects
with ``type=NVME`` and reports NVMe capability traits alongside
``OWNER_CYBORG``. The ``std_board_info`` field contains the
``product_id`` and ``pci_address`` of the device's Physical Function
(PF). Resource provider and deployable names use the format
``<hostname>_<pci_address>``.

For example, an Intel NVMe device on compute-1 at PCI address
``0000:01:00.0`` that supports crypto erase and block erase would be
registered in Placement as::

    Resource provider: compute-1_0000:01:00.0
    Resource class:    CUSTOM_NVME_8086_0001
    Inventory:         total=1
    Traits:            OWNER_CYBORG, HW_NVME_CES, HW_NVME_BES, HW_NVME_WZS

During discovery, if the NVMe driver finds that a resource provider
already exists for a PCI address but does not have the ``OWNER_CYBORG``
trait, the driver logs an error and skips that device. This prevents
conflicts when the same device is also managed by Nova.
Nova's PCI placement translator (``nova/compute/pci_placement_translator.py``)
uses the same ``<hostname>_<pci_address>`` naming scheme,
so a Nova-managed device at the same PCI address would have an RP with
``CUSTOM_PCI_<VENDOR_ID>_<PRODUCT_ID>`` but without ``OWNER_CYBORG``.

Cleanup Policy and Execution
----------------------------

The cleanup contract provided by this driver is that a Cyborg-managed
NVMe device is cleaned before it is returned to the available pool. The
minimum guarantee is clearing the host-addressable block device. This
prevents reuse of stale tenant data by the next consumer, but it is not
a claim that previous data is unrecoverable with advanced forensic
techniques. Tenants that require confidentiality of data at rest should
use guest-managed encryption such as LUKS.

When Nova deletes an instance, it calls
``DELETE /v2/accelerator_requests``. For devices that support
cleaning (``device.supports_cleaning``), the Cyborg conductor
dispatches a ``cleanup_device(context, device)`` RPC cast to the
agent on the device's host (see RPC API impact section). For
non-NVMe devices, the conductor transitions the device directly to
``available`` without an RPC. During cleanup of NVMe devices, all
device state management is handled by the agent, not the conductor.
On the agent side, the agent sets
``device_state`` to ``pending_cleaning``, then to ``cleaning``, and
dispatches cleanup to a futurist thread pool with a configurable
timeout (see Timeout and Crash Recovery section). All nvme-cli
commands run under privsep with per-device locking as described in the
Security impact section.

The agent executes exactly the cleanup action that was locked in during
discovery (see Discovery and Cleanup Resolution section above). There
is no runtime fallback. If the action fails, the device moves to
``error`` state. Operators can re-trigger cleanup via
``POST /v2/devices/{uuid}/clean``; the retry uses the same locked-in
action.

NVMe cleanup requires the controller character device (``/dev/nvmeN``)
to be accessible on the host after the guest is destroyed. Libvirt's
managed mode handles this: it rebinds the device to the host NVMe
kernel driver on guest teardown. Driver binding is entirely managed
by libvirt; Cyborg assumes the device is already bound to the host
driver when cleanup begins.

**Cleanup execution order**

For sanitize-based cleanup:

#. Resolve the NVMe controller device from the PCI address
#. Check sanitize status; if a previous sanitize is still running,
   poll it instead of starting a new one
#. Run the selected sanitize action (sanitize erases the entire
   controller regardless of how many namespaces exist)
#. Poll until completion, failure, or timeout
#. Move the device to ``available`` on success, or ``error`` on
   failure

For zero-based cleanup:

#. Resolve the NVMe controller device from the PCI address
#. If more than one namespace exists, delete all namespaces and
   create a single namespace covering the full device (so the
   entire storage is exposed for zeroing)
#. Write zeroes to the full device (``nvme write-zeroes`` if
   supported, otherwise ``shred``)
#. Move the device to ``available`` on success, or ``error`` on
   failure

After cleanup, the controller is available with no specific namespace
layout guaranteed. The next tenant decides how to configure it.

Device State Machine
--------------------

The device lifecycle is tracked using a new ``device_state`` field
added to the ``Device`` versioned object (``cyborg/objects/device.py``)
and the ``devices`` database table. ``device_state`` is added to all
devices regardless of type; every device follows the same state
transitions with no type-specific checks. The bind guard and
Placement ``reserved`` updates also apply uniformly. This is
separate from the existing ``status`` field which remains
exclusively for the enabled/disabled scheduling control.

The state transitions are (``reserved=total`` for all states
except ``available`` which has ``reserved=0``)::

    +-----------+       bind       +-----------+
    | available |----------------->| allocated |
    +-----------+                  +-----------+
          ^                              |
          |                              | unbind /
          | success                      | init_host: missed cleanup
          |                              v
    +-----------+   agent starts   +------------------+
    |  cleaning |<-----------------| pending_cleaning |<---------+
    +-----------+   / no-op        +------------------+          |
          |                              |                       |
          | failure / timeout /          | init_host:            |
          | init_host: crash rec.        | crash rec.            | operator
          v                              v                       | POST /clean
    +------------------------------------------------------------+
    |                           error                            |
    +------------------------------------------------------------+

A device starts in ``available`` state with ``reserved=0``, meaning it
is clean and ready for allocation. When bound to an instance,
``reserved`` is set to ``total`` and the device moves to ``allocated``.

On instance deletion, the conductor checks
``device.supports_cleaning``. For non-NVMe devices, the conductor
sets ``device_state`` to ``available`` directly without dispatching
an RPC. For NVMe devices, the conductor dispatches a cleanup RPC to
the agent (see RPC API impact section). The agent sets the device to
``pending_cleaning``, then ``cleaning`` while running the sanitize
operation. All device state transitions during cleanup are managed by
the agent, not the conductor.

On success, ``reserved`` is set back to ``0`` and the device returns
to ``available``. On failure or timeout the device moves to ``error``
with ``reserved=total`` so it cannot be reallocated.

An operator can re-trigger cleanup from ``error`` using
``POST /v2/devices/{uuid}/clean``, which transitions the device back
to ``pending_cleaning``.

On agent restart, ``init_host()`` reconciles device states. A device
in ``allocated`` with no active ARQ indicates a cleanup RPC was missed
while the agent was down; ``init_host()`` moves it to
``pending_cleaning`` and triggers cleanup automatically. A device
found in ``pending_cleaning`` or ``cleaning`` indicates the agent
crashed mid-operation; ``init_host()`` moves it to ``error`` so the
operator can investigate before re-triggering via POST /clean. See
`Timeout and Crash Recovery`_ for details.

The invariant ``reserved=total`` in Placement is always consistent with
``device_state`` not being ``available``. A mismatch indicates a defect
in the bind or cleanup path; the agent logs a warning during
``init_host()`` so operators can investigate and manually correct the
state.

Device Bind
^^^^^^^^^^^

When the Cyborg conductor binds a device to an instance, it sets
``reserved=total`` in Placement by calling
``update_rp_inventory_reserved()``
(``cyborg/common/placement_client.py``) on the device's resource
provider. This prevents the scheduler from allocating the same device
to another instance while it is in use or awaiting cleanup. The bind
path in ``ExtARQ.bind()`` (``cyborg/objects/ext_arq.py``) also checks
that ``device_state`` is ``available`` before proceeding; if the
device is in any other state the bind is rejected. Both behaviors are
new additions to the shared bind path and apply to all device types,
not only NVMe. The existing ``update_rp_inventory_reserved()`` method
(currently used for device enable/disable) is reused for bind.

Timeout and Crash Recovery
--------------------------

NVMe cleanup operations can take minutes depending on drive size and
sanitize method. The agent enforces the timeout by calling
``future.result(timeout=cleanup_timeout)`` on the futurist thread
pool future. The timeout is configurable::

    [nvme]
    cleanup_timeout = 900      # seconds (default: 15 minutes)

If cleanup does not complete within the timeout, the agent sets
``device_state`` to ``error`` and logs a warning with the device UUID
for operator intervention.

Crash recovery is handled by the agent during ``init_host()``
(``cyborg/agent/manager.py``), which runs after the RPC server starts
but before the first ``discover()`` call. The agent queries the
database for devices in ``cleaning`` or ``pending_cleaning`` state on
its host. For each such device, the agent moves it to ``error`` state
so the operator can investigate and re-trigger cleanup via
``POST /v2/devices/{uuid}/clean``. The agent also checks for devices
in ``allocated`` state that have no active ARQ allocation, which
indicates a missed cleanup RPC. For each such device, the agent
transitions it to ``pending_cleaning`` and triggers cleanup. The
Placement ``reserved=total`` set at bind time prevents the device
from being reallocated between the missed RPC and the next agent
restart.

Adding Vendor-specific Drivers
------------------------------

Vendor-specific code may extend the generic driver to use dedicated
tooling where available. A vendor driver inherits from ``NVMeDriver``
and overrides the ``cleanup(device)`` method to add custom driver
logic.

Alternatives
------------

Relying on Nova PCI passthrough alone was rejected. Nova has a
``one_time_use`` flag (``nova/compute/pci_placement_translator.py``)
that sets ``reserved=total`` when a device is allocated, but Nova
never unreserves the device — it expects an external entity to do so
after cleanup. Nova PCI passthrough has no cleanup mechanism for
stateful devices like NVMe SSDs.

Using the Cyborg generic PCI driver with external cleanup automation
was rejected. The PCI driver (``cyborg/accelerator/drivers/pci/``)
only implements ``discover()`` with no lifecycle management. There is
no ``device_state`` field or ``cleanup()`` method. External tooling
has no way to coordinate with Cyborg's Placement reservations.


Data model impact
-----------------

**Schema migration**

An Alembic migration adds a ``device_state`` column to the
``devices`` table as a nullable Enum over the values ``available``,
``allocated``, ``pending_cleaning``, ``cleaning``, and ``error``.
All existing rows start as ``NULL``. The column cannot default to
``available`` because devices that are currently bound to instances
would be incorrectly marked as available.

The ``NVME`` value is added to the ``type`` column Enum which
currently contains ``GPU``, ``FPGA``, ``AICHIP``, ``QAT``, ``NIC``,
and ``SSD``.

**Online data migration**

An online data migration backfills the correct ``device_state`` for
existing devices, following the same pattern as
``heal_arq_project_ids`` (``cyborg/common/data_migrations.py``).
For each device with a ``NULL`` ``device_state``, the migration
checks whether the device has bound ARQs: devices with bound ARQs
are set to ``allocated``; devices without are set to ``available``.

The migration is callable from three entry points:

* ``cyborg-manage db online_data_migrations``
  (``cyborg/cmd/dbsync.py``)
* Conductor startup via ``init_host()``
  (``cyborg/conductor/manager.py``)
* Agent restart via ``init_host()`` for state reconciliation

A ``cyborg-status`` upgrade check (``cyborg/cmd/status.py``) is
added to verify that all ``device_state`` rows have been backfilled
before proceeding with further upgrades. In a future release
(2027.2), a contract migration will remove the nullability from the
column.

**Relationship between device_state and status**

``device_state`` and ``status`` are orthogonal. A device with
``status=maintaining`` can be in any ``device_state`` —
``maintaining`` prevents new scheduling, ``device_state`` tracks
lifecycle.

**Versioned object changes**

The ``Device`` oslo.versionedobjects definition
(``cyborg/objects/device.py``, currently version ``1.2``) is bumped
to ``1.3`` to include the ``device_state`` field.

An ``obj_make_compatible()`` method is added to pop ``device_state``
when downleveling to version ``1.2``. This follows the pattern used
in ``ExtARQ`` and ``DeviceProfile`` objects and Nova's
``ImageMeta.obj_make_compatible()`` (``nova/objects/image_meta.py``),
ensuring upgraded conductors can communicate with N-1 agents.

A ``supports_cleaning`` property is added, returning
``self.type == 'NVME'``. The conductor uses this to decide whether
to dispatch the ``cleanup_device`` RPC or transition the device
directly to ``available`` on unbind. A driver-independent capability
model is out of scope (see Scope section). The existing ``status``
field remains unchanged.


REST API impact
---------------

A new microversion is required. The exact version number depends on
merge ordering with other in-flight specs; this spec claims the next
available microversion after its dependencies land. The microversion
is driven by the new ``device_state`` field in device responses and the
new ``POST /v2/devices/{uuid}/clean`` endpoint. Adding ``NVME`` to the
``type`` field does not require a microversion because the type field
is ``wtypes.text`` (free-form string) at the API layer.

For API microversions lower than the new version, ``GET /v2/devices``
and ``GET /v2/devices/{uuid}`` responses remain as today. For the new
microversion and later, device responses include the ``device_state``
field.

**Example GET /v2/devices/{uuid} response (new microversion):**

Device in available state after successful cleanup::

  {
    "uuid": "7c8a5f3b-2d4e-4a9c-b1e7-9f8d3c2a1b0e",
    "type": "NVME",
    "vendor": "8086",
    "model": "0001",
    "std_board_info": "{\"product_id\": \"0001\",
        \"pci_address\": \"0000:01:00.0\"}",
    "vendor_board_info": null,
    "hostname": "compute-1",
    "status": "enabled",
    "device_state": "available",
    "created_at": "2026-05-15T10:23:45Z",
    "updated_at": "2026-05-15T10:25:30Z"
  }

Device in error state after failed cleanup::

  {
    "uuid": "3f2a8b9c-1e5d-4c7b-a3f6-8d2e9c1b4a7f",
    "type": "NVME",
    "vendor": "144d",
    "model": "a808",
    "std_board_info": "{\"product_id\": \"a808\",
        \"pci_address\": \"0000:02:00.0\"}",
    "vendor_board_info": null,
    "hostname": "compute-2",
    "status": "enabled",
    "device_state": "error",
    "created_at": "2026-05-15T14:00:00Z",
    "updated_at": "2026-05-15T16:30:15Z"
  }

**POST /v2/devices/{uuid}/clean (new endpoint, new microversion):**

This is an admin-only endpoint that triggers device cleanup. It
dispatches the ``cleanup_device`` RPC to the agent, which transitions
the device from ``error`` to ``pending_cleaning`` and runs cleanup.
This endpoint is the supported mechanism for operators to re-trigger
cleanup after a failure.

Normal response code: ``202 Accepted``

Error response codes:

* ``400 Bad Request`` — device does not support cleaning
* ``400 Bad Request`` — agent does not support the cleanup RPC
  (pre-2026.2 agent)
* ``403 Forbidden`` — caller lacks the ``cyborg:device:clean`` policy
* ``404 Not Found`` — device UUID does not exist
* ``409 Conflict`` — device is in ``available`` state (already clean)
* ``409 Conflict`` — device is in ``allocated`` state (bound to an
  instance)
* ``409 Conflict`` — device is already in ``cleaning`` or
  ``pending_cleaning`` state

Example::

  POST /v2/devices/3f2a8b9c-1e5d-4c7b-a3f6-8d2e9c1b4a7f/clean

  HTTP/1.1 202 Accepted

Policy is unchanged for existing endpoints. The new ``POST
/v2/devices/{uuid}/clean`` endpoint is governed by the
``cyborg:device:clean`` policy rule, defaulting to ``role:admin``.

RPC API impact
--------------

A new method is added to the agent RPC API (version bump ``1.0`` →
``1.1``)::

    def cleanup_device(context, device):
        """Trigger async device cleanup.

        :param context: request context
        :param device: Device object to clean
        """

The method is intentionally generic — it takes a ``Device`` object
rather than NVMe-specific parameters, so any driver that implements
``cleanup()`` can use the same RPC infrastructure.

The conductor guards the RPC dispatch with two checks, following
the pattern in Nova's ``scheduler/rpcapi.py``::

    if not device.supports_cleaning:
        # non-NVMe -> available
        device.device_state = 'available'
        device.save()
        return
    if not self.client.can_send_version(version):
        # NVMe on old agent -> error
        device.device_state = 'error'
        device.save()
        return
    cctxt.cast(context, 'cleanup_device', device=device)

Non-NVMe devices transition directly to ``available`` without an
RPC. If the device is NVMe but the target agent does not support
version ``1.1`` (a 2026.1 agent), the conductor sets
``device_state`` to ``error`` because an NVMe device on an agent
that cannot clean requires operator attention.

When the agent does support the RPC, the call returns immediately
so Nova's instance deletion completes without waiting for cleanup.
The agent manages all ``device_state`` transitions during cleanup
directly via the database: ``pending_cleaning``, ``cleaning``,
``available``, and ``error``.


Security impact
---------------

A device only returns to ``available`` (``device_state``) after the
erase operation is confirmed complete. Failed or timed-out cleanups
leave the device in ``error`` state so it is never reallocated with
stale data.

Elevated privilege is limited to destructive NVMe operations only.
NVMe-cli runs under privsep (``sys_admin_pctxt`` in
``cyborg/privsep/__init__.py``, which includes ``CAP_SYS_ADMIN``).
``CAP_SYS_ADMIN`` is required because the NVMe controller character
device (``/dev/nvmeN``) is root-only (mode 0600) and the Linux kernel
NVMe driver checks ``capable(CAP_SYS_ADMIN)`` for admin passthrough
ioctls (``NVME_IOCTL_ADMIN_CMD``). There is no lesser capability set
for ``nvme sanitize``, ``nvme write-zeroes``, ``nvme ns-rescan``,
``nvme create-ns``, ``nvme delete-ns``, or ``nvme id-ctrl``.
A more minimal privsep context is out of scope (see Scope section).

Verification of successful erasure is based on completed
administrative commands and their reported status, not on assuming
detach alone erases media. Cyborg trusts nvme-cli's output for erasure
confirmation; physical-level verification is the responsibility of
nvme-cli and device firmware.

Long-running subprocesses are bounded by the ``cleanup_timeout``
configuration. Concurrent cleanups per host are bounded by the futurist
thread pool size, and per-device locking (``@utils.synchronized()``)
prevents concurrent cleanup attempts on the same device.


Notifications impact
--------------------

Notifications are out of scope for this spec. Cyborg's notification
infrastructure was never fully implemented and a separate spec will
cover notifications for all Cyborg operations.


Other end user impact
---------------------

End users request NVMe devices through Cyborg device profiles. The
cleanup process is transparent to end users and requires no action on
their part.

**OpenStack SDK**: The ``Device`` resource gains a ``device_state``
attribute for the new microversion. A ``clean()`` action method is
added to trigger re-cleanup on devices in ``error`` state.

**python-cyborgclient**: ``device list`` and ``device show`` output
includes ``device_state`` for the new microversion. A ``device clean``
CLI command wraps the ``POST /v2/devices/{uuid}/clean`` endpoint.


Performance Impact
------------------

Cleanup runs asynchronously after instance deletion. Devices stay
reserved in Placement for the full cleanup window, which can be up to
15 minutes by default on large drives. Operators should monitor for
accumulation of devices in ``cleaning`` or ``error`` state.

The ``zero`` cleanup method (especially the ``shred`` fallback) on
large drives can take significantly longer than sanitize. Operators
using ``clear_action=zero`` on multi-terabyte drives should increase
``cleanup_timeout`` accordingly.


Other deployer impact
---------------------

Operators must install ``nvme-cli`` on compute nodes and set
``[agent] enabled_drivers`` and ``[nvme] device_spec`` in
``cyborg.conf``. The ``device_spec`` JSON accepts optional
``clear_action`` (auto|sanitize|zero) and ``clear_strategy``
(auto|crypto|block) keys to control cleanup policy; both default to
``auto``. Cleanup timeouts should be tuned for the largest drives in
the fleet.


Developer impact
----------------

The base ``Driver`` class gains two new methods with no-op default
implementations: ``cleanup(device)`` and ``init_host()``. Driver
authors implementing async cleanup for new device types should override
``cleanup()`` and use the same ``cleanup_device`` RPC path. No changes
are required for existing drivers that do not implement cleanup.
The conductor checks ``device.supports_cleaning`` before dispatching
the ``cleanup_device`` RPC. For device types that do not support
cleanup (currently all non-NVMe types), the conductor transitions
the device directly to ``available`` without an RPC. If the agent is
unavailable during unbind of an NVMe device, the device remains in
``allocated`` until the next agent restart, when ``init_host()``
reconciles the state.


Upgrade impact
--------------

Enabling the generic NVMe driver is a post-upgrade configuration
activity, not an in-place migration. The required upgrade ordering is:
run ``cyborg-manage db sync`` to apply the schema migration, then
upgrade the conductor and API services, and finally upgrade the agents.

Existing instances cannot directly use Cyborg-managed NVMe flavours.
Operators must recreate VMs with a new flavour that includes the
appropriate device profile.

The existing Inspur NVMe driver and SSD driver are deprecated in this
release in favour of the generic NVMe driver. No new feature
development will be done on the deprecated drivers outside of bug
fixes. The drivers will not be removed until Nova supports resizing
instances with Cyborg-managed devices, ensuring operators have a
migration path from vendor-specific to generic driver. The earliest
target release for removal is 2027.2; the drivers may remain deprecated
beyond that release if the resize prerequisite is not yet satisfied.
Driver documentation will include examples of how to manage Inspur
devices with the new generic driver.

This release introduces N-1 agent compatibility. A 2026.2 conductor
and API can operate with 2026.1 agents. The conductor checks
``can_send_version('1.1')`` before dispatching ``cleanup_device``
(see RPC API impact section). For non-NVMe devices, the conductor
sets ``device_state`` to ``available`` directly without an RPC
regardless of agent version. For NVMe devices where
``can_send_version`` fails, the conductor sets ``device_state`` to
``error``. Calling ``POST /v2/devices/{uuid}/clean`` on a device
whose agent does not support the cleanup RPC returns
``400 Bad Request``. After upgrading the agent, the operator can
retry the request.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  chandankumar

Other contributors:
  None

Work Items
----------

**Phase 1 — Foundation (independent patches):**

* Alembic migration for nullable ``device_state`` column and ``NVME``
  type. Bump ``Device`` object version to ``1.3`` with
  ``obj_make_compatible()`` and ``supports_cleaning`` property.
* Online data migration for ``device_state`` backfill
  (``cyborg/common/data_migrations.py``, ``cyborg/cmd/dbsync.py``,
  ``cyborg/conductor/manager.py``).
* ``cyborg-status`` upgrade check for ``device_state`` backfill
  (``cyborg/cmd/status.py``).
* Add no-op ``cleanup()`` and ``init_host()`` to base ``Driver`` class.
* Create basic ``NVMeDriver`` skeleton as a standalone driver.
* Submit os-traits companion patch for NVMe capability traits
  (``CES``, ``BES``, ``WZS``).

**Phase 2 — Driver functionality:**

* Implement ``init_host()`` validation for ``nvme-cli``.
* Implement device discovery with ``device_spec`` matching, capability
  detection (``sanicap``, ``ONCS`` bit 3 for WZS, ``OACS`` bit 3 for
  namespace management), and policy resolution from ``clear_action`` /
  ``clear_strategy`` to a locked-in cleanup operation.
* Report all discovered hardware traits to Placement.

**Phase 3 — Cleanup:**

* Add generic ``cleanup_device`` RPC with ``cctxt.cast()``.
* Implement ``NVMeDriver.cleanup()`` with futurist thread pool, state
  transitions, and Placement reserved handling on bind and unbind.
* For zero path: if more than one namespace exists, delete all and
  create a single namespace (conditional on OACS bit 3).
* Implement zero path: prefer ``nvme write-zeroes`` (ONCS bit 3), fall
  back to ``shred`` following Nova's ``volume_clear`` pattern.
* Wire ``ExtARQ._deallocate_attach_handle()`` to dispatch cleanup.

**Phase 4 — Recovery:**

* Implement crash recovery in ``init_host()`` for stale cleanup states.
* Add bind-path guard rejecting bind if ``device_state != available``.

**Phase 5 — REST API and microversion (last):**

* Add ``device_state`` to device responses for the new microversion.
* Add ``POST /v2/devices/{uuid}/clean`` admin-only endpoint.
* OpenStack SDK: add ``device_state`` attribute and ``clean()`` action.
* python-cyborgclient: add ``device_state`` to ``device list``/``show``
  output and ``device clean`` CLI command.
* API sample tests, documentation and release notes.


Dependencies
============

``nvme-cli`` must be installed on compute nodes and is added to
``bindep.txt`` as a runtime binary dependency. An os-traits companion
patch adding ``os_traits/hw/nvme/__init__.py`` with traits ``CES``,
``BES``, and ``WZS`` must land before or alongside the Cyborg
implementation. ``futurist`` is added as a new Python dependency.


Testing
=======

First-party CI includes API unit tests covering all cleanup methods,
failure scenarios (timeout, unsupported capability, Placement call
failure, ``nvme-cli`` not found, agent crash mid-cleanup), and
integration tests to validate Cyborg and Placement integration through
the full lifecycle.

Additional test scenarios for the cleanup policy:

* Policy matrix resolution: all valid ``clear_action`` × ``clear_strategy``
  combinations produce the correct cleanup operation.
* Invalid configuration rejection: ``zero`` + ``crypto`` is rejected
  at discovery time with a logged error.
* Device exclusion: policy requiring a capability the hardware lacks
  results in the device being excluded from the pool.
* Namespace consolidation (zero path only): conditional on OACS bit 3.
  Verify namespaces are consolidated before zero-write.
* Retry via ``POST /v2/devices/{uuid}/clean`` uses the same locked-in
  action from ``std_board_info``.

Whitebox Tempest plugin tests cover end-to-end validation on real or
emulated hardware. A manual run of the whitebox Tempest plugin
covering all cases must be submitted before this feature is merged.


Documentation Impact
====================

Cyborg administrator guide updates for NVMe driver configuration,
cleanup operations, and upgrade ordering. API reference updates for
the new microversion and endpoint.


References
==========

* Cyborg accelerator device API:
  https://docs.openstack.org/api-ref/accelerator/#devices

* Cyborg generic PCI driver configuration:
  https://docs.openstack.org/cyborg/latest/configuration/drivers.html#generic-pci-driver

* Compute API guide — Accelerator support:
  https://docs.openstack.org/api-guide/compute/accelerator-support.html

* NVM Express specifications: https://nvmexpress.org/specifications/

* nvme-cli: https://github.com/linux-nvme/nvme-cli

* nvme-cli manual pages:
  https://github.com/linux-nvme/nvme-cli/tree/master/Documentation
  (sanitize.txt, write-zeroes.txt)

* os-traits HW namespace:
  https://github.com/openstack/os-traits/tree/master/os_traits/hw


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2026.2
     - Introduced
