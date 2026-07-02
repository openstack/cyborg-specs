..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Generic Mediated Device (mdev) Driver Support
=============================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/generic-mdev-driver


Problem description
===================

Cyborg currently supports mediated devices (mdev) only through the
NVIDIA GPU driver, where mdev functionality is tightly coupled to
GPU-specific code. This spec proposes a standalone, generic mdev driver
that can discover and report any Linux kernel mdev-capable device,
along with a reusable ``MdevBusManager`` class that encapsulates mdev bus
discovery and eliminates code duplication between the new driver and
the existing GPU driver.

Use Cases
---------

As a Cyborg developer I would like to test mdev support easily and cheaply.
Adding a generic mdev driver would allow testing that functionality with fake
mdev devices via the pci-sim kernel module recently introduced.
Reusing this proven mdev discovery class in other drivers like the GPU driver
means increasing the testing coverage across the project, reducing the risk of
bugs and the cost of ongoing maintenance.

As an operator I would like to expose non-GPU mdev-capable devices
such as Intel QAT, or vendor-provided vFPGA devices to
my tenants without waiting for a dedicated Cyborg driver for each
device type. A generic mdev driver lets me make any mdev-capable
hardware available through Cyborg immediately.


Proposed change
===============

This spec proposes a reusable ``MdevBusManager`` class and a new generic
mdev driver. The ``MdevBusManager`` class encapsulates generic mdev bus
discovery and type introspection by reading from the standard kernel
sysfs hierarchy under ``/sys/class/mdev_bus/``. For each parent PCI
device the class enumerates the supported mdev types, reading their
names, descriptions, available instance counts, and device API strings
from sysfs. It returns this information as plain dictionaries, without
constructing Cyborg driver objects. The proposed driver will not handle
creation nor deletion of mdevs, instead will delegate that to
Nova [1]_.

The proposed generic mdev driver holds an ``MdevBusManager`` instance as
a private attribute, set during ``__init__``. In its ``discover()``
method, the driver calls ``MdevBusManager.discover_parent_devices()`` to
enumerate mdev-capable PCI devices, then calls
``MdevBusManager.get_mdev_types()`` for each parent to retrieve the
supported mdev types.

Resource provider topology
--------------------------

Cyborg's current data model maps each ``DriverDeployable`` to
exactly one Placement resource provider with a single resource
class.  The ``num_accelerators`` field is a single integer and
the conductor extracts only the first ``rc`` attribute to build
inventory.  This means that, without changes to the conductor and
the driver object model, a single deployable cannot carry
inventory for multiple resource classes.

A physical parent device may support several mdev types, each of
which needs its own resource class.  There are three possible
approaches to represent this in Placement:

a) **One deployable per mdev type** -- each type becomes its own
   ``DriverDeployable`` with its own child resource provider
   under the compute node.  This matches the pattern used by
   every existing Cyborg driver and requires no conductor
   changes.  The trade-off is one child RP per type rather than
   one per physical device.

b) **One deployable with multiple resource classes** -- encode
   per-type inventory in the attribute list and modify the
   conductor to build a multi-key inventory dict.  This also
   requires making attach handle allocation type-aware (the
   current query grabs any free handle on the deployable).

c) **Nested resource providers** -- keep one deployable per type
   but introduce a device-level RP as an intermediate parent,
   giving a tree of the form
   ``compute-node > device_<pci_addr> > type_<mdev_type>``.
   Driver objects are unchanged but the conductor needs to create
   the intermediate RP.

This spec proposes **option (a)** for the initial implementation:
the driver produces one ``DriverDevice`` per physical parent PCI
device and one ``DriverDeployable`` per mdev type on that parent.
Each deployable maps to its own child resource provider parented
to the compute node.  Options (b) and (c) can be considered in a
future spec if a single-RP-per-device topology is desired.

