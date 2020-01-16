..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
Device Profiles
===============

https://blueprints.launchpad.net/openstack-cyborg/+spec/device-profiles

This spec introduces the notion of device profiles, explains why they are
needed and describes how they are used.

Problem description
===================
The request to create or update an instance can specify various resources,
spanning compute, networking, storage and accelerators. While all of them can
be specified using the flavor syntax including extra specs, that results in
two significant problems: lack of separation of concerns and flavor explosion.

It is desirable to separate the requirements for different resources into
modular pieces and then compose them into the flavor. This is done for
networking with port binding profiles. It is desirable to propose a
similar model for accelerators and hardware resources.

Requests for devices can vary hugely in the type, number and combination of
devices requested. For example, a web server workload may require offloads for
video transcoding, SSL and log file compression. Offloads may involve
different types of devices, including programmable hardware (GPUs, FPGAs,
etc.), fixed function hardware (QuickAssist, TPUs, etc.) and special-purpose
hardware such as High precision Time Synchronization cards. If each
combination of hardware resources were to be expressed as parts of flavors,
the number of flavors will increase rapidly. That is a problem because
operators often use flavors for usage and billing.

So, we wish to separate the specification of accelerator resources in an
instance request into a new entity called a device profile.

Use Cases
---------
* The operator should be able to create, publish, update and delete device
  profiles.
* The operator should be able to compose a device profile into a flavor, so
  that the end user can request an instance using the flavor alone.
* The end user should be able to request an instance to be created or updated,
  using a flavor and a device profile separately.
* The operator should be able to enable or disable specific device profiles.
* On an upgrade, device profiles from a certain number of past releases
  should continue to be usable by operators.

The ability to update or enable/disable device profiles may be addressed in
the future after operator feedback.

The operator may use device profile usage to bill end users.
That is outside the scope of this spec.

Proposed change
===============

Overview
--------
A device profile is a named set of the user requirements for one or more
accelerators. It can be viewed as a flavor for devices. Broadly it includes
two things: the desired amounts of specific resource classes and the
requirements that the resource provider(s) must satisfy. While the resource
classes are the same as those known to Placement, some requirements would
correspond to Placement traits and others to properties that Cyborg alone
knows about.

The device profile format is a Python dictionary with syntax similar to
granular resource requests for Nova flavors. Here is an example::

  {
    'name': 'my_device_profile',
    'description': 'Image classification',
    'groups': [
      { 'resources:CUSTOM_ACCELERATOR_FPGA=1',
        'trait:CUSTOM_FPGA_INTEL_ARRIA10=required',
        'accel:function_id=3AFB'
      },
      { 'resources:CUSTOM_ACCELERATOR_GPU=2',
        'trait:CUSTOM_GPU_MODEL_NAME=required',
        'accel:video_ram=2GB'
      },
    ]
    'uuid': '8411555e-3cbf-47de-988c-b5e30bbfa035' # auto-generated
  }

The fields and syntax are explained in the Section
`Device Profile structure`_.

The device profiles can be queried, created and deleted from the Cyborg API
(see Section `REST API impact`_) and from the command line (see Section
`Cyborg client command line`_). The ability to update them or enable/disable
them may be taken up after operator feedback.

Only the admin can create, update and delete device profiles. In the future,
the admin may be able to publish them in Horizon. The operator and end user
both can list device profiles from the command line today and potentially view
them in Horizon in the future.

Device profile structure
------------------------
The salient points of the structure of the device profile are noted in this
section.

Each field uses a format similar to extra specs; it specifies either a
resource, a trait or a Cyborg-specific property. The properties are
specified using the ``accel:`` prefix and are of the form
``accel:<key>=<value>``. Both the key and the value are strings, which may
contain alphabets (lower or upper case), digits, hyphens and underscores. No
other characters are allowed. In the future, modifiers may be introduced after
the ``accel:`` prefix. For example, to indicate that some resource is
preferred but not required, a possible syntax could be
``accel:preferred:resource_name=amount``. The valid keys in this release are
listed in Section `Valid accel keys`_.

The requests are grouped in the same way as granular resource syntax. The
ordering among the groups is not significant. However, no group policy is
allowed here because there may be one in the flavor.

Unlike the granular request syntax, the operator need not number the groups.
Cyborg will do the numbering.

