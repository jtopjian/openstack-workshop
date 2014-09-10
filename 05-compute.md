# The Compute Node

At this point, the Cloud Controller has the minimum amount of services running to support running virtual machines. So we'll jump over to the Compute Node and configure it.

## Packages

Install `apt` tools and the Havana `apt` repo:

    $ sudo apt-get update
    $ sudo apt-get install python-software-properties
    $ sudo add-apt-repository cloud-archive:havana
    $ sudo apt-get update

## Nova

### Installation

    $ sudo apt-get install nova-compute-qemu

### Configuration

In `/etc/nova/nova.conf`, add the following settings to the `[DEFAULT]` section:

    auth_strategy = keystone

    rpc_backend = nova.rpc.impl_kombu
    rabbit_host = cc-ip
    rabbit_port = 5672
    rabbit_user = guest
    rabbit_password = guest

    image_service=nova.image.glance.GlanceImageService
    glance_api_servers=cc-ip:9292

    network_api_class = nova.network.neutronv2.api.API
    neutron_url = http://cc-ip:9696
    neutron_auth_strategy = keystone
    neutron_admin_tenant_name = services
    neutron_admin_username = neutron
    neutron_admin_password = password
    neutron_admin_auth_url = http://cc-ip:35357/v2.0
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    security_group_api = neutron
    service_neutron_metadata_proxy = True
    neutron_metadata_proxy_shared_secret = password
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver

    vnc_enabled=true
    vncserver_listen=0.0.0.0
    vncserver_proxyclient_address=compute-ip
    novncproxy_base_url=http://cc-ip:6080/vnc_auto.html

    [database]
    sql_connection = mysql://nova:password@cc-ip/nova

    [keystone_authtoken]
    auth_uri = http://cc-ip:5000
    auth_host = cc-ip
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = services
    admin_user = nova
    admin_password = password

### Restart

Once all of the above has been entered in to `nova.conf`, restart the `nova-compute` service:

    $ sudo /etc/init.d/nova-compute restart

## Neutron

### Installation

    $ sudo apt-get install -y neutron-plugin-openvswitch-agent

### Open vSwitch

Create a bridge for the NIC that acts as a trunk to all VLANs:

    $ ovs-vsctl add-br br-eth1
    $ ovs-vsctl add-port br-eth1 eth1


### Configuration

Edit `/etc/neutron/neutron.conf`:

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

Edit `/etc/neutron/plugins/ml2/ml2_conf.ini`:

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

### Restart

Restart the Neutron agent:

    $ sudo restart neutron-plugin-openvswitch-agent

## Launching an Instance!

At this point you can *finally* launch an instance. Jump back to the Cloud Controller and run the following command:

    $ nova boot --image CirrOS --flavor 1 myvm

You can check the status of the launch by running:

    $ nova show myvm

You can also watch the console of the instance:

    $ nova console-log myvm

If everything works, the console will show a login prompt. You can verify that the virtual machine is on the network by first allowing ICMP traffic:

    $ nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0


In order to log in, you must allow traffic to port 22:

    $ nova secgroup-add-rule default tcp 22 22 0.0.0.0/0

Next, execute SSH inside the network namespace:

    $ ip netns
    $ ip netns exec qdhcp-xxx ssh 192.168.1.101

## Questions?
