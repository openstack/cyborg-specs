..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Policy Default Refresh
=======================

Role Based Access Control (RBAC) policies in OpenStack has long been the
forefront of operator concerns and pain. The implementation is complicated to
understand, inconsistent across projects, and lacks secure defaults. To
improve the existing default policy of OpenStack, the
Consistent_and_Secure_Default_Policies_Popup_Team [#policy_popup_team]_ was
built up to track policy refresh for all projects, where Keystone was the lead.
Keystone defined ongoing policy goals and roadmaps, default roles, as well as
a reclarification of system-scoped and project-scoped RBAC. As a member of this
popup_team, Cyborg is also going to follow up policy default refresh.

Problem description
===================

The current default policy in Cyborg is incomplete and not good enough.
Since Cyborg V2 API is newly implemented in Train, RBAC check for V2 API still
remains incomplete. And now Cyborg mainly has three policy rules:

* allow
* admin_only
* admin_or_owner

Firstly "allow" means any access will be passed. Now "allow" rule is used by
cyborg:arq:create, which is too slack. We've reached an agreement that this
should not be open for all users [#arq-create-discussion]_. As for the new
rule, please see that in the cyborg policy table introduced in Use Cases part.

Secondly "admin_only" is used for the global admin that is able to make almost
any change to cyborg, and see all details of the cyborg system.
The rule actually passes for any user with an admin role, it doesn't matter
which project is used, any user with the ``admin`` role gets this global
access.

Thirdly "admin_or_owner" sounds like it checks if the user is a member of a
project. However, for most APIs we use the default target which means this
rule will pass for any authenticated user.

With all the above policy rules, there still some cases which are not well
covered. For example, it is impossible to allow a user to retrieve/update
devices which are shared by multiple projects from a system level without
being given the global admin role. In addition, cyborg now doesn't have a
"reader" role.

Keystone comes with member, admin and reader roles by default. We can
use these default roles:
https://specs.openstack.org/openstack/keystone-specs/specs/keystone/rocky/define-default-roles.html

In addition, we can use the new "system scope" concept to define
which users are global administrators:
https://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/system-scope.html

Use Cases
---------

The following user roles should be supported by cyborg default configuration:

* Add System Scoped Admin: to operate system-level resources
* Add System Scoped Reader: read-only role for system-level resources
* Add Project Scoped Reader: read-only role for project-level resources
* Refresh existed admin to Project Scoped Admin by default
* Add Project Scoped Member: role for project-level non-admin APIs

Cyborg V2 APIs need RBAC check. Objects of V2 APIs are listed in the following:

* device_profile [#device_profile]_, as the "flavor" in accelerator request,
  is generally supposed to be a system-level resource especially for public
  cloud, where billing is based on the device_profile. So we reached an
  agreement [#agreement-on-device_profile]_ that sys_admin is required for
  device_profile:create and delete. As for other specific cloud, such as
  private cloud providers, they can change the policy by themselves if they
  want other users to create device_profiles.

* devices and deployables [#device-and-deployable-data-model]_ objects are
  used to describe hardware accelerators, where a device refers to the
  hardware and deployables are derived from a device. device:update API is
  introduced for sys_admin to enable or disable a specific device, while
  deployable:update API offers project_admin a way to update the shell
  image/FPGA bitstream to custom user logic for the specified deployable.
  NOTE that sys_admin is required to do the pre_configuration for devices.

* accelerator_requests are sent by nova to bind/unbind accelerators to VMs
  during instance boot. Any peoject user who can create VM should be able to
  create an arq, so "member" role is required for arq:create. And for the
  arq:patch and arq:delete admin_or_owner should make sense.

To be clear, the roles needed for all Cyborg V2 APIs' operations are defined
in the table `cyborg-policy-table <https://wiki.openstack.org/wiki/Cyborg/Policy>`_.

In introducing the above new default permissions, we must ensure:

* Operators using default policy are given at least one cycle to add
  additional roles to users (likely via implied roles)
* Operators with over-ridden policy are given at least one cycle to
  understand how the new defaults may or may not help them

Proposed change
===============

According to the discussions above, we will try to make the changes as less
as possible to meet the requirements. For the current stage, there should be at
least the following changes. Each policy rules will be covered with appropriate
oslo.policy's "scope_types", 'system' and 'project' in cyborg case. And we will
use the DocumentedRuleDefault to update policy and follow the oslo.policy
deprecation workflow, where both old and new policy check strings are active
during the deprecation period.

* Add system scoped admin policy
  This policy will be useful for situations where devices are shared by
  multiple projects, and we want a system-level admin to operator the devices
  like programming or firmware upgrade. In addition, a system admin is required
  to do the service disable/enable things.
* Add system scoped reader policy
  This policy will be useful for situations where a read-only role is required
  for a more secure access to devices that are shared by multiple projects in
  a system.
* Add project scoped reader policy
  Similarly, this policy will be useful for situations where a read-only role
  is required for a more secure access at a project level. For example, some
  users should only have the permission to check and use the resources but
  shouldn't update the resources.
* Add project scoped member plicy
  This policy will be used to check if a user is eligible to post a non-admin
  API such as arq creation.

POC: https://review.opendev.org/#/c/699102/, https://review.opendev.org/#/c/700765/

Alternatives
------------

Keep the old policy mechanism, but on one hand, as more apis features added,
the policy check will become more complicated, on the other hand, there are
some important situations which are not covered or well-covered by old policy
mechanism. This refresh will make cyborg policy basically comprehensive and
clean.

Data model impact
-----------------

None

REST API impact
---------------

Operations for each API should be reassessed and associated which scope, or
scopes are appropriate.

The following new roles will be added to replace legacy policies, more details
can be found in the aforementioned `cyborg-policy-table <https://wiki.openstack.org/wiki/Cyborg/Policy>`_:

* Project Reader check

  * GET /v2/device_profiles

  * GET /v2/device_profiles/{device_profiles_uuid}

  * GET /v2/accelerator_requests

  * GET /v2/accelerator_requests/{accelerator_request_uuid}

* System Reader check

  * GET /v2/devices

  * GET /v2/devices/{device_uuid}

* System Admin check

  * PATCH /v2/devices

  * POST /v2/device_profil

  * DELETE /v2/device_profiles/{device_profiles_uuid}

  * DELETE /v2/device_profiles?value={dev_profile_name1},{dev_profile_name2}

* Project Admin check

  * PATCH /v2/deployables/{uuid}

* Project Member check

  * POST /v2/accelerator_requests

  * PATCH /v2/accelerator_requests/{arq_uuid}

  * DELETE /v2/accelerator_requests?arqs={arq_uuid}

  * DELETE /v2/accelerator_requests?instance={instance_uuid}

.. note::

  The default roles discussed will be created by Keystone, during the bootstrap
  process, using `implied roles
  <https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/implied_role.html>`_.
  As indicated in the above list, having ``admin`` role implies a user also
  has the same rights as the ``member`` role. Therefore this user will also has
  the same rights as the ``reader`` role as ``member`` implies ``reader``.

  This keeps policy files clean. For example, the following are equivalent as a
  result of implied roles:

  "cyborg:device:get_all": "role:reader OR role:member OR role:admin"
  "cyborg:device:get_all": "role:reader"

   The chain of implied roles will be documented alongside of the
   `policy-in-code defaults
   <https://github.com/openstack/keystone/blob/master/keystone/common/policies/base.py>`_
   in addition to general Keystone documentation updates noting as much.

Security impact
---------------

Policy defaults refresh will help keep the system secure.

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

Deployers will need to look through the new policies
(communicated via release notes) to make sure they can adopt them.

Developer impact
----------------

New APIs must add policies that follow the new pattern.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <yumeng-bao>

Work Items
----------

In order to make sure existed policies run normally when every changes happen,
we will follow the [#policy-migration-steps]_ and propose changes in the
following order:

* Add new roles to cyborg policy including Project-Reader, Project-Member,
  System-Reader, System-Admin.

* Update APIs and unit tests that are using the above new roles.

* Update APIs and unit tests that are using other roles such as Project-Admin,
  Admin_or_user etc.

* Refactor cyborg policy file.

Dependencies
============

None

Testing
=======

Tests for policy rules and APIs should be added.
Reference RBAC test in keystone-tempest-plugin:
https://review.opendev.org/#/c/686305/

Documentation Impact
====================

API Reference should be kept consistent with any policy changes, in particular
around the default reader role.

References
==========
.. [#policy_popup_team] `Consistent and Secure Default Policies Popup Team
   <https://wiki.openstack.org/wiki/Consistent_and_Secure_Default_Policies_Popup_Team>`_

.. [#arq-create-discussion] `Roles for arq-create
   <http://eavesdrop.openstack.org/meetings/openstack_cyborg/2020/openstack_cyborg.2020-02-06-03.01.log.html#l-74>`_

.. [#device_profile] `Device profile definition
   <http://specs.openstack.org/openstack/cyborg-specs/specs/train/approved/device-profiles.html>`_

.. [#agreement-on-device_profile] `Roles for device_profile
   <http://eavesdrop.openstack.org/meetings/openstack_cyborg/2020/openstack_cyborg.2020-02-06-03.01.log.html#l-143>`_

.. [#device-and-deployable-data-model] `Device and deployable definitions
   <http://specs.openstack.org/openstack/cyborg-specs/specs/stein/approved/cyborg-database-model-proposal.html>`_

.. [#policy-migration-steps] `Policy Migration Steps
   <https://etherpad.openstack.org/p/policy-migration-steps>`_

History
=======

Optional section intended to be used each time the spec is updated to describe
new design, API or any database schema updated. Useful to let reader understand
what's happened along the time.

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Ussuri
     - Introduced