The resulting Placement and driver object trees for a host with
two parent devices are shown below::

    Placement resource provider tree
    ================================

    compute-node-1 (root RP, created by Nova)
      |
      +-- mdev_0000:41:00.0_mtty-2
      |     CUSTOM_MDEV_MTTY_2: total=4
      |
      +-- mdev_0000:41:00.0_mtty-4
      |     CUSTOM_MDEV_MTTY_4: total=2
      |
      +-- mdev_0000:42:00.0_i915-GVTg_V5_4
            CUSTOM_MDEV_I915_GVTG_V5_4: total=8


    Driver object tree
    ==================

    DriverDevice (0000:41:00.0)
      |
      +-- DriverDeployable "mdev_0000:41:00.0_mtty-2"
      |     num_accelerators=4, rc=CUSTOM_MDEV_MTTY_2
      |     +-- DriverAttachHandle (MDEV, asked_type=mtty-2)  x4
      |
      +-- DriverDeployable "mdev_0000:41:00.0_mtty-4"
            num_accelerators=2, rc=CUSTOM_MDEV_MTTY_4
            +-- DriverAttachHandle (MDEV, asked_type=mtty-4)  x2

    DriverDevice (0000:42:00.0)
      |
      +-- DriverDeployable "mdev_0000:42:00.0_i915-GVTg_V5_4"
            num_accelerators=8, rc=CUSTOM_MDEV_I915_GVTG_V5_4
            +-- DriverAttachHandle (MDEV, asked_type=i915-GVTg_V5_4)  x8

The custom resource class name for each mdev type is derived from
the type name (the directory name under
``mdev_supported_types/``) using the format
``CUSTOM_MDEV_<TYPE_NAME>``.  The type name is normalized by
converting hyphens to underscores and uppercasing all alphabetic
characters to produce a valid Placement custom resource class
name. There exists functions to do this normalization in the os-traits and
os-resource-classes libraries that will be used for this purpose. As an
example, a type named ``mtty-2`` produces the resource class
``CUSTOM_MDEV_MTTY_2``.  This allows operators to write device profiles
that request specific mdev types.

Inventory counting
------------------

The ``available_instances`` value exposed by the kernel in sysfs
reflects only the currently *unused* capacity -- it decreases each
time an mdev instance is created, regardless of whether that
instance is assigned to a VM. The total inventory reported to
Placement must therefore account for both available and
already-created instances::

    total = available_instances + created_instances

where ``created_instances`` is the number of mdev instances of
that type that already exist under
``/sys/class/mdev_bus/<pci_addr>/mdev_supported_types/<type>/devices/``.
When a ``max_instances`` value is specified in the ``device_spec``
configuration for this parent/type combination, the reported total
is capped to that value.

The only trait that will be reported by the generic mdev driver is
the ``OWNER_CYBORG`` trait that is common to all devices managed
by Cyborg. Additional custom traits can be added through the
``device_spec`` configuration.

The driver also populates ``DriverControlPathID`` and
``DriverAttribute`` objects as needed. Configuration-based
filtering is passed through to the ``MdevBusManager`` methods via
their ``pci_filter`` and ``type_filter`` parameters.

Finally, the driver will create ``DriverAttachHandle`` objects of type MDEV,
which in its attach handle info will have a new ``asked_type`` field in
addition to the PCI address fields. The attach handle will look as
follows:

::

    {
        "attach_handle_type": "MDEV",
        "attach_handle_uuid": "91ac1606-427e-44bb-8233-f4ff4bf3d241",
        "attach_handle_info": {
            "asked_type": "mtty-2",
            "domain": "0000",
            "bus": "10",
            "device": "1",
            "function": "0"
        }
    }

This spec also proposes refactoring the existing GPU driver to hold an
``MdevBusManager`` instance and delegate mdev bus discovery to it, instead
of carrying its own sysfs parsing implementation. The GPU driver will
retain all NVIDIA-specific logic such as vendor name parsing,
lspci-based PCI discovery, trait generation, vGPU type configuration
mapping, and construction of its own Cyborg driver objects. Only the
raw mdev type introspection (reading ``available_instances``, ``name``,
``device_api`` from sysfs) is delegated to ``MdevBusManager``. As a result of
the refactor, the existing methods for creating and deleting mdevs will be
removed, since that functionality will be delegated completely to Nova.

