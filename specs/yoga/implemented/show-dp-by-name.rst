..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Show device_profile by name
===========================

https://blueprints.launchpad.net/openstack-cyborg/+spec/show-device-profile-with-name

Nowadays, A device_profile can only be got by it's UUID, but the field 'name'
is not supported. This spec will support getting a device profile by name.


Problem description
===================

When using `openstack accelerator device profile show` command to getting
one device profile, the '<uuid>' of the device profile is only accepted.
The device profile has 'name' field that can display in humanable manner.
The end-users prefer to use name instead of hard-to-remember UUID. By this
way you can get the device profile details directly instead of searching
its UUID first.

Use Cases
---------

* As an administrator or end user, i want to get the details of the device
  profile by its name.

Proposed change
===============

* Change the `Get One Device Profile` API to accept 'name' as parameter.
* Introduce a microverison for `Get One Device Profile` API
* Then, change the python-cyborgclient with
  `openstack accelerator device profile show`, to support getting one device
  profile details by name.

Alternatives
------------
None


Data model impact
-----------------
None


REST API impact
---------------
A microversion need to be introduced since the Device Profile API changed.

Get Device Profile API
^^^^^^^^^^^^^^^^^^^^^^
GET /v2/device_profiles/{device_profile_name_or_uuid}
The 'device_profile_name_or_uuid' should be uuid or name, the response has
not changed.

.. code-block::

    {
       "device_profile":{
          "name":"fpga-dp1",
          "uuid":"5518a925-1c2c-49a2-a8bf-0927d9456f3e",
          "description": "",
          "groups":[
             {
                "trait:CUSTOM_CHENKE_TRAITS":"required",
                "resources:FPGA":"1",
                "accel:bitstream_id":"d5ca2f11-3108-4426-a11c-a959987565df"
             }
          ],
          "created_at": "2020-03-09 11:26:05+00:00",
          "updated_at": null,
          "links":[
             {
                "href":"http://192.168.32.217/accelerator/v2/device_profiles/5518a925-1c2c-49a2-a8bf-0927d9456f3e",
                "rel":"self"
             }
          ]
       }
    }

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
  Eric Xie(eric_xiett@163.com)

Work Items
----------
* Change the device profile get api to support 'name'
* Extend 'python-cyborgclient' to support 'name'
* Add related unit tests


Dependencies
============
None


Testing
=======
Need add unit test.


Documentation Impact
====================
Use 'device_profile_name_or_uuid' instead of 'device_profile_uuid' and change
its description in 'api-ref/source/device_profile.inc'.

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
   * - Yoga
     - Introduced

