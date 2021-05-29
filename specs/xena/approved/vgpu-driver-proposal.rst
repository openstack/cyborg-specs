..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Cyborg NVIDIA GPU Driver support vGPU management
================================================

The Cyborg NVIDIA GPU Driver has implemented pGPU management in the Train
release, this spec proposes the specification of supporting vGPU management
in the same driver.

Problem description
===================

GPU devices can provide supercomputing capabilities, and can replace the CPU
to provide users with more efficient computing power at a lower cost. GPU cloud
servers have great value in the following application scenarios, including:
video encoding and decoding, scientific research and artificial intelligence
(deep learning, machine learning).

In the OpenStack ecosystem, users can now use Nova to pass gpu resources to
guest by two methods:

* Pass the GPU hardware to the guest (PCI pass-through).

* Pass the Mediated Device(vGPU) to the guest.

With the long-term goal that Cyborg will manage heterogeneous accelerators
including GPUs, Cyborg needs to support GPU management and integrate with Nova
to provide users with gpu resources allocation in the aforementioned methods.
The existing Cyborg GPU driver, NVIDIA GPU Driver, has supported the first
method (PCI pass-through), while the second method is not yet supported.
Please see ref [1]_ for Nova-Cyborg vGPU integration spec.

Use Cases
---------

* When the user is using Cyborg to manage GPU devices, he/she wants to boot
  up a VM with Nvidia GPU (pGPU or vGPU) attached in order to accelerate the
  video coding and decoding, Cyborg should be able to manage this kind of
  acceleration resources and to assign it to the VM(binding).

Proposed changes
================

To be clear, in the following, we will describe the whole process of how does
the NVIDIA GPU Driver discover, generate Cyborg specific driver objects of the
vGPU devices(comply with Cyborg Database Model), and report it to cyborg-db
and Placement by cyborg-conductor. Features that are aleady supported in
current branch is marked as DONE, new changes are marked as NEW CHANGES.

1. Collect raw info of GPU devices from compute node by "lspci" and grep
nvidia related keyword.(DONE)

2. Parsing details from each record including ``vendor_id``, ``product_id``
and ``pci_address``.(DONE)

3. Generate Cyborg specific driver objects and resource provider modeling
for the GPU device as well as its mdiated devices. Below is the objects to
describe a vGPU devices which complies with the Cyborg database mode [4]_
and placement data model [5]_.(NEW CHANGE)

::

  Hardware     Driver objects       Placement data model
     |               |                      |
  1 GPU         1 device                    |
     |               |                      |
     |         1 deployable       ---> resource_provider
     |               |            ---> parent resource_provider: compute node
     |               |                      |
  4 vGPUs     4 attach_handles    ---> inventories(total:4)

4. Supporting set the vGPU type for a specific GPU device in cyborg.conf. The
implementation is similar to that in Nova [9]_.(NEW CHANGE)

* Firstly, we propose [gpu]/enabled_vgpu_types to define which vgpu type Cyborg
  driver can use:

  ::

    [gpu]
    enabled_vgpu_types = [str_vgpu_type_1, str_vgpu_type_2, ...]

* And also, we propose that Cyborg driver will accept configuration sections
  that are related to the [gpu]/enabled_vgpu_types and specifies which
  exact pGPUs are related to the enabled vGPU types and will have a
  device_addresses option defined like this:

  ::

    cfg.ListOpt('device_addresses',
                default=[],
                help="""
    List of physical PCI addresses to associate with a specific GPU type.

    The particular physical GPU device address needs to be mapped to the vendor
    vGPU type which that physical GPU is configured to accept. In order to
    provide this mapping, there will be a CONF section with a name
    corresponding to the following template: "vgpu_type_%(vgpu_type_name)s

    The vGPU type to associate with the PCI devices has to be the section name
    prefixed by ``vgpu_``. For example, for 'nvidia-11', you would declare
    ``[vgpu_nvidia-11]/device_addresses``.

    Each vGPU type also has to be declared in ``[devices]/enabled_vgpu_types``.

    Related options:

    * ``[gpu]/enabled_vgpu_types``
    """),

  For example, it would be set in cyborg.conf

  ::

    [gpu]
    enabled_vgpu_types = nvidia-223,nvidia-224
    [vgpu_nvidia-223]
    device_addresses = 0000:af:00.0,0000:86:00.0
    [vgpu_nvidia-224]
    device_addresses = 0000:87:00.0

