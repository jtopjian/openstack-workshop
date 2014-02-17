# Horizon

#### The OpenStack Dashboard

Horizon provides a web-based dashboard to work with OpenStack.

## Installation

    $ sudo apt-get install openstack-dashboard

## Configuration

### Ubuntu Theme

If you wanted to, you could remove the Ubuntu theme that is automatically installed:

    $ sudo apt-get purge openstack-dashboard-ubuntu-theme

### Console Access

To enable VNC console access from Horizon, add the following in the `[DEFAULT]` section to `/etc/nova/nova.conf` on the compute node:

    vnc_enabled=true
    vncserver_listen=0.0.0.0
    vncserver_proxyclient_address=c01-ip
    novncproxy_base_url=http://cc-ip:6080/vnc_auto.html

Then restart the `nova-compute` service:

    $ sudo /etc/init.d/nova-compute restart

### Test

In your browser go to [http://cc-ip/horizon](http://cc-ip/horizon) and login using `admin` and `password`.


## Questions?
