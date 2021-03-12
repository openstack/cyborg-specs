..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Cyborg Inspur NVMe SSD Driver Proposal
======================================

This spec proposes to provide the initial design for Cyborg's Inspur NVMe SSD
driver.

Problem description
===================

Inspur NVMe SSD devices provide the ability to accelerate VM IO rate.
In the OpenStack ecosystem, we don't have any tool to manage this kind of
accelerators. This spec will add an Inspur NVMe SSD driver in Cyborg to
automatically manage Inspur NVMe SSD devices.

The management in the driver scope includes:

Cyborg Inspur NVMe SSD driver can automatically discover the NVMe SSD and
report to database by calling cyborg-conductor.
Cyborg Inspur NVMe SSD driver needs to do the device binding/unbinding to VM.


Use Cases
---------

* As an operator, I would like to use Cyborg to manage Inspur NVMe SSD
  devices.

* As a developer, I would like to use Cyborg agent to start or to do resource
  checking periodically, the Cyborg Inspur NVMe SSD driver should provide
  ``discover()`` function to enumerate the list of the Inspur NVMe SSD
  devices, and report the details of all available NVMe SSD accelerators on
  the host, such as PID(Product ID), VID(Vendor ID), Device ID.

* As a user, I would like to boot up a VM with Inspur NVMe SSD card attached
  in order to accelerate IO rate. Cyborg should be able to manage this kind
  of acceleration resources and assign it to the VM(binding).


Proposed changes
================

1. Collect raw info of Inspur NVMe SSD devices from compute node by "lspci"
command and grep ``Non-Volatile memory controller`` related keyword and
then grep Inspur vendor ID.

2. Parsing details from each record including ``vendor_id``, ``product_id``
and ``pci_address``.

3. Generate Cyborg specific driver objects and resource provider modeling for
the Inspur NVMe SSD devices. Below is the objects to describe a Inspur NVMe
SSD device which complies with the Cyborg database mode and Placement data
model.

::

  Hardware     Driver objects       Placement data model
     |               |                      |
  1 SSD         1 device                    |
     |               |                      |
     |         1 deployable       ---> resource_provider
     |               |            ---> parent resource_provider: compute node
     |               |                      |
  1 SSD        1 attach_handle    ---> inventories(total:1)

4. We report ``CUSTOM_SSD`` as  resource_class, ``CUSTOM_SSD_INSPUR``
and ``CUSTOM_SSD_PRODUCT_ID_1003`` as traits to Placement.


Alternatives
------------

None

Data model impact
-----------------

Add ``SSD`` type to the column device_type in table devices.

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

User can manage Inspur NVMe SSD cards by Cyborg Inspur NVMe SSD driver.
Such as list the Inspur NVMe SSD devices, report the details of all
available Inspur NVMe SSD accelerators on the host, binding an Inspur
NVMe SSD to VM.

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
  Wenping Song

Work Items
----------

* Implement Inspur NVMe SSD driver in Cyborg
* Add related test cases.
* Add test report to wiki page.
* Update doc page and release note.

Dependencies
============

None

Testing
========

* Unit tests will be added to test this driver.
* Add test result in Cyborg Wiki which is required by the Cyborg community.

Documentation Impact
====================

Document Inspur NVMe SSD driver in Cyborg project.

References
==========

None

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release
     - Description
   * - Wallaby
     - Introduced
