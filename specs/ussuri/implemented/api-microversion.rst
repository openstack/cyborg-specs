..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
API Microversions for Cyborg
============================

Problem description
===================

Cyborg has deprecated v1.0 API since Train release and use v2.0 API as current
API version. As Cyborg evolves and obtains new features which are also
represented in API changes, we should support microversion of Cyborg which will
not break current infrastructure.

Use Cases
---------

Allows developers to modify the Cyborg API in backwards compatible way and
signal to users of the API dynamically that the change is available without
having to create a new API extension.

Allows developers to modify the Cyborg API in a non backwards compatible way
while still supporting the old behaviour. Users of the REST API are able to
decide if they want the Cyborg API to behave in the new or old manner on a per
request basis. Deployers are able to make new backwards incompatible features
available without removing support for prior behaviour as long as there is
support to do this by developers.

Users of the REST API are able to decide, on a per request basis, which version
of the API they want to use (assuming the deployer supports the version they
want). This means that a deployer who does not upgrade will not break
compatibility, and clients that do not upgrade will also remain compatible.

Proposed change
===============

Cyborg API should receive the API version from client that represents the
list resources and their GET/POST/PATCH attributes it can work with.

Cyborg will use a framework we will call "API Microversions" for allowing
changes to the API while preserving backward compatibility. The basic idea is
that a user has to explicitly ask for their request to be treated with a
particular version of the API. So breaking changes can be added to the API
without breaking users who don't specifically ask for it. This is done with an
HTTP header "OpenStack-API-Version: accelerator 2.0" which is a monotonically
increasing semantic version number starting from 2.0.

In these cases, we need a new microversion:
1. If the parameters of an API change, we need to add a new microversion.
For example, when we add a new parameter to limit the length of return value of
the device list API.

2. No parameters to any API has changed, but the functionality of some API has
changed.
For example, we already have a device list API with a filter dictionary as the
input param, it currently support list the device filtered by "attribute_A" and
"attribute_B", now we also want this API to return the device list filtered by
"attribute_C". In this case, the API input parameter is always the filter dict.
But the API's functionality changes, so we also need to add a microverison.

If a user makes a request without specifying a version, they will get the
DEFAULT_API_VERSION as defined in cyborg/api/versions.py. This value
is currently 2.0 and is expected to remain so for quite a long time. We
will not change this default version unless the major version changed and
we deprecate V2 api version. It seems that we have very little change to
update this value.

Versioning of the API should be a single monotonically increasing counter. It
will be of the form X.Y where it follows the following convention:

X will only be changed if a significant backwards incompatible API change is
made which affects the API as whole. That is, something that is only very
rarely incremented.

Y will only be changed when you make any change to the API. Note that this
includes semantic changes which may not affect the input or output formats or
even originate in the API code layer. We are not distinguishing between
backwards compatible and backwards incompatible changes in the versioning
system. However, it will be made clear in the documentation as to what is a
backwards compatible change and what is a backwards incompatible one.

Version Discovery
=================

The Version API will return the minimum and maximum microversions. These values
are used by the client to discover the API's supported microversion(s).

Requests to / will get version info for all endpoints. A version response would
look as follows for GET ``http://host_ip/accelerator/``

.. code-block:: json

 {
   "default_version": {
     "status": "CURRENT",
     "min_version": "2.0",
     "max_version": "2.1",
     "id": "v2.0",
     "links": [
       {
         "href": "http://host_ip/accelerator/v2",
         "rel": "self"
       }
     ]
   },
   "versions": [
     {
       "status": "CURRENT",
       "min_version": "2.0",
       "max_version": "2.1",
       "id": "v2.0",
       "links": [
         {
           "href": "http://host_ip/accelerator/v2",
           "rel": "self"
         }
       ]
     }
   ],
   "name": "OpenStack Cyborg API",
   "description": "Cyborg is the OpenStack project for lifecycle "
                  "management of hardware accelerators, such as GPUs,"
                  "FPGAs, AI chips, security accelerators, etc."
 }

Alternatives
------------

Leave it as is, when user updates Cyborg along with new OpenStack release.
It may lead to potential incapability to serve request from legacy clients.

Data model impact
-----------------

None

REST API impact
---------------

Every API method should be decorated by a version-checking decorator.

For more Cyborg API details, please see Cyborg API Documentation
[#cyborg-api-doc]_

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

SDK authors will need to start using the OpenStack-API-Version header to
get access to new features. The fact that new features will only be added
in new versions will encourage them to do so.

python-cyborgclient is in an identical situation and will need to be updated
to support the new header in order to support new API features.

Performance Impact
------------------

None

Other deployer impact
---------------------
None

Developer impact
----------------

Any future changes to Cyborg's REST API (whether that be in the request or
any response) must result in a microversion update, and guarded in the code
appropriately.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Xinran Wang(xin-ran.wang@intel.com)

Other contributors:
  Shogo Saito

Work Items
----------

* Push OpenStack-API-Version header to API layer.
* Add decorator for microversion check.
* Implement versions.py module with history of API changes.
* replace api_version_request.py by versions.py and use microversion_parser
  library.
* Change python-cyborgclient side to support v2.0 microversion.


Dependencies
============

None

Testing
=======

Appropriate unit and functional tests should be added.

Documentation Impact
====================

* Need a documentation to record microversion history.
* Need a documentaiton to explain what is the backwards
  compatible/incompatible, what is the microverison of Cyborg, how to discover
  the versions and how to interact with cyborgclient.

References
==========

.. [#microversion-spec] `Microversion Specification
   <http://specs.openstack.org/openstack/api-wg/guidelines/microversion_specification.html>`_
.. [#cinde-mv-spec] `Cinder Microversion Spec
   <https://specs.openstack.org/openstack/cinder-specs/specs/mitaka/api-microversions.html>`_
.. [#cyborg-api-doc] `Cyborg API Documentation
   <https://docs.openstack.org/api-ref/accelerator/v2/index.html>`_

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Ussuri
     - Introduced
