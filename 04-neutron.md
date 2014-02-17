# Neutron

#### The OpenStack Networking Service

We will use the following Neutron components:

  * neutron-server: accepts HTTP/REST api calls.
  * neutron-dhcp-agent: handles DHCP for the instances.
  * neutron-plugin-linuxbridge: uses the standard Linux Bridge as the network backend.
  * neutron-metadata-agent: handles metadata proxying to the Nova metadata service
  * neutron-l3-agent: handles L3 routing such as virtual routers

The standard Neutron architecture is documented as using three dedicated servers: a cloud controller, a network controller, and a compute node. For this workshop, we'll use two servers: a cloud controller and compute node. The network node will be combined with the cloud controller.

## Installation

    $ sudo apt-get install neutron-server neutron-dhcp-agent neutron-plugin-linuxbridge neutron-metadata-agent neutron-l3-agent neutron-plugin-linuxbridge-agent

## Configuration

### Keystone

#### User

Create a Neutron Keystone user

    $ keystone user-create --name neutron --tenant services --pass password --email root@localhost
    $ keystone user-role-add --user neutron --tenant services --role admin

#### Catalog Entry

We previously saw that Keystone came with a default catalog prepopulated with most services. Neutron was not one of those services, so we need to add it ourselves.

Edit `/etc/keystone/default_catalog.templates` and add the following:

    catalog.RegionOne.network.publicURL = http://<eth0 ip>:9696
    catalog.RegionOne.network.adminURL = http://<eth0 ip>:9696
    catalog.RegionOne.network.internalURL = http://<eth0 ip>:9696
    catalog.RegionOne.network.name = Networking Service

### Neutron

Neutron consists of several configuration files.

#### /etc/default/neutron-server

    NEUTRON_PLUGIN_CONFIG="/etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini"

#### /etc/neutron/neutron.conf

Search for and set the following:

    [DEFAULT]
    core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
    allow_overlapping_ips = True
    rabbit_host = <cc ip>
    rabbit_port = 5672
    rabbit_userid = guest
    rabbit_password = guest

    [database]
    connection = mysql://neutron:password@localhost/neutron

    [keystone_authtoken]
    fill in the usual values

#### /etc/neutron/dhcp_agent.ini

Search for and set the following:

    interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = True
    enable_isolated_metadata = True
    enable_metadata_network = True

#### /etc/neutron/metadata_agent.ini

Search for and set the following:

    auth_url = http://<cc ip>:5000/v2.0
    auth_region = RegionOne
    admin_tenant_name = services
    admin_user = neutron
    admin_password = password
    metadata_proxy_shared_secret = password

#### /etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini

Add the following sections:

    [ml2]
    type_drivers = vxlan
    mechanism_drivers = linuxbridge
    tenant_network_types = vxlan

    [ml2_type_vxlan]
    vni_ranges = 1:1000

Search for and change the following:

    [vxlan]
    enable_vxlan = true
    local_ip = <cc ip>

    [securitygroup]
    firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

### Restart Services

Restart all Keystone, Nova, and Neutron services:

    $ sudo restart keystone

    $ for i in /etc/init.d/neutron-*
    > do
    > sudo $i restart
    > done

### Verfication

The following command should not return an error:

    $ neutron net-list

## Network Creation

Now that Neutron is installed and configured with both Keystone and Nova, it's time to create a network.

We will create a single subnet that all virtual machines will run on:

    $ neutron net-create --shared default
    $ neutron subnet-create default 192.168.1.0/24 --name default --no-gateway --allocation-pool start=192.168.1.100,end=192.168.1.200

If everything worked correctly, you should see your network listed when you do:

    $ neutron net-list

Additionally, you can run the following command:

    $ ip netns

And should see something like `qdhcp-f367d657-f5b0-4dd7-98c6-453342074a93`. This is a "network namespace" and is how Neutron can support overlapping IP addresses. Network namespaces are like virtual machines for the network layer.

Run the following command:

    $ ip netns exec qdhcp-xxx ip a

And you will see that an IP address of 192.168.1.100 has been added to an internal interface.

## Questions?
