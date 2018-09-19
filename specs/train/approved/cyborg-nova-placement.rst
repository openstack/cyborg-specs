..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Cyborg-Nova Placement Interaction
=================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/cyborg-nova-placement

This specification describes the way Cyborg represents devices and
accelerators in Placement.

This spec is common to all accelerators, including GPUs, ASIC-based devices,
etc. Since FPGAs have more aspects to be considered than other devices, some
sections highlight FPGA-specific factors.

Problem description
===================
Cyborg should represent devices and accelerators in Placement so that Nova can
schedule instances with accelerators.

Though PCI Express is entrenched in the data center, devices may be identified
by some identifier other than PCI bus-device-functions and accelerators may be
attached to instances by attach handles other than PCI functions. Accordingly,
Cyborg should not represent PCI functions in Placement.

Terminology
-----------
* Accelerator: The unit that can be assigned to an instance for offloading
  specific functionality. For non-FPGA devices, it is either the device itself
  or a virtualized version of it (e.g. vGPUs). For FPGAs, an accelerator is
  either the entire device, a region within the device or a function.

* Bitstream: An FPGA image, usually a binary file, possibly with
  vendor-specific metadata. A bitstream may implement one or more functions.

* Function: A specific functionality, such as matrix multiplication or video
  transcoding, usually represented as a string or UUID. This term may be used
  with multi-function devices, including FPGAs and other fixed function
  hardware like Intel QuickAssist.

* Region: A part of the FPGA which can be programmed without disrupting other
  parts of that FPGA. If an FPGA does not support Partial Reconfiguration, the
  entire device constitutes one region. A region may implement one or more
  functions.

Background
----------
A Cyborg device has one or more components named deployables, each of which
contains one or more accelerators. A device has a management interface, whose
address is the control path identifier: for SR-IOV devices, this is usually
the PCI Physical Function (PF). Each deployable has one or more attach
handles, usally one for each accelerator resource in the deployable. The
attach handle represents the object by which an accelerator is associated with
an instance: for SR-IOV devices, this is usually the PCI Virtual Function
(VF).

A device may have components, such as flash memory or BMC, which are not of
relevance to Nova or Placement. Those components may have attributes, such as
flash memory capacity or BMC firmware version.

This is diagrammatically shown below::

 Control Path Identifier
       +----+
       |    |
     +-+----+--------------------------------------+
     |                                             |
     |     Attach Handles       Attach Handles     |
     |      +--+    +--+         +--+    +--+      |
     |      |  |    |  |         |  |    |  |      |
     |    +-+--+----+--+-+     +-+--+----+--+-+    |
     |    |  ACC1  ACC2  |     |  ACC1  ACC2  |    |
     |    +--------------+     +--------------+    |
     |      Deployable 1         Deployable 2      |
     |                                             |
     +---------------------------------------------+

Use cases
---------
Operators should be able to define device profiles for any or all of the
tenant use cases defined below, using the set of resources, traits and
properties published by Cyborg. The tenant (user) should be able to specify
the accelerators needed for an instance with an operator-defined device
profile.

The use cases for the tenant role are as below:

* Device as a Service (DaaS): The user asks for a deployable that she will
  manage herself. This is meant for power users who know device-specific
  details. Example: A GPU user asks for a specific GPU model that can support
  specific driver version(s).

  * FPGA variation: The user chooses the bitstream that needs to be programmed
    into the device (or region). To address the potential security concerns,
    we define three variations, the first two of which delegate bitstream
    programming to Cyborg for mitigating some risks:

    * Request-time Programming: The device profile specifies a bitstream.
      Cyborg applies the bitstream before instance bringup.

    * Run-time Programming: The instance may request one or more
      bitstreams dynamically. Cyborg receives the request and does
      the programming. (This use case may not be addressed in Train.)

* Accelerated Function as a Service (AFaaS): The user asks for a function
  (e.g. ipsec) or an algorithm that needs to be offloaded. The accelerator
  containing that function should be assigned to the instance.

  This does not require detailed knowledge of the devices or bitstreams, and
  thus enables a larger audience of users.

  The operator may satisfy this use case in two ways:

  * Pre-programmed: Do not allow orchestration to modify any function,
    for any of these reasons:

    * Only fixed function hardware is available. (E.g. ASICs.)

    * Operational simplicity.

    * Assure tenants of programming security, by doing all programming offline
      through some audited process.

  * Orchestration-programmed: For FPGAs, allow orchestration to program as
    needed, to maximize flexibility and availability of resources.

