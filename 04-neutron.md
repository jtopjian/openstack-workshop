# Neutron

#### The OpenStack Networking Service

We will use the following Neutron components:

  * neutron-server: accepts HTTP/REST api calls.
  * neutron-dhcp-agent: handles DHCP for the instances.
  * neutron-plugin-openvswitch-agent: uses Open vSwitch as the network backend.
  * neutron-plugin-ml2: The Neutron ML2 Plugin
  * neutron-metadata-agent: handles metadata proxying to the Nova metadata service
  * neutron-l3-agent: handles L3 routing such as virtual routers

The standard Neutron architecture is documented as using three dedicated servers: a cloud controller, a network controller, and a compute node. For this workshop, we'll use two servers: a cloud controller and compute node. The network node will be combined with the cloud controller.

## Installation

    $ sudo apt-get install neutron-server neutron-plugin-ml2 neutron-plugin-openvswitch-agent openvswitch-datapath-dkms neutron-l3-agent neutron-dhcp-agent

## Configuration

### Open vSwitch

Create a bridge for the NIC that acts as a trunk to all VLANs:

    $ ovs-vsctl add-br br-eth1
    $ ovs-vsctl add-port br-eth1 eth1

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

#### /etc/neutron/neutron.conf

    [DEFAULT]
    verbose = True
    state_path = /var/lib/neutron
    lock_path = $state_path/lock
    core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
    service_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
    allow_overlapping_ips = True
    rabbit_host = cc-ip
    notification_driver = neutron.openstack.common.notifier.rpc_notifier
    nova_region_name = RegionOne
    nova_admin_username = nova
    nova_admin_tenant_id = <tenant_id>
    nova_admin_password = password
    [quotas]
    [agent]
    root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
    [keystone_authtoken]
    auth_host = cc-ip
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = services
    admin_user = neutron
    admin_password = password
    signing_dir = $state_path/keystone-signing
    [database]
    connection = mysql://neutron:password@cc-ip/neutron
    [service_providers]
    service_provider=LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
    service_provider=VPN:openswan:neutron.services.vpn.service_drivers.ipsec.IPsecVPNDriver:default

#### /etc/neutron/dhcp_agent.ini

Search for and set the following:

    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = True

#### /etc/neutron/metadata_agent.ini

Search for and set the following:

    auth_url = http://localhost:5000/v2.0
    auth_region = RegionOne
    admin_tenant_name = services
    admin_user = neutron
    admin_password = password
    metadata_proxy_shared_secret = password

#### /etc/neutron/plugins/ml2/ml2_conf.ini

Add the following sections:

    [ml2]
    type_drivers = vlan
    tenant_network_types = vlan
    mechanism_drivers = openvswitch

    [ml2_type_vlan]
    network_vlan_ranges = trunk:3811:3819

    [securitygroup]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_security_group = True

    [ovs]
    bridge_mappings = trunk:br-eth1

### Restart Services

Restart all Keystone, Nova, and Neutron services:

    $ sudo restart keystone
    $ sudo restart neutron-server
    $ sudo restart neutron-l3-agent
    $ sudo restart neutron-dhcp-agent
    $ sudo restart neutron-metadata-agent
    $ sudo restart neutron-plugin-openvswitch-agent

### Verfication

The following command should not return an error:

    $ neutron net-list

## Network Creation

Now that Neutron is installed and configured with both Keystone and Nova, it's time to create a network.

We will create a single subnet that all virtual machines will run on:

    $ neutron net-create vlan-3811 --provider:physical_network=trunk --provider:network_type=vlan --provider:segmentation_id=3811 --shared
    $ neutron subnet-create vlan-3811 --name vlan-3811 --allocation-pool start=10.100.0.10,end=10.100.0.100 --dns-nameserver 8.8.8.8 10.100.0.0/24

If everything worked correctly, you should see your network listed when you do:

    $ neutron net-list

Additionally, you can run the following command:

    $ ip netns

And should see something like `qdhcp-f367d657-f5b0-4dd7-98c6-453342074a93`. This is a "network namespace" and is how Neutron can support overlapping IP addresses. Network namespaces are like virtual machines for the network layer.

Run the following command:

    $ ip netns exec qdhcp-xxx ip a

And you will see that an IP address of 192.168.1.100 has been added to an internal interface.

## Questions?