MdevBusManager class design
----------------------------


The ``MdevBusManager`` class lives in
``cyborg.accelerator.handlers.mdev``.

The constructor accepts an optional ``sysfs_path`` parameter
(defaulting to ``/sys/class/mdev_bus/``) that specifies the root
directory for mdev bus discovery. This allows unit tests to point
the handler at a fake sysfs tree without patching.

The class exposes two methods:

* ``discover_parent_devices(pci_filter=None)`` -- Scans
  ``/sys/class/mdev_bus/`` and returns a list of PCI addresses of
  mdev-capable parent devices. The parameter ``pci_filter`` controls
  which devices should be discovered by Cyborg, the default is ``None`` which
  means no devices will be reported. The special value ``['*']`` means all
  found devices will be reported.

* ``get_mdev_types(pci_address, type_filter=None)`` -- Reads
  ``/sys/class/mdev_bus/{pci_address}/mdev_supported_types/`` and
  returns a list of dictionaries, one per supported mdev type.
  Each dictionary contains the keys ``type_name`` (directory name),
  ``name`` (contents of the ``name`` file), ``available_instances``
  (integer from the ``available_instances`` file), ``device_api``
  (contents of the ``device_api`` file), and ``description`` (contents
  of the ``description`` file, or empty string if absent). When
  ``type_filter`` is a non-empty list, only types whose
  ``type_name`` appears in the list are returned.

Drivers that need mdev support create an ``MdevBusManager`` instance in
their ``__init__`` method and store it as a private attribute. The
driver's ``discover()`` method calls ``MdevBusManager`` methods and then
builds its own Cyborg driver objects from the returned dictionaries.
This keeps ``MdevBusManager`` focused on sysfs reading and keeps
driver-specific object construction in the driver.

