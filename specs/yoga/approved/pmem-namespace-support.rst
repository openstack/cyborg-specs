..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Cyborg Intel PMEM Driver Proposal
=================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/add-pmem-driver

This spec proposes to provide the initial design for Cyborg's Intel PMEM
driver.

Problem description
===================

This spec will add Intel PMEM driver for Cyborg to manage specific Intel
PMEM devices.

PMEM devices can be used as a large pool of low latency high bandwidth memory
where they could store data for computation. This can improve the performance
of the instance.

PMEM must be partitioned into PMEM namespaces [1]_ for applications to use.
This vPMEM feature only uses PMEM namespaces in devdax mode as QEMU vPMEM
backends [2]_. If you want to dive into related notions, the document NVDIMM
Linux kernel document [3]_ is recommended.

Starting in the 20.0.0 (Train) release, the virtual persistent memory (vPMEM)
feature in Nova allows a deployment using the libvirt compute driver to provide
vPMEMs for instances using physical persistent memory (PMEM) that can provide
virtual devices [4]_.

Use Cases
---------
* As an operator, I would like to use Cyborg agent managing PMEM resource
  and checking periodically, the Cyborg Intel PMEM driver should provide
  ``discover()`` function to enumerate the list of the Intel PMEM devices,
  and report the details of all available Intel PMEM accelerators on the
  host, such as PID(Product id), VID(Vendor id), Device ID.

* As a user, I would like to boot up a VM with Intel PMEM Device attached in
  order to accelerate compute ability. Cyborg should be able to manage this
  kind of acceleration resources and assign it to the VM(binding).

Proposed change
===============
1. In general, the goal is to develop a Intel PMEM Device driver that supports
discover interfaces for Intel PMEM accelerator framework. The driver should
include the ``discover()`` function. This function works excuting "ndctl list"
command that reports devices' raw info sample as following::

  [
    {
    "vendor": "8086",
    "product": "ns200_0",
    "device": "dax0.0"
    }
  ]

2. Generate Cyborg specific driver objects and resource provider modeling
for the PMEM device. Below is the objects to describe a PMEM devices which
complies with the Cyborg database mode and Placement data model.

::

  Hardware     Driver objects       Placement data model
     |               |                      |
  1 PMEM         1 device                    |
     |               |                      |
     |         1 deployable       ---> resource_provider
     |               |            ---> parent resource_provider: compute node
     |               |                      |
  n Namespace  n attach_handle    ---> inventories(total:n)

3. Need add the "enable_driver=intel_pmem_driver" in the Cyborg Agent
   configure file.

4. Need add the "pmem_namespaces=$LABEL:$NSNAME|$NSNAME,$LABEL:$NSNAME|$NSNAME"
   in the Cyborg Agent configure file as:
   "pmem_namespaces = 6GB:ns0|ns1|ns2,LARGE:ns3"

5. Resource class follows standard resources classes as:
    "CUSTOM_PMEM_NAMESPACE_$LABEL"

6. Traits follows the placement custom trait format. In the Cyborg driver, it
   will report two traits for PMEM accelerator using the format below:
   trait1:"CUSTOM_PMEM_NAMESPACE_$LABEL1"
   trait2:"CUSTOM_PMEM_NAMESPACE_$LABEL2"


7. Before cyborg discover the namespaces, they should be created. How to create
   the namespce can reference [5]_ and [6]_.

Alternatives
------------

None

Data model impact
-----------------

Need add new type such as PMEM in devices and attach_handle tables.

REST API impact
---------------

None.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

User can manage Intel PMEM Device by Cyborg Intel PMEM driver. Such as list
of the Intel PMEM devices, report the details of all available Intel PMEM
accelerators on the host, binding with Intel PMEM and so on.

Performance Impact
------------------

None

Other deployer impact
---------------------

None.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  qiujunting(qiujunting@inspur.com)

Work Items
----------

* Implement Intel PMEM driver in Cyborg
* Add related test cases.


Dependencies
============

None

Testing
========

* Unit tests will be added to test this driver.

Documentation Impact
====================

Document Intel PMEM driver in Cyborg project.
Add test report in cyborg wiki.

References
==========
.. [1] https://pmem.io/ndctl/ndctl-create-namespace.html
.. [2] https://github.com/qemu/qemu/blob/19b599f7664b2ebfd0f405fb79c14dd241557452/docs/nvdimm.txt#L145
.. [3] https://www.kernel.org/doc/Documentation/nvdimm/nvdimm.txt
.. [4] https://docs.openstack.org/nova/latest/admin/virtual-persistent-memory.html
.. [5] https://docs.openstack.org/nova/latest/admin/virtual-persistent-memory.html#configure-pmem-namespaces-compute
.. [6] https://pmem.io/ndctl/ndctl-create-namespace.html

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release
     - Description
   * - Yoga
     - Introduced

