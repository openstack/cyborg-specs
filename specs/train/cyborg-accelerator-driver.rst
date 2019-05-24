..
This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
New Cyborg Generic Driver Proposal
==================================

This spec proposes to provide the initial design for Cyborg generic device
driver.

Problem description
===================

Currently, the FPGA and GPU have been supported in Cyborg, but the capability
for the generic accelerator is not supported yet.

In general, these generic devices are the specific accelerators in some specific
scenarios. For example:
1. The AI chips. which can be used for AI training and inference.
2. The security accelerator, which can be used for encryption and decryption.

In order to add the support for these specific accelerators, we propose to
improve the generic driver to manage these devices. We propose to improve the
existing GenericDriver.

Use Cases
---------

- As an AI chip vendor, I want to add the driver in Cyborg, but the existing
  driver type doesn't meet our requirement. We hope the driver can provide the
  firmware upgrade, device configure, device stats query.
- As a security accelerator vendor, I want add the driver in Cyborg with security
  accelerator configure and device stats query

Proposed change
===============
1. The Generic driver change
As the initial version, the new ``GenericDriver`` should include the following
attributes:

   - VENDOR: the vendor name of the driver.
   - TYPE: the type of the driver, such as, "FPGA", "GPU"

and the following interfaces also should be included:

   - discover()
     Discover specific accelerator

   - update(control_path, image_path):
     Upgrade the device firmware with a specific image.
     control_path: the image update control path of device.
     image_path: The image path of the firmware binary, the image would be
     downloaded by Cyborg agent.

   - get_stats():
     Collects device stats. The ``get_stats`` method is used
     to collect information from the device about the device capabilities.
     Such as performance info like temprature, power, volt, packet_count info.
     The response of get_stats should follow the current Cyborg
     device-deploy-accelerator model.

     A real stats look like::

      {
        "device": {
          "device_name": "XXX",                 # Standard properties
          "device_number": "RFD1644N48373",     # Standard properties
          "properties": {                       # Vendor/Custom properties
            "id": "1",
            "temperature": "26",
            "volt": "",
            "packet_count": "",
            "memeory": {
              "model": "DDR4",
              "description": ""
            },
            "board": {

            },
            "flash": {

            }
          }
          "deploy": [
              {
                "accelerator": {
                   "acc_name": "",               # Standard properties
                   "properties": {               # Vendor/Custom properties
                   }
                }
              },
              {
                "accelerator": {
                   "acc_name": "",               # Standard properties
                   "properties": {               # Vendor/Custom properties
                   }
                }
              }
          ]
        }
      }

This ``GenericDriver`` should not be used directly, after adding this
``GenericDriver``, for every new device, we need to introduce a new driver
that inherits from this ``GenericDriver``.


2. Existing driver change
We should also improve FPGA and GPU driver to inherit from this driver and
implements the base driver interface.

* The generic FPGA driver interface:
The ``discover``, ``update``, ``get_accelerator_stats`` should be implemented
in the FPGA driver. And the below interface is the FPGA specifc interface:

   - program(controlpath, image_path)
     Program the FPGA with the provided bitstream image.

* The generic GPU driver interface:
The ``discover``, ``update``, ``get_accelerator_stats`` should be implemented
in the GPU driver.

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
None

Developer impact
----------------
None


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  Yikun Jiang <yikunkero@gmail.com>
  Sundar Nadathur <sundar.nadathur@intel.com>
  wangzhh <wangzh21@lenovo.com>


Work Items
----------
* Improve the generic driver for generic device
* Improve the existing FPGA driver
* Improve the existing GPU driver


Dependencies
============

Testing
=======

Documentation Impact
====================
None

References
==========
None

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Train
     - Introduced
