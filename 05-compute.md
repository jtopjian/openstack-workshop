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

    sql_connection mysql://nova:password@cc-ip/nova
    rpc_backend=nova.rpc.impl_kombu
    rabbit_host=cc-ip
    rabbit_user=guest
    rabbit_password=guest
    image_service=nova.image.glance.GlanceImageService
    glance_api_servers=cc-ip:9292
    auth_strategy=keystone
    network_api_class=nova.network.neutronv2.api.API
    neutron_url=http://cc-ip:9696
    neutron_auth_strategy=keystone
    neutron_admin_tenant_name=services
    neutron_admin_username=neutron
    neutron_admin_password=password
    neutron_admin_auth_url=http://cc-ip:35357/v2.0
    firewall_driver=nova.virt.firewall.NoopFirewallDriver
    security_group_api=neutron
    service_neutron_metadata_proxy=True
    neutron_metadata_proxy_shared_secret=password
    libvirt_vif_type=ethernet
    libvirt_vif_driver=nova.virt.libvirt.vif.NeutronLinuxBridgeVIFDriver
    linuxnet_interface_driver=nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver

### Restart

Once all of the above has been entered in to `nova.conf`, restart the `nova-compute` service:

    $ sudo /etc/init.d/nova-compute restart

## Neutron

### Installation

    $ sudo apt-get install -y neutron-plugin-linuxbridge-agent

### Configuration

Edit `/etc/neutron/neutron.conf`:

    [DEFAULT]
    core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
    rabbit_host = <cc ip>
    rabbit_port = 5672
    rabbit_userid = guest
    rabbit_password = guest

Edit `/etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini`:

    [vxlan]
    enable_vxlan = true
    local_ip = <c01 ip>

    [securitygroup]
    firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

### Restart

Restart the Neutron agent:

    $ sudo /etc/init.d/neutron-plugin-linuxbridge-agent restart

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
