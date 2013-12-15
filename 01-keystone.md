# Keystone

#### The OpenStack Identity Service

Keystone performs the following functions:

  * User management. Tracks users and their permissions.
  * Service catalog. Provides a catalog of available services with their API endpoints.
Keystone provides the

## Installation

    $ sudo apt-get install keystone

Verify that the Havana version of Keystone was installed by doing:

    $ dpkg -l | grep keystone

and confirming that the version is `2013.2`.

## Configuration

The following changes will be done in `/etc/keystone/keystone.conf`. Use your choice of a text editor to edit the file.

### Admin Token

They Keystone Admin Token is similar to the root password on a Linux server. To set it, do the following:

  * Edit `/etc/keystone/keystone.conf`.
  * Search for `admin_token` (third line).
  * Uncomment it.
  * Set the value to a password of your choice:
    * `admin_token = password`

### Token Format

Keystone supports two types of tokens: UUID and PKI. UUID-based tokens are simple, short token strings. PKI tokens are full SSL-compatible tokens. While the PKI-based tokens give you the ability to tie into an existing PKI infrastructure, the UUID tokens are more simple and appropriate for basic environments.

  * In `/etc/keystone/keystone.conf`, search for `token_format`.
  * Change the value to `UUID`.

### Database

By default, Keystone uses a SQLite database located at `/var/lib/keystone/keystone.db`. In order to configure it to use MySQL, do the following:

  * In `/etc/keystone/keystone.conf`, search for `sql`.
  * Change the value of `connection` to `mysql://keystone:password@localhost/keystone`.
  * Save and exit the file.

Now the Keystone database schema needs to be created:

    $ keystone-manage db_sync

You can verify that the schema was created by doing the following:

    $ mysql -u root -p keystone
    mysql> show tables;


With the database and admin token, and token format configured, restart Keystone:

    $ sudo restart keystone

Verify there were no problems with Keystone starting:

    $ sudo tail /var/log/keystone/keystone.log

### Create Projects and Users

Keystone provides the user database for OpenStack. To start working with OpenStack, some projects, users, and roles will need to be created.

To begin, set some environment variables to make running the commands easier:

    $ export OS_SERVICE_TOKEN=your-admin-token
    $ export OS_SERVICE_ENDPOINT=http://localhost:35357/v2.0

Verify that connectivity to Keystone is working:

    $ keystone tenant-list

You should see a blank line returned.

#### Default Projects

There are usually two default projects created in OpenStack:

  * admin: Used by administrators
  * services: Used by the OpenStack components

Create them by doing the following:

    $ keystone tenant-create --name=admin --description="Admin Tenant"
    $ keystone tenant-create --name=services --description="Services Tenant"

You should now see two results when running:

    $ keystone tenant-list

#### Default Users

Now create the `admin` user:

    $ keystone user-create --name admin --tenant admin --pass password --email root@localhost

#### Default Roles

Roles are not used often in OpenStack as there are really only two roles possible:

  * admin: administrator of the cloud
  * member: a regular user

To create the admin role, do the following:

    $ keystone role-create --name admin

You can see that a `_member_` role already exists:

    $ keystone role-list

`_member_` has a special format to denote that it's an internal role.

Finally, grant the `admin` user the `admin` role in the `admin` tenant:

    $ keystone user-role-add --user admin --tenant admin --role admin

## The Keystone Service Catalog

Keystone stores a catalog of services as well as a user database. This allows users to query Keystone for what cloud services are available and where to find them.

The catalog can either be stored in the MySQL database or in a text file. While most documentation will show how to store the catalog in MySQL, storing the catalog as a text file is actually much easier and a lot less resource intensive.

To create the text-based catalog file:

  * Open `/etc/keystone/keystone.conf`.
  * Search for `catalog`.
  * Comment out the first `driver` entry.
  * Uncomment the second `driver` entry a few lines below the first.
  * Save and exit the file.

You'll notice that `/etc/keystone/default_catalog.templates` already exists and includes the basic OpenStack services.

Restart Keystone and verify the catalog works by doing:

    $ keystone catalog

## Create an openrc File

The `openrc` file is the standard authentication file for OpenStack. While it's optional, it's very useful as it allows you to:

  * Run shorter commands
  * Run commands as several different OpenStack users

To create the file, use any text editor to add the following contents to a file called `openrc`:

    export OS_AUTH_URL=http://localhost:35357/v2.0/
    export OS_REGION_NAME=RegionOne
    export OS_USERNAME=admin
    export OS_TENANT_NAME=admin
    export OS_PASSWORD=password
    export OS_NO_CACHE=1

Next, unset the previous environment variables:

    $ unset OS_SERVICE_TOKEN
    $ unset OS_SERVICE_ENDPOINT

and finally, source the `openrc` file:

    $ source openrc

Verify it works by doing:

    $ keystone user-list

## Exercises

  * Create a Keystone Project and User for yourself.
  * Copy `openrc` to `<username>rc`.
  * Edit the new `rc` file with the values for the account you just created.
  * Verify it works by running:
    * `keystone token-get`
  * Note that you cannot run commands such as `keystone tenant-list`. Why not?

## Questions?
