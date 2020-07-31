..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Cyborg Intel® QAT Driver Proposal
=================================

This spec proposes to provide the initial design for Cyborg's Intel® QAT
driver.

Problem description
===================

Intel® QuickAssist Technology(Intel® QAT) provides security(encryption) HW
acceleration and compression HW acceleration. It improves performance across
applications and platforms. That includes symmetric encryption and
authentication, asymmetric encryption, digital signatures, RSA, DH, and ECC,
and lossless data compression. Hence, using Intel® QAT for application
acceleration in cloud has been becoming desirable.

Intel® QAT card also support SRIOV, which means admin can virtualize it into
serveral VFs and assign VFs to the VM. One Intel® QAT Card usually has 3 or 6
PFs which correspond to "Device" notion in Cyborg, and each of them can be
virtualize into 8 or 16 VFs. It depends on diffirent devices.

This spec will add a Intel® QAT driver for Cyborg to manage specific Intel® QAT
devices.

Use Cases
---------
* When user want to boot up a VM with Intel® QAT card (PF or VF) attached in
  order to accelerate TLS workload. Cyborg should be able to manage this kind
  of acceleration resources and to assign it to the VM(binding).

Proposed changes
================

In general, the goal is to develop a Cyborg Intel® QAT driver that supports
discover() interfaces, this driver is reponsible to encapsule the device
information to Cyborg's unified data model, such as Device, Deployale,
AttachHandle and so on.

The driver should include discover functions which will be called by agent
periodically.


Image Format
----------------------------

Alternatives
------------

None

Data model impact
-----------------

None


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

Deployers should install the specific Intel® QAT software package that the
driver depends on.

Please see ref [1]_ for details.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Xinran Wang

Work Items
----------

* Implement Intel® QAT driver in Cyborg
* Add related test cases.


Dependencies
============

None

Testing
========

* Unit tests will be added to test this driver.

Documentation Impact
====================

Document Intel® QAT driver in Cyborg project.

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

References
==========
.. [1] https://01.org/intel-quickassist-technology
