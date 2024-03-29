..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Support attribute API
=====================

This spec adds a new group of APIs to manage the lifecycle of accelerator's
attributes.

Problem description
===================

Attribute is designed for describing customized information of an accelerator.
Now they are generated by drivers, users can not add/delete/update them, it's
not applicable to our scenarios now.

Use Cases
---------

An admin or operator needs a group of APIs to manage his accelerator's
attributes.
Here are some useful scenarios:

* For a NIC accelerator, we need to add a phys_net attribute, it's should be
  created by deployer or other components.
* For some Function Volatile Accelerators, we can create the Function name as
  an attribute.
* Also for some information, such as Function_UUID is machine readable.

Proposed change
===============
None

Alternatives
------------

None

Data model impact
-----------------

* Add attribute object to deployable object.

REST API impact
---------------

URL: ``/v2/deployable/{uuid}/attribute``

METHOD: ``GET``

    List all attributes of specified deployable.

Normal response code (200) and body::

 {
     "attributes":[{
         "key":"key1",
         "value":"value1",
         "uuid":"uuid1"
         }
     ]
 }

Error response code and body:

* 401 (Unauthorized): Unauthorized

* 403 (Forbidden): RBAC check failed

* No response body


URL: ``/v2/deployable/{uuid}/attribute/{uuid_or_key}``

METHOD: ``GET``

    GET specified attribute of specified deployable.

Query Parameters: None

Normal response code (200) and body::

 {
     "attribute":
     {
         "key":"key1",
         "value":"value1",
         "uuid":"uuid1",
         "created_at":"2020-05-28T03:03:20",
         "updated_at":"2020-05-28T03:03:20"
     }
 }

Error response code and body:

* 401 (Unauthorized): Unauthorized

* 403 (Forbidden): RBAC check failed

* 404 (NotFound): No deployable of that UUID or no attribute of that UUID
  exists

* No response body


URL: ``/v2/deployable/{uuid}/attribute``

METHOD: ``POST``

    Create one or more deployable attribute(s).

Request body::

 [
   {
     "key": "key1",
     "value": "value1"
  },
   {
     "key": "key2",
     "value": "value2"
   },
  ...
 ]

Normal response code and body:

* 204 (No content)

* No response body

Error response code:

* 401 (Unauthorized): Unauthorized

* 403 (Forbidden): RBAC check failed

* 409 (Conflict): Bad input or key is not unique

Error response body::

 {"error": "error-string"}


URL: ``/v2/deployable/{uuid}/attribute/{uuid_or_key}``

METHOD: ``DELETE``

    Delete an exist deployable attribute.

Query Parameters: None

Normal response code and body:

* 204 (No content)

* No response body

Error response code:

* 401 (Unauthorized): Unauthorized

* 403 (Forbidden): RBAC check failed

* 404 (NotFound): No deployable of that UUID or no attribute of that UUID
  exists

Error response body::

 {"error": "error-string"}


URL: ``/v2/deployable/{uuid}/attribute``

METHOD: ``DELETE``

    Delete all attributes of a deployable.

Query Parameters: None

Normal response code and body:

* 204 (No content)

* No response body

Error response code:

* 401 (Unauthorized): Unauthorized

* 403 (Forbidden): RBAC check failed

Error response body::

 {"error": "error-string"}


URL: ``/v2/deployable/{uuid}/attribute/{uuid_or_key}``

METHOD: ``PUT``

    Update an exist deployable attribute.

Query Parameters: None

Request body (Value of deployable attribute)::

 {"value": "value1"}

Normal response code and body:

* 204 (No content)

* No response body

Error response code and body:

* 401 (Unauthorized): Unauthorized

* 403 (Forbidden): RBAC check failed

* 404 (NotFound): No deployable of that UUID or no attribute of that UUID
  exists

Error response body::

 {"error": "error-string"}

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------
* Change Cyborg Attribute table.


Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
* If the user want to use these feature, they should upgrade their Cyborg
* project to latest to support these changes.

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  hejunli

Work Items
----------

* Change Cyborg REST APIs.
* Change Cyborg Attribute table.
* Change Cyborg deployable object.
* Change cyborgclient to support Attribute management action.
* Add related tests.

Dependencies
============
None

Testing
=======
Appropriate unit and functional tests should be added.

Documentation Impact
====================
* Need a documentation to record microversion history.
* Need a documentaiton to explain api usage.

References
==========
None

History
=======
.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Antelope
     - Introduced
