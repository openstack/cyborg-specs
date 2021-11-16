..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Add disable/enable device API
=============================

https://blueprints.launchpad.net/openstack-cyborg/+spec/disable-enable-device

Nowadays, Cyborg discovers the device on compute node by each driver. All
devices matching the spec of driver are discovered and reported to the
Placement service as an accelerator resources.
This spec proposes a set of new APIs which allow admin users to
disable/enable a device.


Problem description
===================

Cyborg maintains a configuration file to configure the enabled drivers. Once
the driver is enabled, the agent will discover all devices whose vendor ID,
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
We propose to add new API in order to enable/disable a device. If the device is
disabled, Cyborg will report this device as a reserved resource to Placement,
so that Nova can not schedule to this device. On the contrary, if the device is
enabled, the device should become available and the 'reserved' field in
Placement shoule be set to 0.
* Since the API layer is modified, a new microversion should be introduced.
* It also need a new field "is_maintaining" in Device object and data model to
indicate whether the device is disbaled. If one device is disabled, the
"is_maintaining" field should be set to "True", and if the device is enabled,
the field should be set to "False". The default value should be "False".
* Cyborg need call Placement API to update the "reserved" field for the
device in this API.
* Add "is_maintaining" field's value check during conductor's periodic report.

Alternatives
------------
None

Data model impact
-----------------
A new column `is_maintaining` should be added in Device's data model.


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
        "devices": [
            {
                "uuid": "d2446439-0142-40b7-9eee-82d855f453d9",
                "type": "FPGA",
                "vendor": "0xABCD",
                "model": "miss model info",
                "std_board_info": "{"device_id": "0xabcd", "class": "Fake class"}",
                "vendor_board_info": "fake_vendor_info",
                "hostname": "devstack01",
                "links": [
                    {
                        "href": "http://172.23.97.140/accelerator/v2/devices/d2446439-0142-40b7-9eee-82d855f453d9",
                        "rel": "self"
                    }
                ],
                "created_at": "2021-11-03T08:48:43+00:00",
                "updated_at": null
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
        "uuid": "d2446439-0142-40b7-9eee-82d855f453d9",
        "type": "FPGA",
        "vendor": "0xABCD",
        "model": "miss model info",
        "std_board_info": "{"device_id": "0xabcd", "class": "Fake class"}",
        "vendor_board_info": "fake_vendor_info",
        "hostname": "devstack01",
        "links": [
            {
                "href": "http://172.23.97.140/accelerator/v2/devices/d2446439-0142-40b7-9eee-82d855f453d9",
                "rel": "self"
            }
        ],
        "created_at": "2021-11-03T08:48:43+00:00",
        "updated_at": null
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
* Add new column `is_maintaining` for device table.
* Add disable/enable API in DeviceController.
* Update the RP `reserved` field according to the operation. For `disable`
  oparation, the `reserved` field need be set by the same value as the
  `total` field, and for `enable` operation, the `reserved` field will be set
  to zero.
* Update GET/LIST device API with `is_maintaining` field added in returned
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
   * - Yoga
     - Reproposed