5. Generate resource_class and traits for device, which later will also be
reported to Placement, and used by nova-scheduler to filter appropriate
accelerators.(NEW CHANGE)

* ``resource class`` follows standard resources classes used by OpenStack [6]_.
  Pass-through GPU device will report 'PGPU' as its resource class,
  Virtualized GPU device will report 'VGPU' as its resource class.

* ``traits`` follows the placement custom trait format [7]_. In the Cyborg
  driver, it will report two traits for vGPU accelerator using the format
  below:

  trait1: **OWNER_CYBORG**.

  trait2: **CUSTOM_<VENDOR_NAME>_<PRODUCT_ID>_<Virtual_GPU_Type>**.

  Meaning of each parameter is listed below.

  * OWNER_CYBORG: a new namespace in os-traits to remark that a device is
    reported by Cyborg when the inventory is reported to placement. It is used
    to distinguish GPU devices reported by Nova.

  * VENDOR_NAME: vendor name of the GPU device.

  * PRODUCT_ID: product ID of the GPU device.

  * Virtual_GPU_Type: this parameter is actually another format of the
    enabled_vgpu_types for a specific device set by admin in cyborg.conf.
    In order to generate this param, driver will first retrieve
    ``enabled_vgpu_type`` and then map it to Virtual_GPU_Type by the way
    showed below. The name is exactly the Virtual_GPU_Type that will be
    reported in traits. For more details about the valid Virtual GPU Types
    for supported GPUs, please refer to [8]_.

  ::

    # find mapping relation between Virtual_GPU_Type and enabled_vgpu_type.
    # The value in "name" file contains its corresponding Virtual_GPU_Type.
    cat /sys/class/mdev_bus/{device_address}/mdev_supported_types/{enabled_vgpu_type}/name

* Here is a example to show the traits of a GPU device in the real world.

  * A Nvidia Tesla T4 device has been successfully installed on host,
    device address is 0000:af:00.0. In addition, the vendor’s vGPU driver
    software must be installed and configured on the host at the same time.

  ::

    [vtu@ubuntudbs ~]# lspci -nnn -D|grep 1eb8
    0000:af:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)

  * Enable GPU types (Accelerator)

    1. Specify which specific GPU type(s) the instances would get from this
    specific device.

    Edit devices.enabled_vgpu_types and device_address in cyborg.conf:

    ::

      [gpu]
      enabled_vgpu_types=nvidia-223
      [vgpu_nvidia-223]
      device_addresses = 0000:af:00.0

    2. Restart the cyborg-agent service.

  * Finally, traits reported for this device(RP) will be:

    **OWNER_CYBORG** and **CUSTOM_NVIDIA_1EB8_T4_2B**

.. NOTE::

  For the last parameter "T4_2B" (<Virtual_GPU_type>), we can validate the
  mapping relation between "nvidia-223" and "T4_2B" by check from the mdev
  sys path:

  ::

    [vtu@ubuntudbs mdev_supported_types]$ pwd
    /sys/class/mdev_bus/0000:af:00.0/mdev_supported_types
    [vtu@ubuntudbs mdev_supported_types]$ ls
    nvidia-222  nvidia-225  nvidia-228  nvidia-231  nvidia-234  nvidia-320
    nvidia-223  nvidia-226  nvidia-229  nvidia-232  nvidia-252  nvidia-321
    nvidia-224  nvidia-227  nvidia-230  nvidia-233  nvidia-319
    [vtu@ubuntudbs mdev_supported_types]$ cat nvidia-223/name
    GRID T4-2B

6. Generate ``controlpath_id``, ``deployable``, ``attach_handle``,
``attribute`` for vGPU.(NEW CHANGE)

7. Create a mdev device in the sys by echo its UUID (actually is the
attach_handle UUID) to the create file when vgpu is bind to a VM.(NEW CHANGE)

create_file_path=
/sys/class/mdev_bus/{pci_address}/mdev_supported_types/{type-id}/create

8. Delete a mdev device from sys by echo "1" to the remove file when vgpu is
unbind from a VM.(NEW CHANGE)

remove_file_path=
/sys/class/mdev_bus/{pci_address}/mdev_supported_types/{type-id}/UUID/remove

