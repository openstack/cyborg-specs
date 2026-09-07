..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Consistent and Secure RBAC for Cyborg APIs
===========================================

https://blueprints.launchpad.net/openstack-cyborg/+spec/consistent-and-secure-rbac

Cyborg's REST API authorization still relies on legacy ``RuleDefault``
policies defined in ``cyborg/common/policy.py``. Although critical
authorization bypasses have been fixed (LP#2143263, LP#2144056) and
project-scoped ARQ isolation is now enforced at the data layer, the policy
definitions have not been migrated to ``DocumentedRuleDefault`` with
persona-based defaults. This spec completes the OpenStack TC's `Consistent
and Secure RBAC community goal`_ for Cyborg in the 2026.2 cycle by
delivering all three phases of the goal: project personas, service role
targeting, and the project-manager persona.

.. _Consistent and Secure RBAC community goal:
   https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html


Problem description
===================

The OpenStack TC's `Consistent and Secure RBAC community goal`_ defines a
three-phase migration to modern role-based access control across all
OpenStack services. Cyborg partially migrated the device-profile controller
to ``DocumentedRuleDefault`` with ``scope_types`` in the Wallaby cycle, but
all other endpoints remain on legacy ``RuleDefault`` policies. Those legacy
rules lack deprecated-rule bridges, persona targeting, and the
``DocumentedRuleDefault`` metadata required for operators to opt into the new
RBAC defaults via ``enforce_new_defaults = True``.

Recent security patches have closed acute authorization vulnerabilities and
established the data-model foundations that the SRBAC migration builds on.
The fix for LP#2143263 replaced all ``rule:allow`` check strings with
role-checked rules and added ``scope_types=['project']`` to every policy in
``cyborg/common/policy.py``. The fix for LP#2144056 was a four-patch series
that populates ``project_id`` on ARQ creation and binding, provides an
online data migration to backfill ``project_id`` on historical ARQs,
enforces project-scoped access at the object and database layers for all ARQ
operations, and requires a valid service token for bound ARQ operations such
as bind, unbind, and delete-by-instance.

With those patches merged, the current policy baseline is secure but
incomplete. Device, deployable, and attribute endpoints are locked to
admin-only access, which is more restrictive than the long-term SRBAC
target. ARQ reads still use the deprecated ``rule:default`` chain, which
relies on the deprecated ``is_admin:True or project_id:%(project_id)s``
check rather than the modern reader persona. The policy definitions are
still ``RuleDefault`` entries without deprecated-rule bridges, so enabling
``enforce_new_defaults = True`` has no effect on those endpoints. There is no
``project_manager_or_admin`` base rule for the manager persona, and no
``project_member_or_service`` rule to begin the transition toward service
role authorization for machine-to-machine APIs.


Use Cases
---------

* As a cloud operator (``admin`` role), I want to manage the hardware
  lifecycle — disabling and enabling devices, programming FPGA bitstreams,
  creating and deleting device profiles, and enriching device metadata —
  while also having read access to all hardware inventory and all ARQs
  regardless of project.

* As a trusted end user (``manager`` role), I want to inspect the
  accelerator hardware inventory — devices, deployables, and attributes —
  available to my project for capacity planning and troubleshooting, without
  being granted write access to hardware management operations.

* As a normal end user (``member`` role), I want Nova to be able to create,
  bind, and delete ARQs on my behalf during instance lifecycle operations,
  using my Keystone token as the primary credential.

* As an auditor (``reader`` role), I want read-only access to the ARQs
  belonging to my project so that I can inspect accelerator request state,
  for example to check bind state during instance scheduling.

* As a deployer, I want Cyborg to default to the legacy authorization
  behaviour during the upgrade window so that I can migrate all services in
  my deployment to the new persona model together, rather than being forced
  to adopt new defaults before the rest of the stack is ready.

* As a Nova developer, I want Cyborg's ARQ write policies to accept the
  ``service`` role alongside ``member`` so that when Nova transitions to
  presenting its service account token as the primary credential, the
  Cyborg policy layer does not need a coordinated change.


Proposed change
===============

All three SRBAC phases will be delivered together in 2026.2 as a single
coherent policy migration. Four new policy modules will be created,
replacing the legacy rule lists in ``cyborg/common/policy.py``. Every new
policy will use ``DocumentedRuleDefault`` with ``scope_types=['project']``
and a ``deprecated_rule`` that bridges the current legacy check string so
that existing deployments continue to work when
``enforce_new_defaults = False``. The ``deprecated_since`` value will be set
to ``versionutils.deprecated.WALLABY`` across all new modules, consistent
with the existing deprecation epoch used by the device-profile policies.

Base rule additions
-------------------

Three new constants and two new base rules will be added to
``cyborg/policies/base.py``::

    SERVICE = 'role:service'
    PROJECT_MANAGER_OR_ADMIN = 'rule:project_manager_or_admin'
    PROJECT_MEMBER_OR_SERVICE = 'rule:project_member_or_service'

The ``project_manager_or_admin`` rule will grant access to users holding
the ``manager`` role on the request's project, or to cloud administrators.
The ``manager`` role represents a project-level delegation tier above
``member`` but below cloud ``admin``, intended for team leads or designated
project administrators who need operational visibility without full admin
privileges. The ``project_member_or_service`` rule (check string:
``rule:project_member_api or rule:service_api``) will accept both the
``member`` role scoped to the request's project and the ``service`` role
for trusted service accounts, bridging the transition from member-based to
service-based authorization for machine-to-machine APIs. Both rules will be
registered in the ``default_policies`` list alongside the existing
``project_member_or_admin`` and ``project_reader_or_admin`` rules.

The existing ``admin_api`` base rule currently includes
``role:administrator`` as a non-standard alias alongside ``role:admin``.
This alias is not part of the Keystone bootstrap role set and will be
removed as part of this work, aligning Cyborg with the standard role names
used by Nova and other services.

ARQ policies
------------

A new ``cyborg/policies/arqs.py`` module will define
``DocumentedRuleDefault`` entries for all five ARQ operations.

The ARQ endpoints currently use a mix of legacy rules: ``get_all``,
``get_one``, ``delete``, and ``update`` use ``rule:default`` (which
resolves to ``rule:admin_or_owner``), while ``create`` uses
``rule:project_member_or_admin`` (set by the LP#2143263 fix).

.. list-table:: ARQ policy mapping
   :header-rows: 1
   :widths: 30 35 35

   * - Policy rule
     - New default
     - Deprecated bridge
   * - ``cyborg:arq:get_one``
     - ``project_reader_or_admin``
     - ``admin_or_owner``
   * - ``cyborg:arq:get_all``
     - ``project_reader_or_admin``
     - ``admin_or_owner``
   * - ``cyborg:arq:create``
     - ``project_member_or_service``
     - ``project_member_or_admin``
   * - ``cyborg:arq:delete``
     - ``project_member_or_service``
     - ``admin_or_owner``
   * - ``cyborg:arq:update``
     - ``project_member_or_service``
     - ``admin_or_owner``

The read operations will grant project readers explicit read-only access to
their own ARQs. The write operations will use the
``project_member_or_service`` composite rule, whose check string is
``rule:project_member_api or rule:service_api``. This accepts both the
``member`` role — what Nova's current behaviour provides when it forwards
the end user's project-scoped token — and the ``service`` role — what Nova
will provide in a future cycle when it presents its own service account
token as the primary credential. The ``admin`` role is not part of the new
default; admin access to ARQ writes during the transition window is
provided solely through the deprecated bridge.

Bound ARQ operations — bind, unbind, and delete-by-instance — will remain
protected by the hardcoded service-token gate introduced in the LP#2144056
fix. That gate is an API-layer check independent of oslo.policy evaluation
and cannot be overridden via ``policy.yaml`` configuration.

.. note::

   The ``member`` component of the ARQ write check string exists to
   support Nova's current token-forwarding behaviour. Once Nova switches
   to presenting its service account token as the primary credential for
   Cyborg calls, the ``member`` component will be deprecated and
   subsequently removed, leaving ``role:service`` as the sole new
   default. Including ``role:service`` in the check string now means
   Cyborg will not require a coordinated policy change when that Nova
   transition happens.

Device, deployable, and attribute policies
------------------------------------------

Three new policy modules will be created:
``cyborg/policies/devices.py`` (four operations),
``cyborg/policies/deployables.py`` (three operations), and
``cyborg/policies/attributes.py`` (four operations). All endpoints in
these groups currently use ``rule:admin_api``. The deprecated bridge for
every endpoint will be ``rule:admin_api``, matching the current state.

The new defaults follow a consistent pattern: read endpoints will widen
access from admin-only to the project-manager persona, while write and
hardware-management endpoints will remain admin-only.

.. list-table:: Hardware inventory policy mapping
   :header-rows: 1
   :widths: 40 35 25

   * - Policy rule
     - New default
     - Deprecated bridge
   * - ``cyborg:device:get_one``
     - ``project_manager_or_admin``
     - ``admin_api``
   * - ``cyborg:device:get_all``
     - ``project_manager_or_admin``
     - ``admin_api``
   * - ``cyborg:device:disable``
     - ``admin_api``
     - ``admin_api``
   * - ``cyborg:device:enable``
     - ``admin_api``
     - ``admin_api``
   * - ``cyborg:deployable:get_one``
     - ``project_manager_or_admin``
     - ``admin_api``
   * - ``cyborg:deployable:get_all``
     - ``project_manager_or_admin``
     - ``admin_api``
   * - ``cyborg:deployable:program``
     - ``admin_api``
     - ``admin_api``
   * - ``cyborg:attribute:get_one``
     - ``project_manager_or_admin``
     - ``admin_api``
   * - ``cyborg:attribute:get_all``
     - ``project_manager_or_admin``
     - ``admin_api``
   * - ``cyborg:attribute:create``
     - ``admin_api``
     - ``admin_api``
   * - ``cyborg:attribute:delete``
     - ``admin_api``
     - ``admin_api``

Device inventory describes physical hardware on compute nodes. Deployables
represent the logical accelerator units exposed by a device. Attributes
are key-value metadata describing accelerator capabilities, populated
automatically by cyborg-agent during hardware discovery. The read endpoints
grant the project-manager persona visibility into hardware topology for
capacity planning and troubleshooting. Write and management operations —
device disable/enable, deployable program (FPGA bitstream reprogramming via
RPC to cyborg-agent), and attribute create/delete — affect shared physical
infrastructure and will remain restricted to cloud admins.

Policy registration
-------------------

``cyborg/policies/__init__.py`` will be updated to import the four new
modules and stop loading the legacy lists from ``cyborg/common/policy.py``.
The legacy module will be retained as a stub during the deprecation window
to satisfy any external ``policy.yaml`` overrides that reference the old
rule names via the ``deprecated_rule`` bridge. In 2027.1 when the new
defaults become the default, the old defaults will be deprecated with
warnings announcing their planned removal. The legacy rules will then be
removed in 2027.2.

SRBAC goal phase coverage
-------------------------

Although the three community goal phases will be delivered as a single code
change, each phase requirement will be satisfied.

Phase 1 will deliver project personas by migrating all endpoints to
``DocumentedRuleDefault`` with ``scope_types=['project']``. Reader access
will be granted for ARQs, member access for ARQ writes, manager access for
hardware inventory reads, and admin access for all management operations.

Phase 2 will introduce the ``service`` role for machine-to-machine APIs
through the ``rule:project_member_or_service`` composite rule on ARQ write
operations. Bound ARQ operations will remain gated by the hardcoded
service-token check from LP#2144056. In a future cycle,
once Nova presents service credentials as primary, the
``member`` component of the ARQ write check string will be marked as
deprecated, announcing the intent to remove it in a subsequent release.

Phase 3 will introduce the project-manager persona through the
``project_manager_or_admin`` base rule, used for device, deployable, and
attribute read endpoints. No Cyborg write operation is appropriate for the
manager role; hardware management operations will remain admin-only.


Alternatives
------------

Grant project-reader access to device, deployable, and attribute reads
  This would be simpler than introducing the manager persona, but device,
  deployable, and attribute data describes the physical hardware topology of
  compute nodes — hostnames, PCI bus addresses, driver names, and Placement
  resource-provider UUIDs. This is operational information not relevant to
  ordinary end users, who interact with accelerators solely through Nova and
  device profiles. The project-manager persona is the appropriate access
  level for operators or designated project administrators who need
  visibility into the underlying hardware.

Use ``role:service`` as the sole 2026.2 default for ARQ writes
  This would be the ideal security posture, but oslo.policy evaluates
  ``role:service`` against the primary (user) token, not the service token.
  Nova currently sends the end user's token as primary, so ``role:service``
  alone would deny all Nova-initiated ARQ operations. The composite
  ``rule:project_member_or_service`` bridges this gap by accepting both
  roles, giving Nova at least one release cycle to transition to presenting
  service credentials before ``member`` is removed from the check string.

Introduce a global auditor persona using a system-scoped reader role
  A system-scoped reader could provide cross-tenant read-only access to all
  Cyborg endpoints, allowing a cloud auditor to inspect hardware inventory
  and ARQ state across all projects without requiring the admin role. This
  is a useful capability but is out of scope for this spec. The community
  goal explicitly deferred system-scope work, and adding it here would
  require reintroducing system scope to Cyborg's ``scope_types`` and
  coordinating with Keystone's system-scope token flow. This could be
  proposed as a future addition once the project-scoped persona model is
  established.


Data model impact
-----------------

None.

The ARQ ``project_id`` column and backfill migration were delivered in
the LP#2144056 fix series and are already merged. Policy changes are
authorization-only and do not affect database schema or object models.


REST API impact
---------------

No request or response schemas, HTTP methods, or URLs change. The only
impact is to authorization semantics: the new ``DocumentedRuleDefault``
policies described in the Proposed change section alter which roles are
accepted for each endpoint. Endpoints that currently require admin will be
widened to accept the manager or reader personas where appropriate, and ARQ
write endpoints will accept the ``service`` role alongside ``member``.

During the transition window (``enforce_new_defaults = False``) oslo.policy
ORs the new check strings with the deprecated bridges, so no endpoint
becomes more restrictive until the operator explicitly enables new defaults.
The per-endpoint policy mappings are defined in the Proposed change section.


Security impact
---------------

This change is a security improvement that builds on the foundation
established by the LP#2143263 and LP#2144056 fixes.

All endpoints will be brought into the ``DocumentedRuleDefault`` framework
with ``scope_types=['project']``, aligning Cyborg with the OpenStack
defence-in-depth authorization model. The ``enforce_scope`` option already
defaults to ``True`` and is planned for removal this cycle, making
project-scoped enforcement the only behaviour. Hardware inventory will move
from admin-only to the project-manager persona, providing appropriate
access without exposing topology to all project members. ARQ reads will be
tightened from the deprecated ``admin_or_owner`` check to the modern
``project_reader_or_admin`` persona, which properly verifies the ``reader``
role and project ownership. The service-token gate for bound ARQ
operations, already merged, provides defence-in-depth that cannot be
overridden by policy configuration.


Notifications impact
--------------------

None.


Other end user impact
---------------------

Project readers can now query the ARQs belonging to their own project.
Previously the interim policy required the member role for ARQ reads.
Project managers gain read access to devices, deployables, and attributes.
Previously the interim policy required admin for these endpoints.

End users launching accelerator-enabled instances via Nova are not affected.
Nova calls Cyborg using the end user's token, which carries the
project-member role, and that role satisfies both the current deprecated
bridge and the new default for ARQ write operations.


Performance Impact
------------------

None.


Other deployer impact
---------------------

The ``manager`` and ``service`` roles must exist in Keystone, both standard
since Yoga. The ``service`` role must be assigned to Nova's service account
for the service-token gate to function. The transition timeline is:

* **2026.2** — Cyborg will override ``enforce_new_defaults`` to ``False``.
  Both old and new authorization paths are accepted. Operators may opt in
  early by setting ``enforce_new_defaults = True``.
* **2027.1** — The override will be removed and new defaults will be
  enforced by default. The legacy rules will be formally deprecated with
  warnings announcing their removal. Operators not yet ready can set
  ``enforce_new_defaults = False`` explicitly for one more cycle.
* **2027.2** — The deprecated legacy rules and bridges will be removed.


Developer impact
----------------

Developers adding new Cyborg API endpoints must use
``DocumentedRuleDefault`` with ``scope_types=['project']``. The
``rule:allow`` pattern must not be used in new code. The
``cyborg/policies/device_profiles.py`` module is the reference
implementation for the expected policy definition style.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sean-k-mooney

Other contributors:
  None

Work Items
----------

* Add ``project_manager_or_admin`` and ``project_member_or_service`` base
  rules and their constants to ``cyborg/policies/base.py``. Override
  ``enforce_new_defaults`` to ``False`` in Cyborg's default configuration.
* Create ``cyborg/policies/arqs.py`` with ``DocumentedRuleDefault`` entries
  for all five ARQ operations. Reads will use
  ``rule:project_reader_or_admin``; writes will use
  ``rule:project_member_or_service``. Deprecated bridges will reflect the
  current interim rules.
* Create ``cyborg/policies/devices.py``, ``cyborg/policies/deployables.py``,
  and ``cyborg/policies/attributes.py`` with ``DocumentedRuleDefault``
  entries for all hardware inventory operations. Reads will use
  ``rule:project_manager_or_admin``; management operations will remain
  ``rule:admin_api``. See the hardware inventory policy mapping table
  in the Proposed change section for the full per-endpoint breakdown.
* Update ``cyborg/policies/__init__.py`` to import the new modules and stop
  loading legacy lists from ``cyborg/common/policy.py``.
* Add unit policy tests covering both ``enforce_new_defaults = True`` and
  ``enforce_new_defaults = False`` behaviour for each new module.
* Validate ``tox -e genpolicy``, regenerate the policy reference page, and
  add a release note describing the new personas and transition timeline.


Dependencies
============

None


Testing
=======

For each new policy module a unit test class will be added under
``cyborg/tests/unit/policies/`` following the ``test_device_profiles.py``
pattern. Each test class will cover both ``enforce_new_defaults = True``
and ``enforce_new_defaults = False`` to verify that the new check strings
and deprecated bridges behave correctly for each persona.

``tox -e genpolicy`` must succeed after adding the new modules.

Tempest RBAC tests covering persona-based access for each endpoint group
are an aspirational goal for this work. The Cyborg tempest plugin currently
has no persona coverage and adding it would significantly improve
confidence in the policy migration. Tempest RBAC tests are not gating for
the policy migration patches but are intended to follow as part of this
effort.


Documentation Impact
====================

The policy reference page ``doc/source/configuration/policy.rst`` must be
regenerated from the updated ``genpolicy`` output to reflect the new default
check strings, deprecated bridges, and descriptions for all migrated
endpoints.

A release note is required describing the new personas available for each
endpoint, the ``enforce_new_defaults = False`` Cyborg default for 2026.2,
and the three-cycle transition plan: new defaults available in 2026.2,
enforced by default in 2027.1, legacy rules removed in 2027.2.


References
==========

* Consistent and Secure RBAC community goal:
  https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html

* RBAC Goal Tracking Etherpad:
  https://etherpad.opendev.org/p/rbac-goal-tracking

* LP#2143263 — rule:allow policy bypass:
  https://bugs.launchpad.net/openstack-cyborg/+bug/2143263

* LP#2144056 — ARQ cross-tenant access and service-token enforcement:
  https://bugs.launchpad.net/openstack-cyborg/+bug/2144056

* Wallaby device-profile policy migration (reference implementation):
  ``cyborg/policies/device_profiles.py``

* oslo.policy DocumentedRuleDefault documentation:
  https://docs.openstack.org/oslo.policy/latest/reference/api/oslo_policy.policy.html

* Nova policy concepts (persona definitions and transition approach):
  https://docs.openstack.org/nova/latest/configuration/policy-concepts.html

* Keystone service API protection (service token composite authorization):
  https://docs.openstack.org/keystone/latest/admin/service-api-protection.html


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2026.2
     - Introduced
