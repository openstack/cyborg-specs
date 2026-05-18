..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Support VFIO variant driver managed mode in Cyborg
==================================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/support-vfio-variant-driver-managed-mode

This spec extends Cyborg's PCI attach handle information so that Nova can
compose Cyborg-managed PCI accelerators with the correct libvirt hostdev
``managed`` mode. This allows Cyborg-managed PCI devices using kernel VFIO
variant drivers to be assigned to guests while preserving the current managed
mode behavior for existing deployments.

Problem description
===================

Starting with kernel 5.16, the kernel has added a VFIO SR-IOV variant driver
interface. Some SR-IOV Virtual Functions (VFs), including accelerator VFs, are
intended to be bound to a device-specific VFIO variant driver instead of
the generic ``vfio-pci`` driver before QEMU uses them. For these devices,
libvirt must not rebind the VF from a host driver and re-attach it after the
guest is destroyed.
Instead, libvirt should create PCI hostdev XML with ``managed='no'``.

Nova's native PCI passthrough support already has a ``managed`` tag in the
``[pci] device_spec`` configuration for this. Cyborg-managed PCI accelerators
use a different path:
Cyborg returns bound Accelerator Requests (ARQs) with a PCI attach handle and
Nova's libvirt driver composes those attach handles into the guest XML. Today,
that Cyborg PCI path does not carry managed mode information and Nova treats
Cyborg PCI attach handles as managed devices. That is not sufficient for
Cyborg-managed PCI accelerators which require VFIO variant drivers.

Cyborg needs a backward-compatible way to communicate whether a PCI attach
handle should be libvirt managed. Nova then needs to consume that information
when building the libvirt domain XML.

Use Cases
---------

* As an operator, I want to use Cyborg to manage PCI accelerator VFs that are
  bound to ``vfio-pci`` or to device-specific VFIO variant drivers
  and set ``managed='no'`` correctly.

* As an operator, I want existing Cyborg PCI accelerators to keep using the
  current managed mode behavior unless I opt into a different value.

* As an operator, I want the managed mode behavior for Cyborg-managed PCI
  accelerators to be equivalent to the behavior available through Nova native
  PCI passthrough configuration.

* As a Nova developer, I want a generic Cyborg ARQ contract for PCI attach
  handles so that Nova's libvirt driver does not need Cyborg driver-specific
  special cases.

Proposed change
===============

This spec proposes a Cyborg change and a small companion Nova change.

Cyborg scope
------------

The Cyborg implementation in this cycle will target the generic PCI driver.
Other Cyborg drivers that create PCI attach handles, such as NIC, QAT, GPU, or
FPGA drivers, may add the same capability later without changing the Nova-side
contract described here.

The generic PCI driver will accept an optional ``managed`` tag in the existing
``[pci] passthrough_whitelist`` entries. The accepted values should be
compatible with the Nova device-specification values and normalized internally
to a boolean value for API output.

For example::

  [pci]
  passthrough_whitelist = {"vendor_id": "10de",
                           "product_id": "25b6",
                           "address": "0000:25:00.4",
                           "managed": false}

or equivalently::

  [pci]
  passthrough_whitelist = {"vendor_id": "10de",
                           "product_id": "25b6",
                           "address": "0000:25:00.4",
                           "managed": "no"}

.. note::

   oslo.utils' flexible string-to-boolean conversion will be used to convert
   this tag. The preferred and documented values are JSON boolean values,
   ``true`` and ``false``. String values such as ``yes`` or ``no`` may also
   be accepted for operator convenience.

``managed=True`` means that Nova will ask libvirt to rebind the PCI device
from the host driver to ``vfio-pci`` or ``pci-stub`` before attaching it to
the guest and to bind it back to the host driver after the guest is deleted.

``managed=False`` means that Nova will not ask libvirt to detach or re-attach
the device. In the ``managed=False`` case the operator is responsible for
configuring the host so that the VF is already bound to a driver usable by
QEMU and is not in use by the host.

If ``managed`` is not configured, the default is ``True``. This preserves the
current behavior and matches Nova's native PCI passthrough default.

ARQ attach handle information
-----------------------------

Cyborg will extend the existing PCI ``attach_handle_info`` mapping in the ARQ
response with an optional ``managed`` key. The key is scoped to PCI attach
handles and will be present only in a new API microversion.

With the new microversion, a bound PCI ARQ will include ``managed`` as a JSON
boolean in ``attach_handle_info``::

  {
    "uuid": "2f59e9d8-5a17-4607-83c5-e2ff83cce2c7",
    "state": "Bound",
    "device_profile_name": "vfio-variant-vf",
    "device_profile_group_id": 0,
    "hostname": "compute-0",
    "device_rp_uuid": "0d59e5a5-d25e-478a-9c06-8d6fb0a0717e",
    "instance_uuid": "a11a5487-696b-4d4e-a487-6b9ec9d7e458",
    "attach_handle_type": "PCI",
    "attach_handle_uuid": "97734f69-5cfa-4f4b-8609-b6d3ddadb0d7",
    "attach_handle_info": {
      "domain": "0000",
      "bus": "25",
      "device": "00",
      "function": "4",
      "managed": false
    }
  }

