..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
        Cyborg Database Model Proposal
==========================================

Blueprint:
https://blueprints.launchpad.net/openstack-cyborg/+spec/\
cyborg-database-modelling

This spec proposes a new DB modeling schema for tracking cyborg resources

Problem description
===================

Heterogeneous acceleration resources have become essential in the cloud, edge,
and high-performance computing scenarios. These devices achieve higher
efficiency by tailoring the architecture to characteristics of the domain.
They provide effective parallelism, effective use of memory bandwidth,
etc. Hence tracking and deploying these accelerators are much-needed features.


Use Cases
---------

For instance, when the user requests FPGA resources, the scheduler will use
placement agent to select appropriate hosts that have the requested FPGA
resources.

For instance, when Nova picks a device (GPU/FPGA/etc.) resource provider to
allocate to a VM, Cyborg needs to track down which exact device has been
assigned in the database. On the other hand, when the resource is released,
Cyborg will need to be detached and free the exact resource.

When a new device is plugged into the system(host), Cyborg needs to discover
it and store all its related information into the database

In addition, when a device is removed from the system, cyborg also needs to
update the database accordingly.

Proposed change
===============

We need to add 5 more table to Cyborg database. First one is Devices. The
purpose of it is to track the physical existence of heterogeneous devices.
Second one is AttachHandles, which tracks the attachment information needed to
attach an accelerator to a VM. Thrid one is ControlPathID table, which
essentially is containing device specific information on where the
accelerator is located and ready to be attached. The fourth one is ExtARQs
(accelerator requests) table. It is a syncing point with Nova for accelerator
requests. At last, DeviceProfiles table is also added. It tracks the set of
requirements for accelerators.


In addition, we need to repurpose the existing table: Deployables. And remove
the existing Accelerators table.

Devices table consists of all the fields needed to describe the physical
existence of a hardware device in the data center system. For instance, type,
std_board_info, vendor_board_info, etc. This table should be populated by the
discovery API.

AttachHandles table tracks the attachment information needed to attach an
accelerator to a VM. Its records spawned when Nova requests to attach
accelerators.

ControlPathID table tracks identifiers for a control path interface to devices.
E.g. PCI PF. Aka Device ID. A device may have more than one of these, in which
case the Cyborg driver needs to know how to handle these.

ExtARQs table tracks the accelerator requests which are sent by Nova. It contains
fields both Nova acknowledgeable as well as only Cyborg specific. For the fields
that are Nova acknowledgeable, they will be used to form ARQ objects. On the
other hand, for cyborg specifc fields, they will be used to form ExtARQ objects.

