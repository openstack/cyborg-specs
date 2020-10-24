..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Cyborg NVIDIA GPU Driver Proposal
==================================

This spec proposes to provide the initial design for Cyborg's NVIDIA physical
GPU management driver. Please note that the virtualized GPU is out of scope.
we only support passthrough one pGPU card to one VM directly.

Problem description
===================

This spec will add a NVIDIA GPU driver for Cyborg to manage specific
NVIDIA physical GPU devices.

Use Cases
---------

* As an operator, I would like to use Cyborg agent starts or does resource
  checking periodically, the Cyborg NVIDIA GPU driver should provider
  ``discover()`` function to enumerate the list of the NVIDIA GPU devices,
  and report the details of all available NVIDIA GPU accelerators on the
  host, such as PID(Product id), VID(Vendor id), Device.

* As a user, I would like to boot up a VM with NVIDIA GPU card attached in
  order to accelerate compute ability. Cyborg should be able to manage this
  kind of acceleration resources and assign it to the VM(binding).


Proposed changes
================

In general, the goal is to develop a Cyborg NVIDIA GPU driver that supports
discover interfaces for NVIDIA GPU accelerator framework. The driver should
include the ``discover()`` function. For physical GPU, this function works by
executing ``lspci`` command and reports devices' raw info sample as
following::

  [
    {
    "vendor": "10de",
    "product": "1db6",
    "device": "0000:af:00:0"
    }
  ]

Generate Cyborg specific driver objects and resource provider modeling
for the GPU device. Below is the objects to describe a pGPU devices which
complies with the Cyborg database mode and Placement data model.

::

  Hardware     Driver objects       Placement data model
     |               |                      |
  1 GPU         1 device                    |
     |               |                      |
     |         1 deployable       ---> resource_provider
     |               |            ---> parent resource_provider: compute node
     |               |                      |
  1 pGPU       1 attach_handle    ---> inventories(total:1)

Alternatives
------------

None

Data model impact
-----------------

NVIDIA GPU driver will not touch Data model.
The Cyborg Agent can call NVIDIA GPU driver to update the database
during the discover operations.

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

User can manage NVIDIA GPU cards by Cyborg NVIDIA GPU driver. Such as list
of the NVIDIA GPU devices, report the details of all available NVIDIA GPU
accelerators on the host, binding with NVIDIA GPU and so on.

Performance Impact
------------------

None

Other deployer impact
---------------------

Deployers need to make sure the GPU device hasn't been virtualized. Otherwise,
we can't use it as a pGPU.

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

* Implement NVIDIA GPU driver in Cyborg
* Add related test cases.


Dependencies
============

None

Testing
========

* Unit tests will be added to test this driver.

Documentation Impact
====================

Document NVIDIA pGPU driver in Cyborg project.
Test report in Cyborg wiki

References
==========

None


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release
     - Description
   * - Train
     - Introduced