For the new microversion, Cyborg will include ``managed`` for all bound PCI
attach handles returned in ARQ responses. If no managed marker is stored for
an existing or newly discovered attach handle, Cyborg will return
``"managed": true``.

For older microversions, Cyborg will not include ``managed`` in the ARQ
response. Older clients therefore continue to see the same response shape as
today.

The initial implementation will store the marker in the existing attach handle
``attach_info`` JSON instead of adding a driver-specific database field. This
keeps the change small and avoids introducing attach-handle-type-specific
schema. A future generic metadata mechanism, such as a reusable JSON blob or a
separate mapping table for attach-handle metadata, can be introduced when more
metadata needs to be persisted. The API shape chosen here is intended to be
compatible with such a future internal refactor because the external contract
is an extension of the existing ``attach_handle_info`` mapping.

Nova libvirt driver companion change
------------------------------------

Nova's libvirt driver already creates PCI hostdev XML for Cyborg ARQs with
``attach_handle_type == 'PCI'``. The Nova change required by this spec is to
read the optional ``managed`` key from each PCI ARQ ``attach_handle_info`` and
use it when setting the libvirt hostdev managed mode.

Conceptually, the existing hard-coded behavior::

  self._set_managed_mode(dev, "true")

will become equivalent to::

  managed = arq['attach_handle_info'].get('managed', "true")
  self._set_managed_mode(dev, str(managed).lower())

or an equivalent boolean-aware implementation. If the key is absent, Nova must
default to managed mode enabled. This preserves compatibility with older
Cyborg services, older Cyborg microversions, and existing ARQs.

The Nova change should be generic for every Cyborg driver that returns a PCI
attach handle. It should not check for the generic Cyborg PCI driver
specifically. A Cyborg driver that returns ``attach_handle_type == 'PCI'`` and
includes ``attach_handle_info.managed`` will therefore get the same Nova
behavior.

With ``managed=False``, Nova should generate libvirt XML similar to::

  <hostdev mode='subsystem' type='pci' managed='no'>
    <driver name='vfio'/>
    <source>
      <address domain='0x0000' bus='0x25' slot='0x00' function='0x4'/>
    </source>
  </hostdev>

With ``managed=True`` or with the key absent, Nova should preserve the current
managed behavior and generate ``managed='yes'`` for KVM/QEMU.

This Nova work should be small enough to be done as a specless Nova blueprint
https://blueprints.launchpad.net/nova/+spec/support-vfio-variant-driver-managed-mode-via-cyborg

Future evolution
----------------

Live migration support for VFIO variant driver devices is intentionally out of
scope for this cycle. The design is intended to be evolvable for that work.
Possible future extensions include additional PCI attach-handle metadata such
as ``live_migratable`` or migration-specific device capabilities. Those
extensions should either use the same generic attach-handle metadata approach
or introduce a generic metadata storage model rather than adding
one-column-per-driver or one-column-per-attach-type schema.

Alternatives
------------

Add a top-level ARQ field for ``managed``
  This would make the API field more explicit, but ``managed`` is a property
  of the PCI attach handle rather than of the accelerator request itself. It
  also does not compose well with future attach-handle-specific metadata.

Add a dedicated database column to ``attach_handles``
  This is straightforward for this one field, but it creates a schema pattern
  where each attach-handle-type-specific option becomes a new database column.
  This spec prefers to defer a schema change until Cyborg needs a generic
  metadata model.

Add a generic attach-handle metadata table or JSON blob now
  This is likely the right long-term shape if Cyborg needs to carry several
  attach-handle-specific attributes. However, this feature only needs one
  optional field and can be implemented using the existing ``attach_info`` JSON
  while preserving an API shape that can survive a future internal refactor.

Require all drivers to implement the field in one cycle
  This would provide broader coverage, but it is not required for the initial
  VFIO variant driver use case. The Nova-side contract will be generic, and
  other Cyborg drivers can opt in later.

Data model impact
-----------------

No database schema change is proposed.

The generic PCI driver will include ``managed`` in the existing
``AttachHandle.attach_info`` JSON when the value is configured. Existing rows
which lack the key remain valid. For the new API microversion, response
serialization will default missing values to ``true`` for PCI attach handles.

