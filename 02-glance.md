# Glance

#### The OpenStack Image Service

Glance is a catalog and library of virtual machine templates, also known as "Images". It consists of daemons:

  * `glance-api`: Accepts Image API calls for image discovery, retrieval, and storage.
  * `glance-registry`: Stores, processes, and retrieves metadata about images. Metadata includes size, type, and so on.

Glance stores meta-data about the images in a database. MySQL is most commonly used.

The actual images are stored in a user-defined storage repository. The basic "file" backend is most commonly used. This will simply store the images under `/var/lib/glance/images`. Other backends such as object storage and RBD also exist.

## Installation

    $ sudo apt-get install glance

## Configuration

### Keystone

Source the admin `openrc` file as you need to be the Keystone admin to do the following.

Create a Keystone user for Glance:

    $ keystone user-create --name glance --tenant services --pass password --email root@localhost

Then make the `glance` user an admin:

    $ keystone user-role-add --user glance --tenant services --role admin

### Glance

The following configuration files will be modified:

  * `/etc/glance/glance-api.conf`
  * `/etc/glance/glance-registry.conf`

Coincidentally, both files will have the same changes made to them, so for each file, do the following:

  * Search for `sql_connection`.
  * Change the value to `mysql://glance:password@localhost/glance`.
  * Search for `keystone_authtoken` and change the following:
    * `admin_tenant_name = services`
    * `admin_user = glance`
    * `admin_password = password`
  * At the very bottom of the file, add:
    * `flavor = keystone`
  * Save and exit the file.

Once both files have been edited, restart both Glance services:

    $ sudo restart glance-api
    $ sudo restart glance-registry

Verify that the services restarted successfully by doing:

    $ ps aux | grep glance

And checking for both `glance-api` and `glance-registry`.

Create the database schema for Glance:

    $ glance-manage db_sync

Finally, verify that Glance is working by running the following command:

    $ glance index

An empty table should be returned.

## Adding Images to Glance

Now that Glance is running, add an image to it. For this workshop, an image known as CirrOS will be used. CirrOS is a very small Linux distribution that works well for testing purposes.

To add it to Glance, first download it:

    $ cd
    $ wget http://download.cirros-cloud.net/0.3.2~pre2/cirros-0.3.2~pre2-x86_64-disk.img

Next, note the _file type_ of the image:

    $ file cirros-0.3.2*

Notice how it's a QCOW v2 file. This will be important when adding the image to Glance:

    $ glance image-create --name CirrOS --disk-format qcow2 --container-format bare --is-public true < cirros-0.3.2*

If the command was successful, details about the image will be returned. You can verify that the image was uploaded to the default `file` backend by doing:

    $ ls /var/lib/glance/images

and seeing a single file.

You can also see that this file is the exact same as the file you downloaded:

    $ md5sum cirros-*
    $ sudo md5sum /var/lib/glance/images/*

### A Note About Unique IDs

One common pattern with OpenStack is to name _everything_ as a unique ID (UUID). A UUID has the form of what you see under `/var/lib/glance/images`. UUIDs can be a little hard to work with as they're both impossible to remember or type out. A lot of copying and pasting is done in OpenStack with UUIDs.

Sometimes, though, you can substitute a UUID for the canonical name _if_ the canonical name is unique. For example, try the following:

    $ glance index
    $ glance image-show CirrOS
    $ glance image-show <uuid>

## Exercise

  * Run `glance help image-download`
  * Download the CirrOS image.

## Questions?

## Exercise

