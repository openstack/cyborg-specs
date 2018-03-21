2. Edit the ``/etc/specs/specs.conf`` file and complete the following
   actions:

   * In the ``[database]`` section, configure database access:

     .. code-block:: ini

        [database]
        ...
        connection = mysql+pymysql://specs:SPECS_DBPASS@controller/specs
