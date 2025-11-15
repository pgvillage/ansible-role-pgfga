PgFga
=========

PgFga is a tool to manage PostgreSQL resources, such as databases, extensions, users and roles.
One of the fine things is that PgFga can manage users from LDAP groups.
This role installs PgFga and runs it as a cronjob, so that periodically, PostgreSQL LDAP users are refreshed.
This role is part of PgVillage, which is an opinated PostgreSQL deployment for Virtual Machines.

Requirements
------------

This role aims at using an RPM from the MannemSolutions repo.

Role Variables
--------------

Please see [defaults](https://github.com/pgvillage/ansible-role-pgfga/blob/main/defaults/main.yml) for all variables


Dependencies
------------

No dependencies


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - pgvillage.pgfga

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
