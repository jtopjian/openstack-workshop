# Neutron

#### The OpenStack Networking Service

As we've seen, basic instance networking can be handed with `nova-network`. We also learned that Cinder used to be part of Nova known as `nova-volume`. Neutron follows the same pattern: in an attempt to provide OpenStack with a dedicated Networking Service that isn't tied to an existing OpenStack component, Neutron was created.

Unfortunately, Neutron did not start as a clone of `nova-network`. Because of that, there is no direct way to translate `nova-network` configurations to Neutron configurations. This is one of the main reasons for Neutron's low adoption rate. Additionally, Neutron documentation tries to get the user to configure a _very_ complex initial environment.

This overview of Neutron will use a very basic configuration that should lay the foundation for future, complex configurations.

We will use the following Neutron components:

  * neutron-server: accepts HTTP/REST api calls.
  * neutron-dhcp-agent: handles DHCP for the instances.
  * neutron-plugin-linuxbridge: uses the standard Linux Bridge as the network backend.

The standard Neutron architecture is documented as using three dedicated servers. For this exercise, we'll continue to use a single neutron server. Two dedicated servers are sufficient for the majority of environments.

## Installation

    $ sudo apt-get install neutron-server neutron-dhcp-agent neutron-plugin-openvswitch neutron-plugin-openvswitch-agent openvswitch-datapath-dkms

## Configuration

### Keystone

#### User

Create a Neutron Keystone user

#### Catalog Entry

We previously saw that Keystone came with a default catalog prepopulated with most services. Neutron was not one of those services, so we need to add it ourselves.

Edit `/etc/keystone/default_catalog.templates` and add the following:

    catalog.RegionOne.network.publicURL = http://localhost:9696
    catalog.RegionOne.network.adminURL = http://localhost:9696
    catalog.RegionOne.network.internalURL = http://localhost:9696
    catalog.RegionOne.network.name = Networking Service

### Neutron

Neutron consists of several configuration files.

#### /etc/neutron/neutron.conf

  * Search for `allow_overlapping_ips` and uncomment the setting.
  * Search for the `[database]` section and set the `connection` setting to the MySQL database
    * _note_: Neutron does not require a schema creation command.
  * Search and set the following RabbitMQ settings:
    * rabbit_host
    * rabbit_port
    * rabbit_userid
    * rabbit_password
  * Search for `keystone_authtoken` and configure the Keystone options for the appropriate settings.

#### /etc/neutron/dhcp_agent.ini

Search for and set the following:

    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = false

#### /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini

  * In the `[ovs]` section, set `tenant_network_type` to `local`

### Nova

Nova will require some reconfiguring so it can work with Neutron.

#### Tear-down

First, detach the volume and terminate the instance from the previous exercise:

    $ nova volume-detach my_vm <vol uuid>
    $ nova delete my_vm

Then delete the `nova-network` network that was created:

    $ sudo nova-manage network modify --disassociate-project 192.168.1.0/24
    $ sudo nova-manage network delete 192.168.1.0/24

Finally, remove `nova-network`

    $ sudo apt-get remove --purge nova-network

#### nova.conf

Then, edit `/etc/nova/nova.conf` and remove the previous `nova-network` settings:

    network_manager=nova.network.manager.FlatDHCPManager
    firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    network_size=254
    force_dhcp_release=True
    flat_network_bridge=br0
    flat_interface=eth0
    public_interface=eth0

Add in these settings in the `[DEFAULT]` section:

    network_api_class=nova.network.neutronv2.api.API
    neutron_url=http://localhost:9696
    neutron_auth_strategy=keystone
    neutron_admin_tenant_name=services
    neutron_admin_username=neutron
    neutron_admin_password=password
    neutron_admin_auth_url=http://localhost:35357/v2.0
    firewall_driver=nova.virt.firewall.NoopFirewallDriver

### Restart Services

Restart all Keystone, Nova, and Neutron services:

    $ sudo restart keystone

    $ for i in /etc/init.d/nova-*
    > do
    > sudo $i restart
    > done

    $ for i in /etc/init.d/neutron-*
    > do
    > sudo $i restart
    > done

### Verfication

The following command should not return an error:

    $ neutron net-list

## Network Creation

Now that Neutron is installed and configured with both Keystone and Nova, it's time to create a network. This step is similar to the `nova-network` step from the last section.

### Neutron

We will create a single subnet that all virtual machines will run on:

    $ neutron net-create --shared default
    $ neutron subnet-create default 192.168.1.0/24 --name default --allocation-pool start=192.168.1.100,end=192.168.1.200

### Server

Neutron uses a bridge called `br-int` to host the virtual machines. This is similar to the `br0` created earlier.

    $ sudo ovs-vsctl add-br br-int

## Instance Creation

With everything in place, create a new instance:

    $ nova boot --image CirrOS --flavor 1 my_neutron_vm

You can check the status of creation with:

    $ nova show my_neutron_vm

Once the instance is active, you should be able to ping and SSH into it.

## Questions?
