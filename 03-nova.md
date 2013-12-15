# Nova

#### The OpenStack Compute Service

Nova is the heart of OpenStack. It provides the functionality to host virtual machines in a cloud environment. Nova is broken up into several daemons:

  * nova-api: accepts HTTP/REST API calls.
  * nova-scheduler: manages which server will host a virtual machine.
  * nova-conductor: new daemon to isolate database access to a single server.
  * nova-consoleauth: provides transparent authentication to the virtual machine's console.
  * nova-novncproxy: provides VNC support to the virtual machine's console.
  * nova-cert: provides SSL connectivity to the virtual machine's console.
  * nova-network: provides network connectivity between virtual machines and the outside. deprecated.
  * nova-compute: launches and terminates virtual machines.

It's common to run all but the last daemon on a single server known as the Cloud Controller. `nova-compute` is then run on one or more servers that will host the actual virtual machines.

For this exercise, all daemons will be run on a single server. This is known as an all-in-one configuration.

## Installation

    $ sudo apt-get install nova-novncproxy novnc nova-api nova-ajax-console-proxy nova-cert nova-conductor nova-consoleauth nova-doc nova-scheduler nova-network nova-compute-qemu python-novaclient

## Configuration

### Keystone

Just like with Glance, create a Keystone user for Nova:

    $ keystone user-create --name nova --tenant services --pass password --email root@localhost
    $ keystone user-role-add --user nova --tenant services --role admin

### Nova

Nova is configured using a single monolithic configuration file called `/etc/nova/nova.conf`.

_note_: The exception to this is with the Ubuntu packages which install a secondary file for `nova-compute` at `/etc/nova/nova-compute.conf`. When the `nova-compute` daemon starts, both files are read.

#### Database

To configure Nova to use MySQL for the database, add the following to the `[DEFAULT]` section of `/etc/nova/nova.conf`:

    sql_connection = mysql://nova:password@localhost/nova

#### RabbitMQ

To configure Nova to use RabbitMQ for the messaging/queuing service, add the following to the `[DEFAULT]` section of `/etc/nova/nova.conf`:

    rpc_backend = nova.rpc.impl_kombu
    rabbit_host = localhost
    rabbit_port = 5672
    rabbit_user = guest
    rabbit_password = guest

#### Glance

To configure Nova to use Glance for the image service, add the following to the `[DEFAULT]` section of `/etc/nova/nova.conf`:

    image_service=nova.image.glance.GlanceImageService
    glance_api_servers=localhost:9292

#### Keystone

To configure Nova to use Keystone, add the following to the `[DEFAULT]` section of `/etc/nova/nova.conf`:

    auth_strategy = keystone

And add the following to the bottom of `/etc/nova/nova.conf`:

    [keystone_authtoken]
    auth_host = localhost
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = services
    admin_user = nova
    admin_password = password

Finally, open `/etc/nova/api-paste.ini` and go down to the bottom. Modify the Keystone authentication information as needed.

### Database Schema

Once `nova.conf` has been modified, run

    $ sudo nova-manage db sync

### Verifying

Once all of the above has been entered in to `nova.conf`, restart all Nova services:

    $ for i in `ls /etc/init.d/nova-*`
    > do
    > sudo /etc/init.d/$i restart
    > done

The following command will display the status of the Nova services:

    $ nova service-list

To confirm that Nova can talk to Glance, run the following:

    $ nova image-list

You should see your CirrOS image.

## Nova Networking

Neutron is the new networking component for OpenStack. Originally, the networking component was handled by `nova-network`. We will get into Neutron later. For now, we'll create a simple network with `nova-network`.

### Server Configuration

To begin, you will need to create a bridge interface. Copy the following into a file called `bridge.sh`:

    cat <<EOF >/etc/network/interfaces
    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet manual

    auto br0
    iface br0 inet dhcp
        bridge_ports eth0
        bridge_stp off
        bridge_fd 0
        bridge_maxwait 0
    EOF

Then run the command:

    $ sudo bash bridge.sh

Once complete, restart networking:

    $ sudo /etc/init.d/networking restart

If everything completed successfully, you will still have connectivity to your server and you will have a new interface called `br0`:

    $ ip a | grep br0

Additionally, you will have a bridge named `br0`:

    $ brctl show

### Nova Configuration

Add the following to the `[DEFAULT]` section of `/etc/nova/nova.conf`:

    network_manager=nova.network.manager.FlatDHCPManager
    firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    network_size=254
    force_dhcp_release=True
    flat_network_bridge=br0
    flat_interface=eth0
    public_interface=eth0

### Network Creation

The final step is to create a virtual network that your virtual machines will communicate on:

    $ source openrc
    $ sudo nova-manage network create nova --fixed_range_v4=192.168.1.0/24 --bridge_interface=eth0 --bridge=br0

Then restart `nova-network`:

    $ sudo restart nova-network

## Launching an Instance

At this point, we have Keystone providing authentication, Glance providing images, and Nova providing virtual machine orchestration. This is the minimum needed to start launching virtual machines in an IaaS environment. Let's try!

    $ nova image-list
    $ nova boot --image <uuid> --flavor 1 my_vm

You can check the status of your vm a few different ways:

    $ nova show my_vm
    $ sudo tail -f /var/log/nova/nova-compute.log

If your instance fails to launch, the latter command will give you more insight as to why.

Once your instance shows a status of `ACTIVE`, you can see the console output by doing:

    $ nova console-log my_vm

If all is successful, you can ping and SSH to your instance:

    $ ping -c 1 192.168.1.13
    $ ssh cirros@192.168.1.3

## Security Groups

Note that you were able to ping and SSH to your instance even though you did not set up a security group. The CirrOS image bypasses security groups. Besides the CirrOS image having a standard default password, the lack of security group feature is another reason why it should not be used in production.

### Exercises

To create add rules to your default security group on the command line, do the following:

    $ nova help | grep secgroup
    $ nova help secgroup-list
    $ nova help secgroup-list-rules
    $ nova help secgroup-add-rule

## Questions?