Alternatives
------------

Using Nova to manage vGPU device [10]_.

Data model impact
-----------------

One pGPU can be virtualized 4 vGPUs, 8 vGPUs, 16 vGPUs base on the pGPU type.

Here we take 2 pGPUs and each pGPU is virtualized 4 vGPUs as example.

The data model in Cyborg should like this::

  +--------------------------------------------------------+
  |                                                        |
  |    device A                    device B                |
  |                                                        |
  |  +-----------------------+  +-----------------------+  |
  |  |  deployable A         |  |  deployable B         |  |
  |  |                       |  |                       |  |
  |  | +-------------------+ |  | +-------------------+ |  |
  |  | | attach handler 1-4| |  | | attach handler 5-8| |  |
  |  | |                   | |  | |                   | |  |
  |  | | +------+ +------+ | |  | | +------+ +------+ | |  |
  |  | | | vgpu1| | vgpu2| | |  | | | vgpu5| | vgpu6| | |  |
  |  | | +------+ +------+ | |  | | +------+ +------+ | |  |
  |  | | +------+ +------+ | |  | | +------+ +------+ | |  |
  |  | | | vgpu3| | vgpu4| | |  | | | vgpu7| | vgpu8| | |  |
  |  | | +------+ +------+ | |  | | +------+ +------+ | |  |
  |  | +-------------------+ |  | +-------------------+ |  |
  |  |                       |  |                       |  |
  |  +-----------------------+  +-----------------------+  |
  |                                                        |
  |  VGPU                                                  |
  +--------------------------------------------------------+

The resource provider tree's structure should be reported as ::

                      +--------------------+
                      |                    |
                      |  compute node      |
                      |                    |
                      +---------+----------+
                                |
                   +------------+----------------+
                   |                             |
            +------+------+               +------+------+
            |             |               |             |
            |  deployable |               | deployable  |
            +-----+-------+               +------+------+
                  |                              |
      +-----------+--------+                +----+---------------+
      |                    |                |                    |
      |                    |                |                    |
 +-----+------+       +-----+-----+     +----+------+      +------+-----+
 | inventory1 | ...   | inventory4|     | inventory5| ...  | inventory8 |
 +------------+       +-----------+     +-----------+      +------------+



REST API impact
---------------

None


Security impact
---------------

None

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

This feature is highly dependent on the version of libvirt and the physical
devices present on the host.

For vGPU management, deployers need to make sure that the GPU device has been
successfully virtualized. Otherwise, Cyborg will report it as a pGPU device.

Please see ref [2]_ and [3]_ for how to install the Virtual GPU Manager package
to virtualize your GPU devices.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <yumeng-bao>

Other contributors:
  songwenping

Work Items
----------

* Implement NVIDIA GPU Driver enhancement in Cyborg
* Add related test cases.
* Add test report to wiki and update the supported driver doc page

Dependencies
============

None

Testing
========

* Unit tests will be added to test this driver.

Documentation Impact
====================

Document Nvidia GPU driver in Cyborg project.

References
==========
.. [1] https://review.opendev.org/#/c/750116/
.. [2] https://docs.nvidia.com/grid/6.0/grid-vgpu-user-guide/index.html
.. [3] https://docs.nvidia.com/grid/6.0/grid-vgpu-user-guide/index.html#install-vgpu-package-generic-linux-kvm
.. [4] https://specs.openstack.org/openstack/cyborg-specs/specs/stein/implemented/cyborg-database-model-proposal.html
.. [5] https://docs.openstack.org/nova/rocky/user/placement.html#references
.. [6] https://github.com/openstack/os-resource-classes/blob/master/os_resource_classes/__init__.py#L41
.. [7] https://specs.openstack.org/openstack/nova-specs/specs/pike/implemented/resource-provider-traits.html
.. [8] https://docs.nvidia.com/grid/latest/grid-vgpu-user-guide/index.html#virtual-gpu-types-grid-reference
.. [9] https://specs.openstack.org/openstack/nova-specs/specs/ussuri/implemented/vgpu-multiple-types.html
.. [10] https://docs.openstack.org/nova/latest/admin/virtual-gpu.html

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release
     - Description
   * - Wallaby
     - Introduced
   * - Xena
     - Reproposed
