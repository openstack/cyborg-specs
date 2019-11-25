..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Cyborg API Version 2
====================

Problem description
===================
A new device model and database schema was introduced in Stein release. That
has obsoleted the accelerator objects, changed the definition of deployable
objects, and introduced the concepts of devices. That has made some old APIs
obsolete and raised the need for new APIs.

Secondly, the proposed scheme to integrate with Nova involves new objects
like device profiles and accelerator requests (ARQs). They require new APIs as
well. Of these, the device profiles API is covered in [#devprof-spec]_.

This spec will specify what v1 APIs will be dropped in v2, and what new APIs
need to be introduced.

Use Cases
---------

* As a user, I want to create VMs with attached accelerators, delete
  them and perform other instance operations. (Live migration is excluded.)

  * Cyborg should provide APIs for device profiles and ARQs, so as to
    support end-to-end workflows with Nova for VM creation,
    deletion and other operations.

* As an operator, I want to query and manage the inventory of accelerators
  and devices.

  * Cyborg should provide an API to query the list of devices based on
    various attributes.

* As a user with appropriate authorization, or as an operator, I want to
  initiate programming of specific devices, either with FPGA bitstreams,
  or with FPGA shell logic or device firmware.

Proposed changes
================

Microversions
-------------
Cyborg should adopt microversions to enable incremental backwards-compatible
changes to APIs. This is important because Cyborg is a rapidly evolving
project which needs to support new devices and features over time. For Train
release, only one microversion will be supported, i.e., 2.0.

The changes should be compatible with Microversion Specification
[#version-disc]_. This means that the output of the API::

  GET /accelerator

must change as specified in Section `Changed APIs`_. Also, a new API::

  GET /accelerator/v2

is introduced conformant to that specification, as documented in Section
`New APIs`_.

The API clients can use the following header to indicate which microversion
to use::

  OpenStack-API-Version: accelerator <microversion>

If they do not, the default microversion of 2.0 will be assumed in Train.

API Changes
-----------

The following v1 URLs will not be supported in v2, because the accelerator
object does not exist anymore and the deployable object is exposed as part of
the device object::

   /v1/accelerators
   /v1/deployables

A new set of APIs based on the URL::

   /v2/device_profiles

is introduced in the Specification for device profiles ([#devprof-spec]_).

This specification introduces another set of APIs based on the URLs::

   /v2/accelerator_requests
   /v2/devices
   /v2/deployables/{uuid}

See Section `New APIs`_ for details.

Alternatives
------------
We could add another API to enable or disable specific devices, or specific
deployables (resource providers). That is left for a future version.

Data model impact
-----------------
The database changes needed for these APIs have mostly been done in Train.
However, we need to add a ``bitstream`` field to ``deployables`` table in the
database, which can serve as a target in the JSON patch for reprogramming.

REST API impact
---------------

Deleted APIs
^^^^^^^^^^^^
Cyborg should drop the APIs for accelerators and deployables in v2. The
deployables, as per the new definition, will be included in the new devices
API.

Changed APIs
^^^^^^^^^^^^

URL: ``/accelerator``

METHOD: ``GET``

ROLE: Admin or any tenant

Current JSON response::

 {
    "description": "Cyborg (previously known as Nomad) is an OpenStack
       project that aims to provide a general purpose management framework for
       acceleration resources (i.e. various types of accelerators such as
       Crypto cards, GPU, FPGA, NVMe/NOF SSDs, ODP, DPDK/SPDK and so on).",
    "name": "OpenStack Cyborg API"
 }

Proposed JSON response::

 {
    "versions": [
        {
            "id": "v1",
            "links": [
                {
                    "href": "http://<ip>/accelerator/v1",
                    "rel": "self"
                }
            ],
            "version": "1.0",
            "status": "DEPRECATED"
        },
        {
            "id": "v2.0",
            "links": [
                {
                    "href": "http://<ip>/accelerator/v2",
                    "rel": "self"
                }
            ],
            "min_version": "2.0",
            "max_version": "2.0",
            "version": "2.0",
            "status": "CURRENT"
        },
    ]
 }

New APIs
^^^^^^^^

URL: ``/accelerator/v2``

METHOD: ``GET``

ROLE: Admin or any tenant

Proposed JSON response::

 {
    "id": "v2.0",
    "links": [
        {
            "href": "http://<ip>/accelerator/v2",
            "rel": "self"
        }
    ],
    "min_version": "2.0",
    "max_version": "2.0",
    "version": "2.0",
    "status": "CURRENT"
 }

Device profile APIs are covered in [#devprof-spec]_. A device profile
may request one or more accelerator resources. The request for each
accelerator resource is encapsulated in an Accelerator Request (ARQ) object.
The following are the REST APIs for ARQs.

URL: ``/accelerator/v2/accelerator_requests``

METHOD: ``GET``

ROLE: Admin or the tenant who owns the specified instance.

Query Parameters:

* instance: UUID of the instance whose ARQs are requested.
* bind_state: Bind state of ARQs. Only supported value is
    'resolved', which means the ARQ is either bound successfully or
    failed to bind. Other states like 'bound' may be supported in the
    future.

Proposed JSON response::

 {
    "arqs": [
       <arq_obj>,
       ...
    ]
 }

URL: ``/accelerator/v2/accelerator_requests``

METHOD: ``POST``

ROLE: Admin or any tenant with RBAC authorization.

Proposed JSON request::

 {
   "device_profile_name": <string>
 }

Action: Create ARQs for the specified device profile. If the device profile
specifies N accelerator resources across all its request groups, N ARQs will
get created in unbound state.

Proposed JSON response::

 {
    "arqs": [
       <arq_obj>,
       ...
    ]
 }

URL: ``/accelerator/v2/accelerator_requests``

METHOD: ``PATCH``

ROLE: Admin or the tenant who owns the specified instance.

Proposed JSON request (in RFC 6902 format)::

 For binding:
 {
   "$arq_uuid": [
      { "path": "/hostname", "op": "add", "value": <string> },
      { "path": "/instance_uuid",  "op": "add", "value": <uuid> },
      { "path": "/device_rp_uuid", "op": "add", "value": <uuid> }
   ],
   "$arq_uuid": [...],
   ...
 }

 For unbinding:
 {
   "$arq_uuid": [
      { "path": "/hostname", "op": "remove" },
      { "path": "/instance_uuid",  "op": "remove" },
      { "path": "/device_rp_uuid", "op": "remove" }
   ],
   "$arq_uuid": [...],
   ...
 }

Action: Bind or unbind

Proposed JSON response: None

URL: ``/accelerator/v2/accelerator_requests``

METHOD: ``DELETE``

ROLE: Admin or the tenant who owns the specified ARQs.

Query Parameters (required):

* arqs: List of one or more comma-separated ARQ UUIDs.

Proposed JSON response: None

URL: ``/accelerator/v2/devices``

METHOD: ``GET``

Query Parameters:

Proposed JSON response::

  {
    "devices": [
       <device_obj>,
       ...
    ]
  }

URL: ``/accelerator/v2/devices/{uuid}``

METHOD: ``PATCH``

ROLE: Admin or any tenant with RBAC authorization.

Proposed JSON request (in RFC 6902 format)::

 {
   [
      { "path": "/firmware", "op": "add", "value": <img_uuid> },
   ]
 }

Action: Update the firmware or shell image (FPGA bitstream) for the
  specified device. The request allows a lst of path specifiers for
  future extensibility.

Proposed JSON response: None.

URL: ``/accelerator/v2/deployables/{uuid}``

METHOD: ``PATCH``

ROLE: Admin or any tenant with RBAC authorization.

Proposed JSON request (in RFC 6902 format)::

 {
   [
      { "path": "/bitstream", "op": "add", "value": <img_uuid> },
   ]
 }

Action: Update the FPGA bitstream for the specified deployable. The
  request allows a lst of path specifiers for future extensibility.

Proposed JSON response: None.

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

TBD

Work Items
----------

Dependencies
============

None

Testing
=======

Unit tests and functional tests need to be written.

Documentation Impact
====================

Cyborg API documentation needs to be updated.

References
==========
.. [#version-disc] `Microversions and Version Discovery
   <http://specs.openstack.org/openstack/api-wg/guidelines/microversion_specification.html>`_

.. [#devprof-spec] `Specification for device profiles
   <http://specs.openstack.org/openstack/cyborg-specs/specs/train/approved/device-profiles.html>`_

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Train
     - Introduced
   * - Ussuri
     - Re-proposed
