# Cinder

#### The OpenStack Block Storage Service

Cinder provides Block Storage. This can be in the form of attachable and detachable storage devices, the ability to snapshot those devices, and the ability to boot from those devices. The last feature is gaining a lot of popularity in OpenStack as it's the easiest way to create more featureful types of instances without having to heavily modify Nova.

Cinder used to be the part of Nova called `nova-volume`. When Folsom was released, the `nova-volume` code was copied one-for-one to Cinder. This allowed a very easy migration path from `nova-volume` to Cinder. It's also why Cinder's configuration is very similar to Nova.

Cinder contains the following components:

  * cinder-api: accepts HTTP/REST API calls.
  * cinder-scheduler: manages which server will host a block storage volume.
  * cinder-volume: launches and terminates volumes.

It's common to run all of these services on a single server, especially if you only have one central storage device. If you choose to utilize the free storage space on each compute node, you will need to run a `cinder-volume` service on each compute node.

For this exercise, all daemons will be run on the single server. We will use `/dev/vdc` as the storage device.

## Installation

    $ sudo apt-get install cinder-api cinder-scheduler cinder-volume

## Configuration

### Keystone

As usual, a Keystone user will need created.

#### Exercise

Create the Cinder Keystone user.

### Cinder

Just like with Nova, Cinder is configured using a single monolithic configuration file called `/etc/cinder/cinder.conf`.

#### Database

To configure Cinder to use MySQL for the database, add the `sql_connection` setting to the `[DEFAULT]` section of `/etc/cinder/cinder.conf`.

#### Exercise

Connect Cinder to MySQL (set sql_connection)

#### RabbitMQ

Add the following to the `[DEFAULT]` section of `/etc/cinder/cinder.conf`:

    rpc_backend = cinder.openstack.common.rpc.impl_kombu
    rabbit_host = localhost
    rabbit_port = 5672
    rabbit_userid = guest
    rabbit_password = guest

#### Keystone

Configure Cinder to connect to Keystone at the bottom of `/etc/cinder/api-paste.ini` with the appropriate values.

### Database Schema

Once `cinder.conf` has been modified, run the `cinder-manage` command to create the database schema.

    cinder-manage db sync

## Volume Configuration

The most basic type of volume configuration is to use a combination of <a href="http://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)">Linux LVM</a> and iSCSI.

### Server

Create a Physical Volume by doing:

    $ sudo pvcreate /dev/vdc

Next, a Volume Group will need created. The default VG name that Cinder expects is `cinder-volumes`:

    $ sudo vgcreate cinder-volumes /dev/vdc

### Cinder

Once the server is set up, just restart the Cinder services. Since LVM/iSCSI is the default storage backend, no modifications to Cinder are required:

    $ for i in /etc/init.d/cinder-*
    > do
    > $i restart
    > done

### Verifying

The following command should run without error:

    $ cinder list

To create a volume, do:

    $ cinder create --display-name my_vol 1

Then do:

    $ cinder list

You can see that a Logical Volume was successfully created:

    $ sudo lvs

You can also use Nova to interact with volumes:

    $ nova volume-list
    $ nova volume-delete my_vol
    $ nova volume-create --display-name my_vol 1

### Attaching a Volume

To attach a volume, you need to know:

  * The volume uuid
  * The server name or uuid which the volume will attach to
  * A device name or "auto"

To attach a volume to your running CirrOS server, do:

    $ nova volume-attach my_vm <vol uuid> auto

Note that Nova reported it will attach as `/dev/vdc`.

Now log in to your CirrOS image and do:

    $ sudo dmesg

You will see a message about `/dev/vdb`. OpenStack and KVM do not work well together with regard to correctly reporting the device name of attached volumes.

You can verify that `/dev/vdb` is the right volume by doing:

    $ fdisk -l /dev/vdb

And seeing that it is a 1gb volume.

## Exercises

  * Format /dev/vdb as ext4
  * Mount /dev/vdb
  * Write data to it
  * Unmount it
  * Detach it using `nova volume-detach`
  * Reattach it

## Questions?