DeviceProfiles [#device-profile-spec]_ table tracks  the set of requirements
for accelerators. A device profile is a named set of the user requirements for
one or more accelerators. It can be viewed as a flavor for devices. Broadly it
includes two things: the desired amounts of specific resource classes and the
requirements that the resource provider(s) must satisfy. While the resource
classes are the same as those known to Placement, some requirements would
correspond to Placement traits and others to properties that Cyborg alone
knows about.

Deployables table now serves the purpose of describing all the derived
resources from a given Device(referred in device_uuid as the foreign key).
Similarly, it consists of all the common attributes columns as well as
a parent_id and a root_id. The parent_id will point to the associated parent
deployable and the root_id will point to the associated root deployable.
By doing this, we can form a nested tree structure to represent different
hierarchies. For the case where FPGA has not been loaded any bitstreams on it,
they will still be tracked as a Deployable but no other Deployables referencing
to it. Once a bitstream is loaded, the structure and properties of deployables
can be changed. In the context of Placement in Nova, deployable's counterpart
is Resource Provider

Accelerators table now should be removed. However, the concept of accelerator
has changed and deployable table tracks a num_accelerators field. It represents
the number of accelerator current deployable can spawn. Additionally, once an
accelerator is assigned, Cyborg creates an attach handle pointing to the
corresponding deployable

For example, a network of device hierarchies can be formed using devices,
deployables, accelerators, and attributes in the following scheme::

                                -------------------
                                |Device   -   FPGA|
                                -------------------
                                        /\
                         device_uuid   /  \  device_uuid
                                      /    \
                      -----------------    -----------------
            |-------->|   Deployable  |    |   Deployable  |<----------|
            |         -----------------    -----------------           |
     root_id|               /                  \                       |
            |    parent_id /          parent_id \              root_id |
            |             /                      \                     |
            |            /                        \                    |
      -----------------                      -----------------         |
      |   Deployable  |num_accelerators=2    |   Deployable  |---------|
      -----------------                      -----------------
           /           \                      ^ ^  -------------
          /             \       deployable_id | |--|Attribute A|
         /deployable_id  \                    |    -------------
    -----------------     -----------------   |     -------------
    |Attach Handle A|     |Attach Handle B|   |---- |Attribute B|
    -----------------     -----------------         -------------

Attributes table should stay the same as before, which consists of a key and a
value columns to represent arbitrary k-v pairs.

For instance, bitstream_id and function kpi can be tracked in this table.
In addition, a foreign key deployable_id refers to the Deployables table and
a parent_attribute_id to form nested structured attribute relationships.

Alternatives
------------

Alternatively, instead of having a flat table to track arbitrary hierarchies,
we can use two different tables in the Cyborg database, one for physical
functions and one for virtual functions. physical_functions should have a
foreign key constraint to reference the id in Accelerators table. In addition,
virtual_functions should have a foreign key constraint to reference the id
in physical_functions.

The problems with this design are as follows. First, it can only track up to
3 hierarchies of resources. In case we need to add another layer, a lot of
migration work will be required. Second, even if we only need to add some new
attribute to the existing resource type, we need to create new migration
scripts for them. Overall the maintenance work is tedious.

Data model impact
-----------------
As discussed in previous sections, 5 table will be added: Devices::


    CREATE TABLE Devices
      (
        id                INTEGER NOT NULL ,     /*Primary Key*/
        uuid              VARCHAR2 (36 BYTE) ,   /*uuid v4 format for the device itself*/
        std_board_info    TEXT ,   /*A dictionary with standard fields*/
        vendor_board_info TEXT ,   /*A dictionary with driver-specific keys*/
        type              VARCHAR2 (30 BYTE)     /*Device Type*/
        vendor            VARCHAR2 (255 BYTE)    /*Device vendor*/
        model             VARCHAR2 (255 BYTE)    /*Device model*/
        hostname          VARCHAR2 (255 BYTE)     /*host name to identify which host this device is located*/
      ) ;
    ALTER TABLE Devices ADD CONSTRAINT Devices_PK PRIMARY KEY ( id ) ;

    CREATE TABLE AttachHandles
      (
        id               INTEGER NOT NULL ,     /*Primary Key*/
        attach_info      TEXT ,                 /*information needed to attach the accelerator to VMs*/
        device_id        INTEGER NOT NULL       /*foreign key references to the devices table*/
        handle_type      INTEGER NOT NULL ,     /*An enum to indicate the handle type, such as PCI, mdev, etc*/
      ) ;
    PRIMARY KEY (id),
    FOREIGN KEY (device_id) REFERENCES devices(id) ON
    DELETE RESTRICT ;

    CREATE TABLE DeviceProfiles
      (
        id               INTEGER NOT NULL ,     /*Primary Key*/
        uuid             VARCHAR2 (36 BYTE) ,   /*uuid v4 format for the DeviceProfile itself*/
        name             VARCHAR2 (32 BYTE) ,   /*Name of the DeviceProfile*/
        json             TEXT ,                 /*JSON blob with all the deivce/vendor specifc information*/
      ) ;

    CREATE TABLE ExtARQs
      (
        id               INTEGER NOT NULL ,     /*Primary Key*/
        uuid             VARCHAR2 (36 BYTE) ,   /*uuid v4 format for the ARQ itself*/
        state            VARCHAR2 (32 BYTE) ,   /*represents current state of the request*/
        device_profile_id    INTEGER NOT NULL     /*foreign key references to the device profile table*/
        hostname          VARCHAR2 (255 BYTE)     /*host name to identify which host this request is targeting*/
        device_rp_uuid   VARCHAR2 (36 BYTE) ,   /*uuid v4 format for the resource provider which this ARQ is pointing to*/
        instance_uuid    VARCHAR2 (36 BYTE) ,   /*uuid v4 format for the instance which this ARQ is pointing to*/
        attach_handle_id INTEGER NOT NULL       /*foreign key references to the attach handle table*/
      ) ;
    PRIMARY KEY (id),
    FOREIGN KEY (device_profile_id) REFERENCES DeviceProfiles(id),
    FOREIGN KEY (attach_handle_id) REFERENCES AttachHandles(id) ON
    DELETE RESTRICT ;

    CREATE TABLE ControlPathID
      (
        id               INTEGER NOT NULL ,     /*Primary Key*/
        type_name        VARCHAR2 (255 BYTE) ,  /*Name of the ControlPathID*/
        device_id        INTEGER NOT NULL ,     /*Foreign Key to point to the device*/
        json             TEXT ,                 /*JSON blob for type specific information*/
      ) ;

In addition, the Deployables and Accelerators will be changed to the following
scheme::

    CREATE TABLE Deployables
      (
        id           INTEGER NOT NULL ,     /*Primary Key*/
        parent_id    INTEGER ,              /*Pointer to the parent deployable's primary key*/
        root_id      INTEGER ,              /*Pointer to the root deployable's primary key*/
        num_accelerators   INTEGER ,        /*Number of accelerators contained in this deployable*/
        name         VARCHAR2 (32 BYTE) ,   /*Name of the deployable*/
        uuid         VARCHAR2 (36 BYTE) ,   /*uuid v4 format for the deployable itself*/
        device_id    INTEGER NOT NULL       /*foreign key references to the device table*/
      ) ;
    PRIMARY KEY (id),
    FOREIGN KEY (device_id) REFERENCES Devices(id) ON
    DELETE RESTRICT ;

Disclaimer: more fields may be added to specific tables and the schema may
evolve a little as the implementation progresses.

RPC API impact
---------------

Out of Scope for this spec

REST API impact
---------------

Out of Scope for this spec

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

Other deployment impacts
------------------------
None

Developer impact
----------------

There will be new functionalities available to the dev because of this work.


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  Zhenghao Wang <wangzh21@lenovo.com>
  Coco Gao <gaojh4@lenovo.com>

Work Items
----------
* Create migration scripts to add two more tables to the database
* Create models in sqlalchemy as well as related conductor APIs
* Create corresponding objects
* Create Conductor APIs to allow resource reporting


Dependencies
============

Testing
=======
* Unit tests will be added test Cyborg generic driver.

Documentation Impact
====================
Document FPGA Modelling in the Cyborg project

References
==========
.. [#device-profile-spec] `Specification for Device Profile <https://review.openstack.org/#/c/602978/>`_

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release
     - Description
   * - Stein
     - Introduced