An operator must be able to provide both Device as a Service and Accelerated
Function as a Service in the same cluster.

Proposed change
===============

Representation
--------------

* Cyborg will represent a generic accelerator for a device type as a
  standard or custom Resource Class (RC) for that type. Standard RCs have
  been proposed for GPUs and FPGAs: PGPU and FPGA [#std-names]_. For others,
  Cyborg will create custom RCs of the form CUSTOM_ACCELERATOR_<device-type>.
  E.g. CUSTOM_ACCELERATOR_AICHIP.  Using a different RC for each device type
  helps in defining separate quotas for different device types.

  * Note that an accelerator is not an object, either in Cyborg or in
    Placement. It is a virtual resource, represented as inventories of RPs in
    Placement.

  * In the future, there could be other resource classes. For example, there
    could be a RC for device-local memory (i.e., memory available to the
    device alone, usually in the form of DDR, QDR or High Bandwidth Memory in
    the PCIe board along with the device).

* Since a device may have componnets that are not of relevance to Placement or
  Nova (see Section `Background`_ ), Cyborg does not represent devices
  directly in Placement.

* Each deployable is represented as a Resource Provider (RP) in Placement.
  Each accelerator in that deployable is represented as an inventory of the
  corresponding RP.

* Cyborg will associate a Device Family trait with each deployable as
  needed, of the form CUSTOM_<device-type>_<vendor>_<family>.
  E.g. CUSTOM_FPGA_INTEL_ARRIA10.
  This is not a product name, but the name of a device family, used to
  match software in the instance image with the device family.

* For FPGAs, Cyborg will associate a region type trait with each region
  (or with the FPGA itself if there is no Partial Reconfiguration
  support), of the form CUSTOM_FPGA_REGION_<vendor>_<id-string>.
  E.g.  CUSTOM_FPGA_REGION_INTEL_<id-string>. This is needed for Device
  as a Service with FPGAs.

* For FPGAs, Cyborg may associate a function type trait with a region
  when the region gets programmed, of the form
  CUSTOM_FPGA_FUNCTION_ID_<vendor>_<id-string>. E.g.
  CUSTOM_FPGA_FUNCTION_ID_INTEL_<gzip-id>. This is needed for AFaaS use case.

* In addition to the above, Cyborg will associate additional custom traits
  reported by the Cyborg driver for that deployable.

* In addition to the traits, user requests for accelerators may include
  Cyborg-specific properties in the device profile. These are not interpreted
  by Nova. They are explained in the Device Profiles specification
  ([#accel-keys]_).

Usage in device profiles
------------------------
This section shows how an operator can realize the 'Use cases'_ in device
profiles using the RCs and traits from previous section, along with
Cyborg-specific properties.

To recap from [#devprof]_, the request to create an instance with accelerators
may use a flavor with an embedded device profile, or a flavor and a device
profile separately. In either case, the accelerator specification is entirely
in the device profile.

We now show how to define device profiles for various use cases.

* Some example device profiles for DaaS for a generic device::

  | resources:ACCELERATOR_GPU=1
  | trait:CUSTOM_GPU_AMD_RADEON_R9=required

  | resources:CUSTOM_ACCELERATOR_HPTS=1
  | trait:CUSTOM_HPTS_ZTE=required

* Example device profile for DaaS with request-time programming for FPGAs::

  | resources:ACCELERATOR_FPGA=1
  | trait:CUSTOM_FPGA_REGION_ID_INTEL_<id>=required
  | accel:bitstream_id=3AFB

* Example device profile for DaaS with run-time programming for FPGAs::

  | resources:CUSTOM_ACCELERATOR_FPGA=1
  | trait:CUSTOM_FPGA_REGION_INTEL_<name>=required
  | accel:bitstream_at_runtime=required

* Example device profile for AFaaS pre-programmed FPGAs::

  | resources:CUSTOM_ACCELERATOR_FPGA=1
  | trait:CUSTOM_FPGA_INTEL_ARRIA10=required
  | trait:CUSTOM_FPGA_FUNCTION_ID_INTEL_<gzip-id-string>=required

  Since the function trait is required, if no free accelerator with that trait
  is available, the request would fail. There would be no attempt to reprogram
  another FPGA to get that function.

* Example device profile for AFaaS Orchestration-Programmed::

  | resources:CUSTOM_ACCELERATOR_FPGA=1
  | trait:CUSTOM_FPGA_INTEL_ARRIA10=required
  | accel:function:CUSTOM_FPGA_FUNCTION_INTEL_<gzip-id-string>=required

  The function is not specified as a trait and so Nova ignores it. Placement
  returns the list of all allocation candidates which contain the only stated
  trait, i.e., the list of all nodes with that FPGA device model. During
  binding, Cyborg will notice whether the selected device RP actually has the
  requested function and, if not, will initiate reprogramming.

  NOTE: When Nova supports preferred traits, we can use that instead of
  'function' keyword in extra specs.

  NOTE: For Cyborg to fetch the bitstream for this function, it is assumed
  that the operator has configured the function ID as a property of the
  bitstream image in Glance.

* Another example device profile for AFaaS Orchestration-Programmed which
  refers to a function by name instead of ID for ease of use::

  | resources:CUSTOM_ACCELERATOR_FPGA=1
  | trait:CUSTOM_FPGA_INTEL_ARRIA10=required
  | accel:function_name:<string>=required

  NOTE: This assumes the operator has configured the function name as a
  property of the bitstream image in Glance. The FPGA hardware is not expected
  to expose function names, and so Cyborg will not represent function names as
  traits.

Alternatives
------------

N/A

Data model impact
-----------------

Following changes are needed in Cyborg.

* Do not publish PCI functions as resources in Nova. Instead, publish
  RC/RP info to Nova, and keep RP-PCI mapping internally.

* Cyborg should associate RPs/RCs and PFs/VFs with Deployables in its
  internal DB (as part of the deployable attributes table).

* Driver/agent interface needs to report device/region types so that
  RCs can be created.

* Deployables table should track which RP corresponds to each Deployable.

REST API impact
---------------

No change is needed in Placement API.

Cyborg API to create and use device profiles is covered in the Device Profiles
specification [#devprof]_.

Cyborg API for use by Nova during the instance spawn/update workflow is
covered elsewhere.

Security impact
---------------

The use cases for DaaS run-time programming and direct programming will not be
taken up till the security issues are addressed.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

Operators must define device profiles based on the RCs, traits and Cyborg
properties.

Developer impact
----------------

When a custom trait in one release becomes a standard trait in another
release, there needs to be an upgrade script to translate device profiles.

Implementation
==============

Assignee(s)
-----------

None

Work Items
----------

The code changes needed to realize this device representation are described in
other specs. Work items are stated in those specs.

Dependencies
============

* `Enable use of Nested Resource Providers in Nova Scheduler
  <https://review.openstack.org/#/q/topic:use-nested-allocation-candidates>`_

Testing
=======

The code changes needed to realize this device representation are described in
other specs. Testing requirements are stated in those specs.

Documentation Impact
====================

Usage of RCs, traits and Cyborg properties must be documented for operators.

References
==========

.. [#devprof] `Device Profiles <https://review.openstack.org/#/c/602978/>`_

.. [#std-names] `Standard accelerator names
   <https://review.opendev.org/#/c/657464>`_

.. [#accel-keys] `Device profile accel keys
   <https://opendev.org/openstack/cyborg-specs/src/branch/master/specs/train/approved/device-profiles.rst#valid-accel-keys>`_

.. [#nRP] `Enable use of Nested Resource Providers in Nova Scheduler
  <https://review.openstack.org/#/q/topic:use-nested-allocation-candidates>`_

History
=======

Optional section intended to be used each time the spec is updated to describe
new design, API or any database schema updated. Useful to let reader know
what happened over time.

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Rocky
     - Introduced as cyborg-nova-sched.rst
   * - Stein
     - Updated
   * - Train
     - Updated

