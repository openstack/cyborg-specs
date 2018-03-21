Prerequisites
-------------

Before you install and configure the cyborg-specs service,
you must create a database, service credentials, and API endpoints.

#. To create the database, complete these steps:

   * Use the database access client to connect to the database
     server as the ``root`` user:

     .. code-block:: console

        $ mysql -u root -p

   * Create the ``specs`` database:

     .. code-block:: none

        CREATE DATABASE specs;

   * Grant proper access to the ``specs`` database:

     .. code-block:: none

        GRANT ALL PRIVILEGES ON specs.* TO 'specs'@'localhost' \
          IDENTIFIED BY 'SPECS_DBPASS';
        GRANT ALL PRIVILEGES ON specs.* TO 'specs'@'%' \
          IDENTIFIED BY 'SPECS_DBPASS';

     Replace ``SPECS_DBPASS`` with a suitable password.

   * Exit the database access client.

     .. code-block:: none

        exit;

#. Source the ``admin`` credentials to gain access to
   admin-only CLI commands:

   .. code-block:: console

      $ . admin-openrc

#. To create the service credentials, complete these steps:

   * Create the ``specs`` user:

     .. code-block:: console

        $ openstack user create --domain default --password-prompt specs

   * Add the ``admin`` role to the ``specs`` user:

     .. code-block:: console

        $ openstack role add --project service --user specs admin

   * Create the specs service entities:

     .. code-block:: console

        $ openstack service create --name specs --description "cyborg-specs" cyborg-specs

#. Create the cyborg-specs service API endpoints:

   .. code-block:: console

      $ openstack endpoint create --region RegionOne \
        cyborg-specs public http://controller:XXXX/vY/%\(tenant_id\)s
      $ openstack endpoint create --region RegionOne \
        cyborg-specs internal http://controller:XXXX/vY/%\(tenant_id\)s
      $ openstack endpoint create --region RegionOne \
        cyborg-specs admin http://controller:XXXX/vY/%\(tenant_id\)s