The ``AttachHandle`` and ``DriverAttachHandle`` object versions may not need to
change if the implementation only treats ``attach_info`` as an opaque JSON
string. If implementation code adds helper properties or changes object fields,
the relevant object versions must be bumped according to normal Cyborg object
compatibility rules.

Changing ``managed`` for an existing attach handle changes how Nova should use
that handle for future guest XML generation. Operators should not change this
configuration for devices currently assigned to guests. The implementation
should document whether a service restart, rediscovery, or attach handle
recreation is required for changed configuration to be reflected in stored
``attach_info``.

REST API impact
---------------

A new Cyborg API microversion will be added. Assuming the current maximum
microversion is ``2.3``, this spec proposes ``2.4``.

The changed API response is the ARQ representation returned by:

* ``GET /accelerator/v2/accelerator_requests/{uuid}``
* ``GET /accelerator/v2/accelerator_requests?instance=<uuid>&``
  ``bind_state=resolved``
* ``POST /accelerator/v2/accelerator_requests`` when the returned ARQs are
  already bound by a later flow is not expected, but the ARQ schema is common.

No request body changes are proposed.

For API microversions lower than ``2.4``, the PCI ARQ response remains as it
is today::

  {
    "attach_handle_type": "PCI",
    "attach_handle_info": {
      "domain": "0000",
      "bus": "25",
      "device": "00",
      "function": "4"
    }
  }

For API microversion ``2.4`` and later, bound PCI ARQs include a boolean
``managed`` key in ``attach_handle_info``::

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

The value is ``true`` when the device is configured as managed or when the
underlying attach handle has no explicit managed marker. The value is
``false`` when the operator configured the matching PCI device with
``managed=false`` or an equivalent false value.

When microversion ``2.4`` or later is used, ``managed`` will always be
present for PCI attach handles. This includes PCI attach handles created by
drivers that do not yet support this setting natively; those drivers will
report the backward-compatible default value of ``true``.

Non-PCI attach handles are unchanged by this microversion.

Policy is unchanged.

Security impact
---------------

This change does not have a material security impact.

Incorrect managed mode configuration can destabilize or crash the host
if the host kernel driver has a bug. That is always a kernel bug and should
be reported if encountered to the kernel driver maintainer.
Only deployers should control this value via Cyborg service configuration
and driver discovery. Tenants cannot request or override the value through
the ARQ API.

Notifications impact
--------------------

None.

Other end user impact
---------------------

The OpenStack SDK accelerator resource model should be updated to understand
that, from microversion ``2.4``, ``attach_handle_info`` for PCI ARQs will
contain a ``managed`` boolean. The existing SDK model already exposes
``attach_handle_info`` as a body mapping, so the required SDK change is
expected to be small: add tests and, if needed, update the maximum supported
accelerator microversion and documentation for the ARQ resource.

``python-cyborgclient`` should update its default or current accelerator
microversion after Cyborg adds ``2.4`` support. Its ARQ list and show commands
already display ``attach_handle_info``, so no new command option is required.
Client tests and help/documentation should be updated to cover the new nested
``managed`` value when using microversion ``2.4``.

Performance Impact
------------------

None expected. The change adds a small amount of JSON parsing and serialization
for ARQ responses and PCI driver discovery. It does not add new periodic tasks,
remote calls, Placement operations, or scheduler filters.

Other deployer impact
---------------------

A deployer may configure the generic Cyborg PCI driver with an optional
``managed`` tag for matching PCI devices. The default is managed mode enabled,
so existing deployments do not need configuration changes.

For ``managed=False`` devices, the deployer is responsible for host setup. The
VF must be prepared in a way that QEMU can use it directly, typically by
binding it to ``vfio-pci`` or the relevant VFIO variant driver before it is
assigned to a guest. Cyborg and Nova will not ask libvirt to detach the device
from a host driver or to re-attach it later.

Deployers must continue to ensure that a device is not configured for both
Nova native PCI passthrough and Cyborg management.

Developer impact
----------------

Cyborg driver developers that emit PCI attach handles may add the same
``managed`` key to their ``attach_handle_info`` in a later change. Nova will
consume the key generically for all Cyborg PCI attach handles.

Nova developers need a small libvirt-driver change to read the key from ARQ
``attach_handle_info``. Nova should default missing values to managed mode
enabled so that older Cyborg services and older ARQ responses continue to
work.

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

* Add the Cyborg API microversion, expected to be ``2.4``.
* Update Cyborg API version history for the new ARQ ``attach_handle_info``
  field.
* Extend the generic PCI whitelist/device-spec parsing to accept and validate
  ``managed``.
* Normalize accepted true values such as ``true`` and ``yes`` and false values
  such as ``false`` and ``no``.
* Store configured managed mode in the generic PCI driver's PCI attach handle
  ``attach_info`` JSON.
