# More Swift

#### More of the internals of Swift

In the interest of time our swift cluster has largely been created ahead of time but we will go through creating a new node and adding it to the cluster.

How to make a Swift cluster:

DETAILS HERE ABOUT DESIGN CONSIDERATIONS (see http://docs.openstack.org/havana/install-guide/install/apt/content/object-storage-network-planning.html - Swift is very chatty)

## Installation From Scratch

First we need to set up our pre-requisites from OpenStack. In our cluster demo we're using volumes created by our cloud controller to be used like hard drives for our virtual swift cluster.

We need to create 7 instances: 1 controller (keystone) and then 5/6 will host Swift. These steps are already done on our controller and 5 of these swift nodes are already set-up.

Each of the instances will require a volume to use as storage for Swift. (`/dev/vdb`)

There are instructions on how to do it with the source on http://docs.openstack.org/developer/swift/development_saio.html but we're going to use the packages.

In each instance the following steps need to be taken:

    sudo apt-get install python-software-properties
    sudo add-apt-repository cloud-archive:havana
    sudo apt-get update
    sudo apt-get upgrade

### Controller

On the controller follow the instructions on `00-prereqs.md` and `01-keystone.md`. (You can ignore the glance, nova, cinder, and neutron users if you wish). For storage nodes skip to the next section.

Then we want to install Swift.

    sudo apt-get install swift-proxy memcached python-swiftclient python-webob xfsprogs

Next on the controller you need to tell Keystone about the Object Storage service:

    keystone user-create --name swift --tenant services --pass password --email root@localhost
    keystone user-role-add --user swift --tenant services --role admin

Obtain admin's tenant ID by running:

    keystone tenant-list

Then add the following to`/etc/keystone/default_catalog.templates`, change `$TENANT_ID` to the tenant ID (eg. `AUTH_d196fd3acba649739e5dfbe5ebce108f`) and be sure to change `localhost` to the correct IP. (Floating IP)

    catalog.RegionOne.object_store.name = Swift Service
    catalog.RegionOne.object_store.publicURL = http://localhost:8080/v1/AUTH_$TENANT_ID
    catalog.RegionOne.object_store.adminURL = http://localhost:8080/
    catalog.RegionOne.object_store.internalURL = http://localhost:8080/v1/AUTH_$TENANT_ID

And restart Keystone

    service keystone restart

Next step is to get memcache to run on the local IP and not localhost: edit `/etc/memcached.conf` and change `-l 127.0.0.0.1` to `-l LOCAL_IP`. For example `-l 10.0.0.48` and restart memcache.

    service memcached restart

A couple housekeeping steps to create folders:

    mkdir -p /var/cache/swift
    mkdir -p /var/swift/keystone-signing
    chown -R swift:swift /var/swift/keystone-signing
    chown -R swift:swift /var/cache/swift

Then create the `proxy-server.conf` file. Edit the IP addresses in the below text or set $PROXY_LOCAL_NET_IP:

    export PROXY_LOCAL_NET_IP=10.0.0.148

    cat >/etc/swift/proxy-server.conf <<EOF
    [DEFAULT]
    bind_port = 8080
    workers = 8
    user = swift

    [pipeline:main]
    pipeline = healthcheck proxy-logging cache authtoken keystoneauth staticweb proxy-server

    [app:proxy-server]
    use = egg:swift#proxy
    allow_account_management = true
    account_autocreate = true

    [filter:proxy-logging]
    use = egg:swift#proxy_logging

    [filter:keystoneauth]
    use = egg:swift#keystoneauth
    operator_roles = Member,admin,swiftoperator

    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    # Delaying the auth decision is required to support token-less
    # usage for anonymous referrers ('.r:*').
    delay_auth_decision = true

    # cache directory for signing certificate
    signing_dir = /var/swift/keystone-signing

    # auth_* settings refer to the Keystone server
    auth_protocol = http
    auth_uri = http://$PROXY_LOCAL_NET_IP:5000/
    auth_host = $PROXY_LOCAL_NET_IP
    auth_port = 35357

    admin_token = epassword

    # the service tenant and swift userid and password created in Keystone
    admin_tenant_name = services
    admin_user = swift
    admin_password = password

    [filter:healthcheck]
    use = egg:swift#healthcheck

    [filter:cache]
    use = egg:swift#memcache
    memcache_servers = $PROXY_LOCAL_NET_IP:11211

    [filter:staticweb]
    use = egg:swift#staticweb
    # Seconds to cache container x-container-meta-web-* header values.
    # cache_timeout = 300
    # You can override the default log routing for this filter here:
    # set log_name = staticweb
    # set log_facility = LOG_LOCAL0
    # set log_level = INFO
    # set access_log_name = staticweb
    # set access_log_facility = LOG_LOCAL0
    # set access_log_level = INFO
    # set log_headers = False
    EOF

We next need to set some hashes specific to our installation:

    cat >/etc/swift/swift.conf <<EOF
    [swift-hash]
    # random unique strings that can never change (DO NOT LOSE)
    swift_hash_path_prefix = `od -t x8 -N 8 -A n </dev/random`
    swift_hash_path_suffix = `od -t x8 -N 8 -A n </dev/random`
    EOF

Copy the resulting `swift.conf` to the other nodes using `scp`. In our setup it requires an extra step (copy from Swift001 to Keystone, and from Keystone to all the rest)

    mkdir -p /var/run/swift
    chown swift:swift /var/run/swift
    mkdir -p /var/cache/swift /srv/node/
    chown swift:swift /var/cache/swift

Back on the `proxy-server` (controller node), we will start making the rings. We don't expect more than 10 drives, 3 copies, and don't want to rebalance the rings for another hour.

    cd /etc/swift
    swift-ring-builder account.builder create 9 3 1
    swift-ring-builder container.builder create 9 3 1
    swift-ring-builder object.builder create 9 3 1

Create our ring: Since we have 5 nodes: (*be sure to set the IPs correctly!*)

    swift-ring-builder account.builder add z1-10.0.0.151:6012/vdb1 100
    swift-ring-builder container.builder add z1-10.0.0.151:6011/vdb1 100
    swift-ring-builder object.builder add z1-10.0.0.151:6010/vdb1 100
    swift-ring-builder account.builder add z2-10.0.0.152:6012/vdb1 100
    swift-ring-builder container.builder add z2-10.0.0.152:6011/vdb1 100
    swift-ring-builder object.builder add z2-10.0.0.152:6010/vdb1 100
    swift-ring-builder account.builder add z3-10.0.0.157:6012/vdb1 100
    swift-ring-builder container.builder add z3-10.0.0.157:6011/vdb1 100
    swift-ring-builder object.builder add z3-10.0.0.157:6010/vdb1 100
    swift-ring-builder account.builder add z4-10.0.0.153:6012/vdb1 100
    swift-ring-builder container.builder add z4-10.0.0.153:6011/vdb1 100
    swift-ring-builder object.builder add z4-10.0.0.153:6010/vdb1 100
    swift-ring-builder account.builder add z5-10.0.0.154:6012/vdb1 100
    swift-ring-builder container.builder add z5-10.0.0.154:6011/vdb1 100
    swift-ring-builder object.builder add z5-10.0.0.154:6010/vdb1 100

If you're setting up node 6 ahead of time:

    swift-ring-builder account.builder add z6-10.0.0.156:6012/vdb1 100
    swift-ring-builder container.builder add z6-10.0.0.156:6011/vdb1 100
    swift-ring-builder object.builder add z6-10.0.0.156:6010/vdb1 100

Verify, then rebalance then verify again:

    swift-ring-builder account.builder
    swift-ring-builder container.builder
    swift-ring-builder object.builder

    swift-ring-builder account.builder rebalance
    swift-ring-builder container.builder rebalance
    swift-ring-builder object.builder rebalance

    swift-ring-builder account.builder
    swift-ring-builder container.builder
    swift-ring-builder object.builder

The ring needs to be copied to `/etc/swift` on each of the nodes. In our case it's done by running the following on the controller:
    scp -i /root/swift.pem *.gz ubuntu@10.0.0.151:
    scp -i /root/swift.pem *.gz ubuntu@10.0.0.152:
    scp -i /root/swift.pem *.gz ubuntu@10.0.0.157:
    scp -i /root/swift.pem *.gz ubuntu@10.0.0.153:
    scp -i /root/swift.pem *.gz ubuntu@10.0.0.154:

Then running the following on each storage node:
    mv /home/ubuntu/*.gz /etc/swift
    swift-init all restart

This is where our installation of the controller ends.

### Storage Nodes

Install Swift:

    sudo apt-get install swift swauth swift-account swift-container swift-object swift-proxy memcached python-keystoneclient python-swiftclient python-webob xfsprogs

Now we set up our volume to be used:

    sudo mkfs.xfs /dev/vdb
    echo "/dev/vdb /srv/node/vdb1 xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" | sudo tee -a /etc/fstab
    sudo mkdir -p /srv/node/vdb1
    sudo mount /srv/node/vdb1
    chown -R swift:swift /srv/node/vdb1

Create `rsyncd.conf` (change IPs as necessary)

    sed -i 's/RSYNC_ENABLE=false/RSYNC_ENABLE=true/g' /etc/default/rsync

    export STORAGE_LOCAL_NET_IP=10.0.0.PLACE_IP_HERE

    cat >/etc/rsyncd.conf <<EOF
    uid = swift
    gid = swift
    log file = /var/log/rsyncd.log
    pid file = /var/run/rsyncd.pid
    address = $STORAGE_LOCAL_NET_IP

    [account]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/account.lock

    [container]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/container.lock

    [object]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/object.lock
    EOF

And restart since we've changed a config file.

    service rsync restart

Now we create (replacing the auto installed ones) our account, container and object config files:

    cat >/etc/swift/account-server.conf <<EOF
    [DEFAULT]
    bind_ip = $STORAGE_LOCAL_NET_IP
    bind_port = 6012
    workers = 2
    recon_cache_path = /var/cache/swift

    [pipeline:main]
    pipeline = recon account-server

    [app:account-server]
    use = egg:swift#account

    [filter:recon]
    use = egg:swift#recon

    [account-replicator]

    [account-auditor]

    [account-reaper]
    EOF

    cat >/etc/swift/container-server.conf <<EOF
    [DEFAULT]
    bind_ip = $STORAGE_LOCAL_NET_IP
    bind_port = 6011
    workers = 2
    recon_cache_path = /var/cache/swift

    [pipeline:main]
    pipeline = recon container-server

    [app:container-server]
    use = egg:swift#container

    [filter:recon]
    use = egg:swift#recon

    [container-replicator]

    [container-updater]

    [container-auditor]

    [container-sync]
    EOF

    cat >/etc/swift/object-server.conf <<EOF
    [DEFAULT]
    bind_ip = $STORAGE_LOCAL_NET_IP
    bind_port = 6010
    workers = 2
    recon_cache_path = /var/cache/swift

    [pipeline:main]
    pipeline = recon object-server

    [app:object-server]
    use = egg:swift#object

    [filter:recon]
    use = egg:swift#recon

    [object-replicator]

    [object-updater]

    [object-auditor]
    EOF

Then fire it up!

    swift-init all restart

Then we test.

    swift-recon --all

    swift --debug stat

A couple of our previous day to day tests:

    export userID=1

    swift post myContainer$userID
    echo 'Hello World' > helloWorld.txt;
    swift upload myContainer$userID helloWorld.txt
    echo 'My Container Information:'; swift stat myContainer$userID
    echo 'File Information:'; swift stat myContainer$userID helloWorld.txt
    swift download myContainer$userID helloWorld.txt -o -
    swift post -m "X-Object-Meta-Hello: World" myContainer$userID helloWorld.txt
    swift stat myContainer$userID helloWorld.txt
    swift post -r '.r:*' myContainer$userID
    swift post -m 'web-listings: true' myContainer$userID
    cat <<EOF > listings.css
    From Toni @dev.hackers.fi (http://redmine.lighttpd.net/boards/3/topics/5418)
    @import url(http://fonts.googleapis.com/css?family=Raleway:200,400,600);
    body, html { background: #222; margin:0; }
    html { font: 14px/1.4 Raleway, 'Helvetica Neue', Helvetica, sans-serif; color: #ddd; font-weight: 400; }

    h2 { font-weight: 200; font-size: 45px; margin: 20px 35px; }

    div.list { background: #111; padding: 20px 35px; }
    div.foot { color: #777; margin-top: 15px; padding: 20px 35px; }

    td { padding: 0 20px; line-height: 21px; }
    tr:hover { background: black; }

    a { color: #32C6FF; }
    a:visited { color: #BD32FF; }
    a:hover { color: #B8EBFF; }
    EOF

    swift upload myContainer$userID listings.css
    swift post -m 'web-listings-css:listings.css' myContainer$userID
    keystone catalog --service object-store | grep '199' | awk '{print $4}' | tail -n 1

## Replication

    001 => 10.0.0.151
    002 => 10.0.0.152
    003 => 10.0.0.157
    004 => 10.0.0.153
    005 => 10.0.0.154

On two nodes run:

    watch -n 1 ls /srv/nodes/objects/

On the controller run:

    for x in {1..30}; do echo "$x" > $x.txt; swift upload myContainer$userID $x.txt; done

    swift-get-nodes /etc/swift/object.ring.gz admin myContainer$userID helloWorld.txt

Switch to one of the nodes and look at `/srv/node/vdb1/` at that location. `cat` the file. The filename is the timestamp of when the file was uploaded.

## Large/Segmented Objects

Segmented Objects allow you to do objects larger than the normal object storage size limit (5GB). It uses the `-S` flag with the number of bytes to split up. (eg. 1 MB = 1048576 byes). We're making a 10 MB file separated in 10 1 MB chunks

    dd if=/dev/zero of=largeObject bs=1M count=10
    swift upload myContainer$userID largeObject -S 1048576

    swift list

It creates a new container based off your first container called name_segments. The individual pieces are stored in that container.

    swift list myContainer$userID\_segments

    swift-get-nodes /etc/swift/object.ring.gz admin myContainer$userID largeObject
    swift-get-nodes /etc/swift/object.ring.gz admin myContainer$userID\segments largeObject/1387474552.389682/10485760/1048576/00000000

## Quarantine/Corruption

    swift-get-nodes /etc/swift/object.ring.gz admin myContainer$userID helloWorld.txt

On one of the primary nodes, locate the data on the disk and take a look at it. (`cd /srv/node/vdb1/objects/...`)

    `cat`
    `xattr -l`

Now let's corrupt the file as if we're a bad disk, cosmic ray or other nefarious element.

    echo 'Goodbye World' > <filename>
    cat <filename>
    cd /srv/node/vdb1
    watch -n 2 ls

Once the quarantined folder appears, check the quarantined file and check that the correct file now exists:

    cat /srv/node/vbd1/quarantined/objects/...
    cat /srv/node/vbd1/objects/...


## Handoff

What happens when a node/zone goes down?

    swift-get-nodes /etc/swift/object.ring.gz admin myContainer$userID helloWorld.txt

Note a primary node and the first handoff node. Connect to the first handoff node and run

    watch -n 2 cat <path>

Then connect to one of the primary nodes
    sudo umount /srv/node/vdb1

It will take a short period of time and then it will appear on the handoff node.

Reconnect the media on the primary node:
    sudo mount /srv/node/vdb1
    watch -n 2 cat <path>

## Exercises



## Questions?
