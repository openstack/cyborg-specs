..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Cyborg Intel® x710 driver proposal
==================================

This spec proposes to provide a new Cyborg driver for Intel® x710 dirver,
which is mentioned in Nova's SRIOV Nic Support spec [1]_. In Cyborg side,
we need to implement a driver for SRIOV Nic in order to manage the lifecycle
of this device, and to allow Nova request this kind of resource from Cyborg.


Problem description
===================
The Intel Ethernet Converaged Network Adapter x710 [2]_ addressed the demanding
need of an agile data center by providing unmatched features for both server
and virtualization, flexibility for LAN or SAN networks and proven, reliable
performance.

This device also support SR-IOV virtualization technology, it can be
virtualized up to 128 VFs, each VF can support a unique and separate data path
for I/O related functions within the PCI Express hierarchy.

Using SR-IOV with the networking device, for example, allow the bandwidth of a
single port(function) to be partitioned into smaller slices that can be
allocated to specific VMs or guest, via a standard interface.

One of the key features of x710 Network Adapter is Dynamic Device
Personalization(DDP) [3]_.

DDP allows dynamic reconfiguration of the packet processing pipeline to meet
specific use case needs on demand, adding new packet processing pipeline
configuration profiles to a network adapter at run time, without resetting or
rebooting the server. Software applies these custom profiles in a nonpermanent,
transaction-like mode, so the original network controller’s configuration is
restored after network adapter reset, or by rolling back profile changes by
software. The DPDK provides all APIs to handle DDP packages.


Use Cases
---------
* As an operator, I want to use Cyborg to manage lifecycle of Intel X710.
  Cyborg side driver should be able to manage this kind of acceleration
  resources including discover it, report it to Placement and program it.
* As an end user, I want to boot up a VM with Intel X710 to accelerate network
  related workloads, Cyborg should be able to interact with Nova to assign it
  the VM.
* As an advanced end user, I want to customize a profile to load on Intel X710
  when booting up a VM with it, Cyborg driver should be able to provide such
  program interface.

Proposed change
===============

In general, the goal is to develop a Cyborg Intel® x710 driver that supports
discover() and program() interfaces.

Discovery
---------

The driver should include discover functions which will be called by agent
periodically, and encapsule the device information to Cyborg's unified data
model, such as Device, Deployable, AttachHandle and so on.

Besides, we should also implement a configuration file to let Cyborg driver
to discover the loaded DDP profile on X710, as well as the physnet name
associated with this nic and then Cyborg can report these as the attributes.

An example of the configuration file is:

.. code-block:: RST

    [nic_devices]
    enabled_nic_types = x710_static

    [x710_static]
    physical_device_mappings = physnet1:eth2|eth3
    function_device_mappings = GTPv1:eth3|eth2


Programming
-----------

The driver should implement a program() function which allow Cyborg API to
call to load a specific profile on the nic, this is a part-implementation of
the third use case mentioned above.

To implement this functionality, Nova needs to parse user's request and get the
profile's name, and Cyborg should get this profile's info from Nova and let
this driver to load profile to the selected device.

Briefly, this function need to call some utility like (DPDK API or ethtool)
to program the nic. For the first phase, we will work on the pre-programmed
scenario. Another spec will be introduced to show the total design of dynamic
programming functionality of this driver in the future.


Alternatives
------------

None

Data model impact
-----------------

One physical X710 card has 2 PFs, and each of them can be virtualized into
several VFs. The maximum number of the VFs are different according to
different models.

Here, we take an example of 4 VFs in each PFs.

The data model in Cyborg should like this::

  +---------------------------------------------------+
  |                                                   |
  |      device A                    device B         |
  |   +------------------+     +------------------+   |
  |   | deployable 1-4   |     | deployables 5-8  |   |
  |   | +----+ +----+    |     | +----+  +----+   |   |
  |   | | vf | | vf |    |     | | vf |  | vf |   |   |
  |   | +----+ +----+    |     | +----+  +----+   |   |
  |   | +----+ +----+    |     | +----+  +----+   |   |
  |   | | vf | | vf |    |     | | vf |  | vf |   |   |
  |   | +----+ +----+    |     | +----+  +----+   |   |
  |   +------------------+     +------------------+   |
  | X710                                              |
  +---------------------------------------------------+

The resource provider tree's structure should be reported as::

                       +--------------+
                       |              |
                       | compute node |
                       |              |
                       +-------+------+
                               |
        +----------------+-----+-----------------------+
        |                |                             |
        v                v                             v
  +-----+-----+     +----+-------+                 +---+--------+
  |           |     |            |                 |            |
  |deployable1|     |deployable 2|   ...           |deployable 8|
  +-----------+     +------------+                 +------------+


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


Developer impact
----------------

Deployer need to install DPDK toolkit to configure the DDP feature.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Xinran Wang(xin-ran.wang@intel.com)


Dependencies
============

None

Testing
========

* Unit tests will be added to test this driver.
* Add test result in Cyborg Wiki which is required by the Cyborg community.

Documentation Impact
====================

Need add or update related documentations.

References
==========
.. [1] https://review.opendev.org/c/openstack/nova-specs/+/742785
.. [2] https://www.intel.com/content/dam/www/public/us/en/documents/product-briefs/ethernet-x710-brief.pdf
.. [3] https://software.intel.com/content/www/us/en/develop/articles/dynamic-device-personalization-for-intel-ethernet-700-series.html

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Wallaby
     - Introduced
