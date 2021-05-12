..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Add disable/enable device API
=============================

https://blueprints.launchpad.net/openstack-cyborg/+spec/disable-enable-device

Nowadays, Cyborg discover the device on compute node by each driver. All
devices matching the spec of driver are discovered and reported to the
Placement service as an accelerator resources.
This spec proposes a set of new APIs which allow admin users to
disable/enable a device.


Problem description
===================

Cyborg maintains a configuration files to configure the enabled drivers. Once
the driver is enabled, the driver will discover all devices whose vendor ID,
device ID match the driver's requirement. If admin user do not want all devices
to be used by virtual machine, there is no way to disable a device currently.


Use Cases
---------
* Alice is an admin user, she wants some FPGAs to be reserved for its own use
  and not allow them to be allocated to a VM at the time. For example, she
  wants to program the FPGA device and use it as the OVS agent running on
  the host.

Proposed change
===============
* Since the API layer is modified, a new microversion should be introduced.
* It also need a new field in Device object and data model to indicate the
  status of a device. If one device is disabled, the status should be set
  to "maintaining", and if the device is enabled, the status should be set to
  "enabled". The default value should be "enabled".
* Cyborg need call Placement API to update the "reserved" field for the
  device.

Alternatives
------------
None

Data model impact
-----------------
A new column `device_status` should be added in Device's data model.


REST API impact
---------------
A microversion need to be introduced since the Device API changed.

List Device API
^^^^^^^^^^^^^^^
* Return a device list
  URL: ``/devices``
  METHOD: ``GET``
  Return: 200

.. code-block::

    {
        "devices":
        [
            {
                "uuid": "359c0990-0258-44fd-8b05-fc510ac3d022",
                "type": "FPGA",
                "vendor": "0xABCD",
                "model": "miss model info",
                "std_board_info": "{'device_id': '0xabcd', 'class': 'Fake class'}",
                "vendor_board_info": "fake_vendor_info",
                "hostname": "computenode",
                "device_status": "Enabled"
                "created_at": "2020-03-13T02:26:31+00:00",
                "updated_at": null,
                "links":
                [
                    {
                        "href": "http://localhost/accelerator/v2/devices/359c0990-0258-44fd-8b05-fc510ac3d022",
                        "rel": "self"
                    }
                ]
            }
        ]
    }

Get Device API
^^^^^^^^^^^^^^
* Get a device by uuid and return the details
  URL: ``/devices/{uuid}``
  METHOD: ``GET``
  Return: 200

.. code-block::

   {
       "uuid": "29e23349-12ee-4978-963c-11484a4ae601",
       "parent_id": null,
       "root_id": null,
       "name": "computenode_FakeDevice",
       "num_accelerators": 16,
       "device_id": 1,
       "attributes_list": "[{'traits1': 'CUSTOM_FAKE_DEVICE'}, {'rc': 'FPGA'}]",
       "rp_uuid": "853f07a6-19de-3dd6-b9f6-6c782daa3f7b",
       "driver_name": "fake",
       "device_status": "Enabled"
       "bitstream_id": null,
       "created_at": "2020-03-13T02:27:35+00:00",
       "updated_at": "2020-03-13T02:27:36+00:00",
       "links":
       [
          {
             "href": "http://localhost/accelerator/v2/deployables/29e23349-12ee-4978-963c-11484a4ae601",
             "rel": "self"
          },
          {
             "href": "http://localhost/accelerator/deployables/29e23349-12ee-4978-963c-11484a4ae601",
             "rel": "bookmark"
          }
       ]
   }



Disable Device API
^^^^^^^^^^^^^^^^^^
* Disable a device
  URL: ``/devices/disable/{device_uuid}``
  METHOD: ``POST``
  Return: 200
  Error Code: 404(the device is not found),403(the role is not admin)

Enable Device API
^^^^^^^^^^^^^^^^^
* Enable a device
  URL: ``/devices/enable/{device_uuid}``
  METHOD: ``POST``
  Return: 200
  Error Code: 404(the device is not found),403(the role is not admin)

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
The deployer need update Cyborg to the microversion which supports
disable/enable API. Otherwise the disable/enable API will be rejected.

Developer impact
----------------
None


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  Xinran Wang(xin-ran.wang@intel.com)

Work Items
----------
* Add new column `device_status` for device table.
* Add disable/enable API in DeviceController.
* Update the RP `reserved` field according to the operation. For `disable`
  oparation, the `reserved` field need be set by the same value as the
  `total` field, and for `enable` operation, the `reserved` field will be set
  to zero.
* Update GET/LIST device API with `device_status` field added in returned
  value.
* Add disable/enable operation in cyborgclient.
* Add unit tests.

Dependencies
============
None


Testing
=======
Need add unit test, and tempest test if needed.


Documentation Impact
====================
Need add related docs.

References
==========
None


History
=======

Optional section intended to be used each time the spec is updated to describe
new design, API or any database schema updated. Useful to let reader understand
what's happened along the time.

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Xena
     - Introduced