* For ARQ responses in microversion ``2.4`` and later, include
  ``attach_handle_info.managed`` as a boolean for bound PCI attach handles,
  defaulting to ``true`` when the stored marker is absent.
* Ensure older microversions do not include ``attach_handle_info.managed``.
* Add Cyborg unit tests for config parsing, attach handle generation, ARQ
  serialization, microversion behavior, and defaulting.
* Update Nova's libvirt driver to consume ``attach_handle_info.managed`` for
  all Cyborg PCI ARQs and to default missing values to managed mode enabled.
* Add Nova unit tests for ``managed=False``, ``managed=True``, and missing
  ``managed`` in Cyborg PCI ARQs.
* Update OpenStack SDK accelerator support for the new microversion and add
  tests for ARQ ``attach_handle_info.managed``.
* Update ``python-cyborgclient`` current/default microversion handling and
  tests so ARQ list/show output covers the nested ``managed`` value.
* Add first-party CI coverage using a fake SR-IOV PCI kernel module and
  Tempest scenario tests for the generic PCI driver.
* Update Cyborg and Nova administrator documentation.

Dependencies
============

* Nova's native PCI passthrough managed-mode support for VFIO variant drivers:
  https://specs.openstack.org/openstack/nova-specs/specs/2025.1/implemented/enable-vfio-devices-with-kernel-variant-drivers.html
* Nova/Cyborg interaction and ARQ flow:
  https://specs.openstack.org/openstack/cyborg-specs/specs/ussuri/implemented/nova-cyborg-interaction.html
* Cyborg ARQ API:
  https://specs.openstack.org/openstack/cyborg-specs/specs/ussuri/implemented/cyborg-api.html
* A companion Nova change is required so that Nova consumes the new ARQ field.
  No new Nova REST API is required.
* OpenStack SDK and ``python-cyborgclient`` updates are required so clients can
  negotiate and test the new Cyborg microversion.

Testing
=======

Cyborg unit tests should cover:

* PCI whitelist entries with no ``managed`` key.
* PCI whitelist entries with true values such as ``true`` and ``yes``.
* PCI whitelist entries with false values such as ``false`` and ``no``.
* Invalid ``managed`` values.
* Generated PCI attach handles with configured managed mode.
* Bound ARQ API responses for older microversions where ``managed`` is absent.
* Bound ARQ API responses for microversion ``2.4`` and later where
  ``managed`` is present and defaults to ``true`` if absent from stored
  ``attach_info``.

Nova unit tests should cover libvirt XML generation for Cyborg PCI ARQs with
``managed=False``, with ``managed=True``, and with no ``managed`` key.

OpenStack SDK and ``python-cyborgclient`` tests should cover microversion
selection and ARQ show/list data containing ``attach_handle_info.managed``.

End-to-end test coverage will be added to first-party CI using a fake SR-IOV
PCI kernel module. The module creates software-backed PCI PFs and VFs which can
be bound to a VFIO variant driver and assigned to QEMU guests without physical
SR-IOV hardware. This will allow Tempest scenario tests to validate the Cyborg
generic PCI driver flow, including ARQ binding, Nova XML generation, and guest
boot with the assigned fake VF.

The first-party fake hardware test will validate the common control-plane and
libvirt integration path. Hardware or third-party tests may still be added
later for specific production devices and vendor drivers.

Documentation Impact
====================

Cyborg administrator documentation should describe the new generic PCI driver
``managed`` tag, its default, accepted values, and the operator responsibility
for ``managed=False`` devices.

Nova administrator documentation for Cyborg-managed accelerators should mention
that Nova consumes ``attach_handle_info.managed`` for Cyborg PCI ARQs when the
Cyborg service supports the new microversion.

OpenStack SDK and ``python-cyborgclient`` release notes or user documentation
should mention support for the new Cyborg microversion and ARQ response field.

References
==========

* Kernel VFIO PCI variant driver documentation:
  https://docs.kernel.org/driver-api/vfio-pci-device-specific-driver-acceptance.html
* Nova VFIO variant driver managed-mode spec:
  https://specs.openstack.org/openstack/nova-specs/specs/2025.1/implemented/enable-vfio-devices-with-kernel-variant-drivers.html
* Nova/Cyborg interaction spec:
  https://specs.openstack.org/openstack/cyborg-specs/specs/ussuri/implemented/nova-cyborg-interaction.html
* Cyborg API spec:
  https://specs.openstack.org/openstack/cyborg-specs/specs/ussuri/implemented/cyborg-api.html
* Nova libvirt XML behavior for PCI hostdevs:
  https://libvirt.org/formatdomain.html#host-device-assignment
* Prototype fake SR-IOV PCI kernel module for first-party CI:
  https://github.com/SeanMooney/cyborg-extra/blob/master/pci-sim/fake_pci_sriov.c

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2026.2
     - Introduced
