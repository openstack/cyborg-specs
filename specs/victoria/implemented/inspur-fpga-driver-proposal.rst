..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Cyborg Inspur FPGA Driver Proposal
==================================

This spec proposes to provide the initial design for Cyborg's Inspur FPGA
driver.

Problem description
===================

This spec will add an Inspur FPGA driver for Cyborg to manage specific
Inspur FPGA devices.

Use Cases
---------

* When Cyborg agent starts or does resource checking periodically, the Cyborg
  Inspur FPGA driver should provider discover() function to enumerate the
  list of the Inspur FPGA devices, and report the details of all available
  Inspur FPGA accelerators on the host, such as PID(Product id),
  VID(Vendor id), Device and BSP(Inspur FPGA program environment).

* When user uses empty Inspur FPGA  as their accelerators, Cyborg agent will
  call driver's program() interface by provide BSP to the driver.


Proposed changes
================

In general, the goal is to develop a Cyborg Inspur FPGA driver that supports
discover/program interfaces for Inspur FPGA accelerator framework.

The driver should include the follow functions:
1. discover()
driver reports devices' raw info sample as following::

  [{
    "vendor": "0x1db4",
    "product": "a115",
    "device": "0000:af:00:0",
    "bsp": {"env": "/var/lib/inspur-fpga",
            "cmd": "aocl"}
  }]

2. program(bsp)
   program by bsp.
   bsp: the environment of accelerator device.

Image Format
----------------------------

Alternatives
------------

None

Data model impact
-----------------

Inspur FPGA driver will not touch Data model.
The Cyborg Agent can call Inspur FPGA driver to update the database
during the discover/program operations.

REST API impact
---------------

The related Inspur FPGA accelerator APIs is out of scope for this spec.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

User can manage Inspur FPGA card by Cyborg Inspur FPGA driver. Such as list
of the Inspur FPGA devices, report the details of all available Inspur
FPGA accelerators on the host, program with Inspur FPGA and so on.

Performance Impact
------------------

None

Other deployer impact
---------------------

Deployers should install the specific Inspur FPGA management stack that the
driver depends on.

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

* Implement Inspur FPGA driver in Cyborg
* Add related test cases.


Dependencies
============

None

Testing
========

* Unit tests will be added to test this driver.

Documentation Impact
====================

Document Inspur FPGA driver in Cyborg project.

References
==========

None


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release
     - Description
   * - Victoria
     - Introduced