The following diagram illustrates the composition relationship::

                       +----------------------+
                       |  GenericDriver (ABC) |
                       +----------+-----------+
                                  |
               +------------------+------------------+
               | (Is-a)                              | (Is-a)
               v                                     v
    +--------------------+                 +--------------------+
    |    MdevDriver      |                 |     GPUDriver      |
    |    (generic)       |                 +---------+----------+
    +----------+---------+                           | (Is-a)
               |                                     v
               |                           +--------------------+
               |                           |  NVIDIAGPUDriver   |
               |                           +---------+----------+
               | (Has-a)                             | (Has-a)
               v                                     v
    +--------------------+                 +--------------------+
    |    MdevBusManager  |                 |    MdevBusManager  |
    |   (Instance #1)    |                 |   (Instance #2)    |
    +--------------------+                 +--------------------+


The entire flow of the discovery would look like::

    AgentManager             ResourceTracker     acc_driver          MdevBusManager
      |                       |                (Mdev / NVIDIA)   (Driver's Instance)
      |                       |                     |                     |
      | periodic_task         |                     |                     |
      |---\                   |                     |                     |
      |   | update_available_ |                     |                     |
      |   | resource()        |                     |                     |
      |<--/                   |                     |                     |
      |                       |                     |                     |
      | update_usage          |                     |                     |
      | (context)             |                     |                     |
      |---------------------->|                     |                     |
      |                       |                     |                     |
      |                       |================\    |                     |
      |                       | LOOP:          |    |                     |
      |                       | for acc_driver |    |                     |
      |                       | in acc_drivers |    |                     |
      |                       |================/    |                     |
      |                       |---|                 |                     |
      |                       |   |                 |                     |
      |                       |   | discover()      |                     |
      |                       |   |---------------->|                     |
      |                       |   |                 |                     |
      |                       |   |                 | discover() / scan() |
      |                       |   |                 |-------------------->|
      |                       |   |                 |                     | -- Scans sysfs/bus
      |                       |   |                 |                     |
      |                       |   |                 |   raw_device_data   |
      |                       |   |                 |<--------------------|
      |                       |   |                 |                     |
      |                       |   |discovered_list  |                     |
      |                       |   |<----------------|                     |
      |                       |   |                 |                     |
      |                       |   |---\             |                     |
      |                       |   |   | acc_list.   |                     |
      |                       |   |   | extend(...) |                     |
      |                       |   |<--/             |                     |
      |                       |===|================ \                     |
      |                       | END LOOP            |                     |
      |                       |==================== /                     |
      |                       |                     |                     |
      |                       |---\                 |                     |
      |                       |   | Audit & Claim   |                     |
      |                       |   | local resources |                     |
      |                       |<--/                 |                     |
      | return                |                     |                     |
      |<----------------------|                     |                     |


Configuration will be provided through a new ``[mdev]`` section in
``cyborg.conf``. The ``device_spec`` option accepts multiple entries of JSON
objects, each describing an mdev type to manage. Only mdevs contained in
this list will be managed by Cyborg. Every object supports the following
fields:

* ``address`` *(required)* -- PCI address of the parent device
  (e.g. ``0000:41:00.0``).
* ``mdev_type`` *(required)* -- mdev type name to create
  (e.g. ``nvidia-319``).
* ``max_instances`` *(optional)* -- maximum number of mdev
  instances the driver may report for this parent/type
  combination.  Defaults value is `None`, which means that the value advertised
  by the kernel in ``available_instances`` will be used (no limit is placed).
* ``resource_class`` *(optional)* -- custom Placement resource
  class.  Defaults to ``None``, which means that the standard
  ``CUSTOM_MDEV_<TYPE_NAME>`` will be used.
* ``traits`` *(optional)* -- list of additional Placement traits
  to set on the resource provider
  (e.g. ``["CUSTOM_GPU", "HW_GPU_API_VULKAN"]``).

Example ``cyborg.conf`` snippet::

    [mdev]
    device_spec = { "address": "0000:41:00.0", "mdev_type": "nvidia-319", "max_instances": 8, "resource_class": "VGPU", "traits": ["CUSTOM_NVIDIA_V100", "VULKAN"] },
    device_spec = { "address": "0000:42:00.0", "mdev_type": "i915-GVTg_V5_4" }

The new driver will be enabled via the existing
``[agent]enabled_drivers`` option.

Operators should not enable both the generic mdev driver and a
vendor-specific driver (e.g., the NVIDIA GPU driver) for the same
parent device. Doing so would not cause duplicate resource reporting to
Placement, but the device would be reported with the attributes discovered
by whichever driver is listed first in the ``enabled_drivers`` option.
This might be suboptimal as for example NVIDIA devices should be discovered
by the NVIDIA driver rather than the generic mdev one. The
``address`` field in each ``device_spec`` entry can be used to
avoid overlap. Nonetheless, an additional check will be
implemented in the conductor to catch duplicated devices and warn the
operator that the configuration should be revised.

Alternatives
------------

Keep mdev support only in vendor-specific drivers. Each new
mdev-capable device type (GVT-g, QAT, vFPGA) would get its own
driver with its own copy of the mdev sysfs logic. This is how Cyborg
currently handles mediated devices. It can lead to code duplication,
increase the maintenance burden, and raise the barrier for adding
new mdev device support.

Extend the existing GPU driver to handle non-GPU mdev devices.
The GPU driver could be broadened to discover all mdev types, not
just NVIDIA vGPUs. This would be difficult since the GPU driver carries
significant NVIDIA-specific assumptions (vendor maps, lspci-based
discovery, vGPU type naming conventions) that do not apply to
non-GPU devices. Overloading the GPU driver would conflate unrelated
concerns and make the code harder to maintain.

Data model impact
-----------------

The ``device_type`` column in the ``Device`` table is
currently stored as an opaque string, but the code uses a ``type``
attribute whose values are drawn from an informal enum in the
existing drivers (e.g. ``GPU``).  This spec requires adding
``MDEV`` to that set so the generic mdev driver can report the
device type accurately instead of labelling every mediated device
as ``GPU`` the way the NVIDIA driver does today.  For completeness
this spec also proposes adding ``PCI`` for the generic PCI driver.

No new database tables or schema migrations are required; the
``type`` field is an opaque string in the API, so the new value
does not require a microversion.

REST API impact
---------------

None. The generic mdev driver is a backend change. Discovered mdev
devices appear through the existing accelerator device and deployable
APIs without modification.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None. End users interact with mdev-backed accelerators through the
same device profile and accelerator request (ARQ) workflow used for
other device types. No changes to python-cyborgclient are required.

Performance impact
------------------

The mdev discovery process reads a small number of sysfs files per
mdev type per parent device. On a host with typical hardware (a few
mdev-capable devices, each with tens of supported types), discovery
completes in milliseconds. The discovery runs during the periodic
agent update cycle, the same cadence as existing drivers, and does
not introduce additional database or conductor calls.

Other deployer impact
---------------------

Deployers who wish to use the generic mdev driver must:

1. Ensure the host kernel supports mdev (``/sys/class/mdev_bus/``
   must be present) and that vendor-specific kernel modules are
   loaded (e.g., ``nvidia-vgpu-vfio``, ``kvmgt``, ``intel_qat``).

2. Add the ``[mdev]`` configuration section to ``cyborg.conf`` to
   control which parent devices or mdev types are
   discovered. The defaults (empty list) will filter out all devices
   and types.

3. Enable the driver in ``[agent]enabled_drivers``.

4. Restart the cyborg-agent service.

Deployers who are already using the NVIDIA GPU driver for vGPU
management do not need to change anything. The GPU driver continues
to function identically after the internal refactoring.

Developer impact
----------------

Driver developers who need mdev support in a new Cyborg driver can
create an instance of ``MdevBusManager`` from
``cyborg.accelerator.handlers.mdev`` and use its
``discover_parent_devices()`` and ``get_mdev_types()`` methods instead
of implementing their own sysfs parsing.

Upgrade impact
--------------

This functionality requires a version of Nova that implements [1]_. Upgrading
Cyborg without upgrading Nova will prevent the devices discovered by the new
generic mdev driver from being usable, since Nova will ignore the mdev arqs.

Since the companion Nova spec adds the support for mdev requests from cyborg,
there will be no migration path for cyborg owned devices. For nova-owned
devices, instances need to be recreated with a flavor with a cyborg-managed
mdev device profile(accel:device-profile=cyborg-vgpu-device-profile-name),
as detailed in the nova spec [1]_.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jgilaber

Other contributors:
  None

Work Items
----------

* Implement the ``MdevBusManager`` class in
  ``cyborg.accelerator.handlers.mdev`` with
  ``discover_parent_devices()`` and ``get_mdev_types()`` methods.

* Implement the generic mdev driver with device discovery and
  configuration-based filtering.

* Add new configuration options for the generic mdev driver and
  register the driver entry point.

* Refactor the existing GPU driver to hold an ``MdevBusManager`` instance
  and delegate mdev type introspection to it. Update the agent manager
  and existing GPU driver tests. In the NVIDIA driver unit tests, create
  a fake `/sys/class/mdev_bus/` directory to validate the discovery process
  after the refactor.

* Create documentation for the new driver.


Dependencies
============

End to end testing of the new driver will require changes in Nova to use the
devices discovered by the new Cyborg driver [1]_.


Testing
=======

Unit tests will be added for all new code. The MdevBusManager class will accept
a parameter for the path where to find the mdevs. This will allow testing the
discovery of different scenarios in temporary folders.

Tempest tests will be added leveraging the mdev sample drivers optionally
included in Nova's devstack plugin. They will allow creating mtty and mdpy
devices which should be discoverable by the new driver. The same test will also
exercise the ARQ bind path with Nova as well as the reporting to placement.


Documentation Impact
====================

Admin Guide: A new page (``doc/source/admin/mdev-driver.rst``)
will document the generic mdev driver, including prerequisites,
configuration options, example configurations for common device
types (NVIDIA vGPU, Intel GVT-g), and troubleshooting steps.


References
==========

.. [1] https://review.opendev.org/c/openstack/nova-specs/+/967515


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2026.2
     - Introduced
