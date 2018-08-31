========================
Team and repository tags
========================

.. image:: http://governance.openstack.org/tc/badges/cyborg-specs.svg
    :target: http://governance.openstack.org/tc/reference/tags/index.html

.. Change things from this point on

===============================
OpenStack Cyborg Specifications
===============================

This git repository is used to hold approved design specifications for additions
to the Cyborg project. Reviews of the specs are done in gerrit, using a similar
workflow to how we review and merge changes to the code itself.

The layout of this repository is::

  specs/<release>/

Where there are two sub-directories:

  specs/<release>/approved: specifications approved but not yet implemented
  specs/<release>/implemented: implemented specifications

This directory structure allows you to see what we thought about doing,
decided to do, and actually got done. Users interested in functionality in a
given release should only refer to the ``implemented`` directory.

You can find an example spec in `doc/source/specs/template.rst`.

To track all the blueprints of cyborg, please refer to the trello board:
https://trello.com/b/4nFtHNSg/queens-dev

To validate that the specification is syntactically correct (i.e. get more
confidence in the Jenkins result), please execute the following command::

  $ tox

After running ``tox``, the documentation will be available for viewing in HTML
format in the ``doc/build/`` directory.