The order of request groups within a device profile has no significance for
scheduling or any system behavior. However, it is preserved by Cyborg. So, if
a client fetches the same device profile twice (with no changes to the device
profile in between), the group order will remain the same. This fact is
important for Nova Cyborg interaction [#nova-cyborg]_.

Lower case and hyphens are allowed: The names of custom RCs and traits can
only include upper case letters, digits and underscores. Cyborg will translate
lower case to upper case and hyphens to underscores.

Usage
-----
The user request for an instance spawn needs to include both a flavor and a
device profile. However, this needs a change in the Nova API for instance
creation to include the device profile name. To avoid the need for such a
change, initially, the operator shall explicitly fold the device profile into
the flavor as below::

  openstack flavor set --property 'accel:device_profile=<profile_name>' flavor

Thus the end user can launch requests with only the flavor just as today.

Later the Nova API will be enhanced to take a device profile name explicitly,
removing the need for operator folding and allowing end users to launch
requests with separate flavor and device profile names. The
device-profile-in-flavor form will continue to be supported even after this
change.

In either case, only one device profile is allowed per user request.

During the instance creation flow, Nova queries Cyborg with the device profile
name and gets the device profile request groups, which are then presented to
Placement along with the request groups from flavor and other sources. This is
the case regardless of whether the device profile was folded into the flavor
or not.

Cyborg client command line
--------------------------
The command line syntax for device profiles is proposed in this section. ::

  openstack accelerator device profile create <name> <json>
  openstack accelerator device profile delete <name>
  openstack accelerator device profile list
  openstack accelerator device profile show <name>

Other commands can be taken up in the future after operator feedback.

Further details will be addressed in a future spec.

Versioning and upgrades
-----------------------
The microversion of the Cyborg APIs used to create/update device profiles also
controls the device profile version. There is no separate version field in the
device profile itself.

A custom trait in one release may become a standard trait in the next one. A
cyborg-specific property in one release may become a custom trait or a
standard trait in the next one. This would invalidate the device profiles
after upgrade.

This must be handled by migrating the device profiles to the new format during
the upgrade.

Valid accel keys
----------------
The Cyborg-specific properties are expressed in the device profile using the
form ``accel:<key>=<value>``, as noted earlier.  The valid key-value pairs in
this release are as noted below:

.. list-table:: Cyborg Properties
   :header-rows: 1

   * - Key
     - Value Type
     - Semantics
   * - ``bitstream_id``
     - UUID
     - Glance UUID of the bitstream that must be programmed.
       The type of the bitstream, which may influence the tool used for
       programming, is then assumed to be a image property.
   * - ``bitstream_name``
     - String
     - Name of the bitstream in Glance, including the suffix that indicates
       bitstream type.
   * - ``function_id``
     - UUID
     - UUID of the needed function.
   * - ``function_name``
     - String
     - Name of the needed function.
   * - ``attach_target``
     - Enum of strings: 'VM', 'host' or 'none'
     - Indicates whether the accelerator should be attached to a VM (default),
       the host (infrastructure offload use case), or not attached to anything
       (indirect access use case). See sections 'Use cases' and 'Indirect
       accelerator access' in [#nova-cyborg]_.

Alternatives
------------

Flavor explosion can be partially addressed by adding metadata to the
instance image that expresses what hardware resources are needed for that
image to work correctly or well. However, that is not a complete solution.

Data model impact
-----------------

Device profiles need to be persisted in Cyborg database. Each device profile
should have a UUID since they may be renamed.

REST API impact
---------------

::

 URL: /v2/device_profiles
 METHOD: GET
 Query Parameters: name=name1,name2,...
 Normal response code and body:
    200
    { 'device_profiles': [ <dev-prof>, ... ] }
 Error response code and body:
    401 (Unauthorized): RBAC check failed
    422 (Unprocessable): No device profiles exist
    No response body
 Note:
    List all device profiles or the set of named device profiles.

 URL: /v2/device_profiles/{uuid}
 METHOD: GET
 Query Parameters: None
 Normal response code and body:
    200
    { 'device_profile': <dev-prof> }
 Error response code and body:
    401 (Unauthorized): RBAC check failed
    422 (Unprocessable): No device profile of that UUID exists
    No response body
 Note:
    List the device profile with the specified name.

 URL: /v2/device_profiles
 METHOD: POST
 Request body: A device profile
    [
        { 'name': <string>,
          'description': <string> # optional
          'groups': [
               {
                   "accel:function_id": "3AFB",
                   "resources:CUSTOM_ACCELERATOR_FPGA": "1",
                   "trait:CUSTOM_FPGA_INTEL_PAC_ARRIA10": "required"
           }
           ],
        },
    ]
 Normal response code and body:
    204 (No content)
    No response body
 Error response code and body:
    401 (Unauthorized): RBAC check failed
    422 (Unprocessable): Bad input or name is not unique
    { 'error': <error-string> }
 Note:
    Create one or more new device profiles. May implement just one in Train.

 URL: /v2/device_profiles?name=string1,...,stringN
 METHOD: DELETE
 Query Parameters: required
 Normal response code and body:
    204 (No content)
    No response body
 Error response code and body:
    401 (Unauthorized): RBAC check failed
    422 (Unprocessable): Bad input
    { 'error': <error-string> }
 Note:
    Delete one or more existing device profiles.

Until device profile updates are supported, there is no need for a PUT or
PATCH.

Security impact
---------------

The APIs and commands proposed here accept and parse user-provided data. To
mitigate the risks, the following measures shall be adopted:

* Only the admin can create, update or delete device profiles. End users
  can only list them or perhaps see them in Horizon.
* The device profile names and fields are restricted to alphabets (lower or
  upper case), digits, underscores, hyphens and the characters ':' and '='.

A device profile is visible to all tenants by default. In the future, we may
provide for device profile visibility only to certain tenants.

Notifications impact
--------------------

None

Other end user impact
---------------------

The end user can list the device profiles using python-cyborgclient. In the
future, she may be able to see them on Horizon.

Performance Impact
------------------

None

Other deployer impact
---------------------

Knobs to enable or disable device profiles may be added in the future.

Developer impact
----------------
Provide upgrade scripts for device profiles if the names of resource classes
or traits change, as noted in `Versioning and upgrades`_.

Implementation
==============

Assignee(s)
-----------

TBD

Work Items
----------

* Create a device profiles table in the Cyborg database.

* Implement REST APIs in the API server and conductor, with validation and
  unit tests.

* Implement the python-cyborgclient CLI.

Dependencies
============

None

Testing
=======

Unit tests and functional tests need to be written.

Documentation Impact
====================

Device profiles should be explained in user docs and in operator docs.

References
==========

.. [#nova-cyborg] `Nova Cyborg interaction specification
   <https://review.openstack.org/#/c/603955/>`_

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Stein
     - Introduced
   * - Train
     - Updated
