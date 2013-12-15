# Horizon

#### The OpenStack Dashboard

Horizon provides a web-based dashboard to work with OpenStack.

## Installation

    $ sudo apt-get install openstack-dashboard

## Configuration

### Ubuntu Theme

If you wanted to, you could remove the Ubuntu theme that is automatically installed:

    $ sudo apt-get remove --purge openstack-dashboard-ubuntu-theme

### Console Access

To enable VNC console access from Horizon, add the following in the `[DEFAULT]` section to `/etc/nova/nova.conf`:

    vnc_enabled=true
    novncproxy_base_url=http://your-floating-ip:6080/vnc_auto.html

## Questions?
