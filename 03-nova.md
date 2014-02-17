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

    $ sudo apt-get install nova-novncproxy novnc nova-api nova-ajax-console-proxy nova-cert nova-conductor nova-consoleauth nova-doc nova-scheduler python-novaclient

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

#### Neutron

To configure Nova to use Neutron for the network service, add the following to the `[DEFAULT]` section of `/etc/nova/nova.conf`:

    network_api_class = nova.network.neutronv2.api.API
    neutron_url = http://<ip of eth0>:9696
    neutron_auth_strategy = keystone
    neutron_admin_tenant_name = services
    neutron_admin_username = neutron
    neutron_admin_password = password
    neutron_admin_auth_url = http://<ip of eth0>:35357/v2.0
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    security_group_api = neutron
    service_neutron_metadata_proxy = True
    neutron_metadata_proxy_shared_secret = password
    libvirt_vif_type = ethernet
    libvirt_vif_driver = nova.virt.libvirt.vif.NeutronLinuxBridgeVIFDriver
    linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver

#### Keystone

To configure Nova to use Keystone, add the following to the `[DEFAULT]` section of `/etc/nova/nova.conf`:

    auth_strategy = keystone

Finally, open `/etc/nova/api-paste.ini` and go down to the bottom. Modify the Keystone authentication information as needed:

    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    auth_host = 127.0.0.1
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = services
    admin_user = nova
    admin_password = password

### Database Schema

Once `nova.conf` has been modified, run

    $ sudo nova-manage db sync

### Verifying

Once all of the above has been entered in to `nova.conf`, restart all Nova services:

    $ for i in /etc/init.d/nova-*
    > do
    > sudo $i restart
    > done

The following command will display the status of the Nova services:

    $ nova service-list

To confirm that Nova can talk to Glance, run the following:

    $ nova image-list

You should see your CirrOS image.

## Questions?
