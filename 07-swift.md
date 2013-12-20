# Swift

#### The OpenStack Object Storage Service

Swift is a highly available, distributed, eventually consistent object/blob store. Swift is designed to store lots of data efficiently, safely, and cheaply.

Swift is not a filesystem or a regular volume but a store of objects. An object can be anything you want.

In many ways you can think of Swift as a bank (not in terms of security, but in terms of direct access) - you're not allowed in the bank vault and need to talk to a teller (proxy) in order to do anything with what is in the vault. You have a list (ring) of what should be in the vault and what areas of the vault it should be in (partition and zones) but you don't actually care about the details. You only care that you want your object, and the bank vault, teller and worker processes take care of all the day to day operations.

We will be installing Swift alongside our current all in one instances - while not production worthy it is a great way to try and test out Swift. In part 2 we will mimic a production style Swift cluster.

It is highly recommended to copy and paste from a rendered Markdown file (eg. from Github's page) as the tabs don't play nice with the EOF command used quite widely in this document.

## Why Swift?

Object storage is different from regular block storage and is aimed at scaling and storing a large amount of data easily. As stated earlier, Swift is a highly available, distributed, eventually consistent object/blob store. Swift is designed to store lots of data efficiently, safely, and cheaply.

  * Highly Available - you have more than one copy of each file, and if designed correctly, multiple proxies will mean it's always available.
  * Distributed - Same deal with highly available. All of your eggs aren't in one basket as objects (objects, container records, and account records) are all stored on multiple nodes.
  * Consistent - It constantly is checking files for their integrity and ensuring that a certain number of copies are always available across the cluster.
  * Efficient, Safe and Cheap storage - Can use commodity hardware.

### Use Cases

Object Storage can be used with almost any data - it's data agnostic and it's all about being able to store a file/object and not caring about the details of how it's stored, where it's stored, etc. You care about the application level details - eg. metadata, contents, etc.

Object Storage works best with "unstructured" data - data that doesn't need to managed, categorized or otherwise sorted **on the filesystem**. However it can be made to use nearly every use case you would normally encounter - folders can be mimicked for example.

Where Object Storage shines:

  - Large amounts of data (eg. map tiles, photos, > max number of files in one directory)
  - Archival
  - Consistency/Protection
  - Capacity Flexibility
  - More extensive metadata

Where Object Storage falls short is very fast disk IO - eg. video editing. However that's not to say it can't be leveraged (eg. store on Object Storage, cached on faster storage for application use)

Where Object Storage is not so hot:

  - Not a filesystem (is it a bad thing?)
  - No Hierarchy (is it a bad thing?)
  - Have to replace file when editing
  - No append support
  - No file locking
  - Consistency is eventual. Not immediate.
  - Lack of searchability built in

## How Swift Works - The Overview

Swift uses an HTTP based API so in order to make changes you can either call an HTTP command (eg. POST, PUT, GET) or use a python client (eg. `swift` on the command line) to make the calls for you. If you want to see what API calls are made behind the scenes, just add `--debug` to your command. eg. `swift --debug stat`

## Servers and Services Overview

Swift is split up into a handful of different sub-services which can be running together on one machine or separate as needed.

### Proxy Server

In our bank analogy - the proxy server is analogous to a teller.

The proxy server(s) is the server that the user will actually contact. The proxy server will go retrieve the file from wherever it is stored in the cluster and hand it off to the user. In essence the proxy server is where all the magic happens between the user and the cluster: determining where a file should be, retrieving the file and other information.

### Storage Node

In our bank analogy - these are analogous to sections of the vault.

The storage node/server is the server where objects are actually stored. Alongside are several daemons that run to do various maintenance tasks (consistency, deletion, among others).

## Swift Jargon

### Rings

Swift uses 3 rings to manage where and what things are. One ring each for accounts, containers, and objects. Each ring can be considered like a map (hash map actually) to know where objects (accounts and containers are treated like objects) are stored. The ring is stored on every node of the cluster. It is not automatically copied and must manually be installed. If your rings become out of sync? Don't do that.

You can find out more on [OpenStack's website](http://docs.openstack.org/developer/swift/overview_ring.html)

### Partitions

Partitions are the areas that the objects are separated in (by a certain number of bits of a path's md5 hash) - files won't be stored equally among partitions.

Partition optimization is difficult to do afterwards, and is best to do when you make the cluster. ([See the calculator from Rackspace](http://rackerlabs.github.io/swift-ppc/))

### Zones

Zones are arbitrary. You can define what is a zone whenever you want and they should have the same disk storage. (Best practice is give or take 10%) In our case they're just separate servers but there's no reason it can't be different racks, different data centres (although not recommended - there are better ways to get Swift to do geographic separation), different networks, etc.

## Day to Day usage

We haven't installed Swift yet on our all in one so for the how to use we need to use a different rc file.

You'll notice in your home folder is a file called `swiftrc`. Run `source swiftrc` to connect to the existing Swift installation. Of note - everyone is in the same group and can see each other's containers. If you're following this at a different time - do the Installation steps first and then come back to the Day to Day Use section.

### `swift help`

Like every other Openstack command line program, run the command followed by `help` for a list of subcommands available. Go ahead and run `swift help` and read what each subcommand does. And then we'll try a couple commands:

### Uploading an Object
`swift stat` will make sure we can connect and see our account information.

Objects must be stored in containers, and we haven't made any containers yet (you can check by `swift list`). We'll create a container called `myContainer` and upload a file to it:

    export userID=XXXX

    swift post myContainer$userID
    echo 'Hello World' > helloWorld.txt;
    swift upload myContainer$userID helloWorld.txt

We can then run a handful of commands to get some information and see what we just did:

    echo 'My Container Information:'; swift stat myContainer$userID
    echo 'File Information:'; swift stat myContainer$userID helloWorld.txt
    swift download myContainer$userID helloWorld.txt -o -

This show you specific information about the container, object, and then looking at the contents of the object we just uploaded. You'll notice that it already has some metadata pre-populated (eg. eTag or MD5 sum)

### Metadata

We can also add arbitrary metadata to an object, say for use by your application, yourself, etc.

"User" based metadata tags should have a prefix in front of them to differentiate from standard metadata entries.

  - `X-Account-Meta` for accounts
  - `X-Container-Meta` for containers
  - `X-Object-Meta` for objects

Then run:

    swift post -m "X-Object-Meta-Hello: World" myContainer$userID helloWorld.txt
    swift stat myContainer$userID helloWorld.txt

Some other metadata features available that aren't shown that you can look up:

  - [Versioning](http://docs.openstack.org/developer/swift/overview_object_versioning.html)
  - [Expiring Objects](http://docs.openstack.org/developer/swift/overview_expiring_objects.html)
  - [Temporary URLs](http://www.cybera.ca/news-and-events/tech-radar/advanced-swift-features-part-3/)
  - [Segmentation/Large Object Support](http://docs.openstack.org/developer/swift/overview_large_objects.html) (will cover later)

### Access Control Lists and Web Access

Swift provides access control lists for *containers* and are posted just the same as metadata. You can set a container to readable by the world:

    swift post -r '.r:*' myContainer$userID

The `-r` flag states you're setting the Read ACL. While the `'.r:*'` states you want anyone to read. You can change it to a list of usernames (separated by commas) instead if you only want other users to view it. And then take a look at the Read ACL attribute:

    swift stat myContainer$userID

One thing to note with ACLs - you're overwriting that metadata "tag" and not appending to anything existing.

Provided you have web access turned on in the configuration (`staticweb`) you can now permit viewing your container via a regular web browser:

    swift post -m 'web-listings: true' myContainer$userID

    keystone catalog --service object-store

The second command spits out a line - `publicURL`. Copy that, add your container name and open that in your favourite browser. If the publicURL has `localhost`, replace it with the floating IP

A couple of other tricks with the same feature are to:

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

Reload that web page and behold CSS application.

    cat <<EOF > index.html
    <html>
    <body><h1>Hello World</h1>
    </html>

    EOF

    swift upload myContainer$userID index.html
    swift post -m 'web-index:index.html' myContainer$userID

Reload that web page again and you can see it now specifies an index HTML page to use.

## Installation

For convenience, all the commands are assuming you're logged in as root. If need be run `sudo su` in order to switch from the standard `ubuntu` user to `root`. As a word of warning, the installation is a bit long and involved.

First we need to install the Swift software: (Much thanks to [OpenStack docs](http://docs.openstack.org/developer/swift/development_saio.html))

    apt-get update
    apt-get install swift swauth swift-account swift-container swift-object swift-proxy memcached python-keystoneclient python-swiftclient python-webob xfsprogs

Like our other services the packages have created a system user but we also need an OpenStack user and to tell Keynote that Swift actually exists. This should be very familiar:

    keystone user-create --name swift --tenant services --pass password --email root@localhost
    keystone user-role-add --user swift --tenant services --role admin

Obtain admin's tenant ID by running:

    keystone tenant-list

Then add the following to`/etc/keystone/default_catalog.templates`:

    catalog.RegionOne.object_store.name = Swift Service
    catalog.RegionOne.object_store.publicURL = http://localhost:8080/v1/AUTH_$(tenant_id)s
    catalog.RegionOne.object_store.adminURL = http://localhost:8080/
    catalog.RegionOne.object_store.internalURL = http://localhost:8080/v1/AUTH_$(tenant_id)s

Then be sure to restart Keystone (`service keystone restart`)

### Storage

Next we need to have somewhere to store everything. We're going to use a 3GB disk image (loopback) device. (XFS as the disk format **needs** to support xattr)

    sudo mkdir /srv
    sudo truncate -s 3GB /srv/swift-disk
    sudo mkfs.xfs /srv/swift-disk

    echo "/srv/swift-disk /mnt/sdb1 xfs loop,noatime,nodiratime,nobarrier,logbufs=8 0 0" | sudo tee -a /etc/fstab

    mkdir /mnt/sdb1
    mount /mnt/sdb1
    mkdir /mnt/sdb1/1 /mnt/sdb1/2 /mnt/sdb1/3 /mnt/sdb1/4
    chown swift:swift /mnt/sdb1/*
    for x in {1..4}; do sudo ln -s /mnt/sdb1/$x /srv/$x; done
    mkdir -p /srv/1/node/sdb1 /srv/2/node/sdb2 /srv/3/node/sdb3 /srv/4/node/sdb4 /var/run/swift
    chown -R swift:swift /var/run/swift
    for x in {1..4}; do sudo chown -R swift:swift /srv/$x/; done

### rsync

Swift utilizes `rsync` extensively to send copies around to each node, so we need to set up `rsyncd` to be sitting and listening for swift uploads.

    sed -i 's/RSYNC_ENABLE=false/RSYNC_ENABLE=true/g' /etc/default/rsync

    cat << EOF > /etc/rsyncd.conf
    uid = swift
    gid = swift
    log file = /var/log/rsyncd.log
    pid file = /var/run/rsyncd.pid
    address = 0.0.0.0

    [account6012]
    max connections = 25
    path = /srv/1/node/
    read only = false
    lock file = /var/lock/account6012.lock

    [account6022]
    max connections = 25
    path = /srv/2/node/
    read only = false
    lock file = /var/lock/account6022.lock

    [account6032]
    max connections = 25
    path = /srv/3/node/
    read only = false
    lock file = /var/lock/account6032.lock

    [account6042]
    max connections = 25
    path = /srv/4/node/
    read only = false
    lock file = /var/lock/account6042.lock

    [container6011]
    max connections = 25
    path = /srv/1/node/
    read only = false
    lock file = /var/lock/container6011.lock

    [container6021]
    max connections = 25
    path = /srv/2/node/
    read only = false
    lock file = /var/lock/container6021.lock

    [container6031]
    max connections = 25
    path = /srv/3/node/
    read only = false
    lock file = /var/lock/container6031.lock

    [container6041]
    max connections = 25
    path = /srv/4/node/
    read only = false
    lock file = /var/lock/container6041.lock

    [object6010]
    max connections = 25
    path = /srv/1/node/
    read only = false
    lock file = /var/lock/object6010.lock

    [object6020]
    max connections = 25
    path = /srv/2/node/
    read only = false
    lock file = /var/lock/object6020.lock

    [object6030]
    max connections = 25
    path = /srv/3/node/
    read only = false
    lock file = /var/lock/object6030.lock

    [object6040]
    max connections = 25
    path = /srv/4/node/
    read only = false
    lock file = /var/lock/object6040.lock
    EOF

As always, when you make changes to configuration files restart the service.

    sudo service rsync restart

You can check that `rsync` is working by running `rsync rsync://pub@localhost/`.

### Swift Proxy

We then have to configure the proxy node. The proxy service is what everything talks to get/put anything into the object-store. First we have to fix memcache by editing `/etc/memcache.conf` changing: (use your local IP eg. 10.0.0.148)

    -l 127.0.0.1
    to
    -l LOCAL_IP

Then restart memcache: `sudo service memcached restart`

Next we need to set up the proxy server's configuration file. Copy and paste the following code block to set up a basic proxy-server config:

    cat <<EOF | sudo tee /etc/swift/proxy-server.conf
    [DEFAULT]
    bind_port = 8080
    user = swift

    [pipeline:main]
    pipeline = healthcheck cache authtoken keystoneauth staticweb proxy-server

    [app:proxy-server]
    use = egg:swift#proxy
    allow_account_management = true
    account_autocreate = true

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
    auth_host = 127.0.0.1
    auth_port = 35357

    # the same admin_token as provided in keystone.conf
    admin_token = 012345SECRET99TOKEN012345

    # the service tenant and swift userid and password created in Keystone
    admin_tenant_name = services
    admin_user = swift
    admin_password = password

    [filter:cache]
    use = egg:swift#memcache

    [filter:catch_errors]
    use = egg:swift#catch_errors

    [filter:healthcheck]
    use = egg:swift#healthcheck

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

Be sure to edit `proxy-server.conf` to change the admin token, and if necessary the keystone settings (`auth_host`).

Additionally we have to need to add a salt for Swift to use when making it's hashes of objects:

    ( cat | sudo tee /etc/swift/swift.conf ) <<EOF
    [swift-hash]
    swift_hash_path_suffix = YOUR_RANDOM_SALT
    EOF

Lastly we need to add configuration files for the Object Expiration daemon and some information for our nodes.

    mkdir -p /etc/swift/object-server
    mkdir -p /etc/swift/account-server
    mkdir -p /etc/swift/container-server

    ( cat | sudo tee /etc/swift/object-expirer.conf ) <<EOF
    [DEFAULT]
    # swift_dir = /etc/swift
    user = swift
    # You can specify default log routing here if you want:
    log_name = object-expirer
    log_facility = LOG_LOCAL6
    log_level = INFO
    #log_address = /dev/log
    #
    # comma separated list of functions to call to setup custom log handlers.
    # functions get passed: conf, name, log_to_console, log_route, fmt, logger,
    # adapted_logger
    # log_custom_handlers =
    #
    # If set, log_udp_host will override log_address
    # log_udp_host =
    # log_udp_port = 514
    #
    # You can enable StatsD logging here:
    # log_statsd_host = localhost
    # log_statsd_port = 8125
    # log_statsd_default_sample_rate = 1.0
    # log_statsd_sample_rate_factor = 1.0
    # log_statsd_metric_prefix =

    [object-expirer]
    interval = 300
    # auto_create_account_prefix = .
    # report_interval = 300
    # concurrency is the level of concurrency o use to do the work, this value
    # must be set to at least 1
    # concurrency = 1
    # processes is how many parts to divide the work into, one part per process
    #   that will be doing the work
    # processes set 0 means that a single process will be doing all the work
    # processes can also be specified on the command line and will override the
    #   config value
    # processes = 0
    # process is which of the parts a particular process will work on
    # process can also be specified on the command line and will overide the config
    #   value
    # process is "zero based", if you want to use 3 processes, you should run
    #  processes with process set to 0, 1, and 2
    # process = 0

    [pipeline:main]
    pipeline = catch_errors cache proxy-server

    [app:proxy-server]
    use = egg:swift#proxy
    # See proxy-server.conf-sample for options

    [filter:cache]
    use = egg:swift#memcache
    # See proxy-server.conf-sample for options

    [filter:catch_errors]
    use = egg:swift#catch_errors
    # See proxy-server.conf-sample for options
    EOF

And more for each of our storage nodes (because this is an all in one)

    ( cat | sudo tee /etc/swift/account-server/1.conf ) <<EOF
    [DEFAULT]
    devices = /srv/1/node
    mount_check = false
    disable_fallocate = true
    bind_port = 6012
    workers = 1
    user = swift
    log_facility = LOG_LOCAL2
    recon_cache_path = /var/cache/swift
    eventlet_debug = true

    [pipeline:main]
    pipeline = recon account-server

    [app:account-server]
    use = egg:swift#account

    [filter:recon]
    use = egg:swift#recon

    [account-replicator]
    vm_test_mode = yes

    [account-auditor]

    [account-reaper]
    EOF

    ( cat | sudo tee /etc/swift/container-server/1.conf ) <<EOF
    [DEFAULT]
    devices = /srv/1/node
    mount_check = false
    disable_fallocate = true
    bind_port = 6011
    workers = 1
    user = swift
    log_facility = LOG_LOCAL2
    recon_cache_path = /var/cache/swift
    eventlet_debug = true

    [pipeline:main]
    pipeline = recon container-server

    [app:container-server]
    use = egg:swift#container

    [filter:recon]
    use = egg:swift#recon

    [container-replicator]
    vm_test_mode = yes

    [container-updater]

    [container-auditor]

    [container-sync]
    EOF

    ( cat | sudo tee /etc/swift/object-server/1.conf ) <<EOF
    [DEFAULT]
    devices = /srv/1/node
    mount_check = false
    disable_fallocate = true
    bind_port = 6010
    workers = 1
    user = swift
    log_facility = LOG_LOCAL2
    recon_cache_path = /var/cache/swift
    eventlet_debug = true

    [pipeline:main]
    pipeline = recon object-server

    [app:object-server]
    use = egg:swift#object

    [filter:recon]
    use = egg:swift#recon

    [object-replicator]
    vm_test_mode = yes

    [object-updater]

    [object-auditor]
    EOF

And then some regex fun to replicate the above for storage nodes 2, 3 and 4.

    cd /etc/swift/account-server
    for i in {2,3,4}
    do
    j=$(($i+1))
    port=$((6000 + ($i*10) + 2))
    cp 1.conf $i.conf
    sed -i "s/1\/node/$i\/node/g" $i.conf
    sed -i "s/6012/$port/g" $i.conf
    sed -i "s/LOCAL2/LOCAL$j/g" $i.conf
    sed -i "s/cache\/swift/cache\/swift$i/g" $i.conf
    mkdir -p "/var/cache/swift$i"
    done

    cd /etc/swift/container-server
    for i in {2,3,4}
    do
    j=$(($i+1))
    port=$((6000 + ($i*10) + 1))
    cp 1.conf $i.conf
    sed -i "s/1\/node/$i\/node/g" $i.conf
    sed -i "s/6011/$port/g" $i.conf
    sed -i "s/LOCAL2/LOCAL$j/g" $i.conf
    sed -i "s/cache\/swift/cache\/swift$i/g" $i.conf
    done

    cd /etc/swift/object-server
    for i in {2,3,4}
    do
    j=$(($i+1))
    port=$((6000 + ($i*10) + 0))
    cp 1.conf $i.conf
    sed -i "s/1\/node/$i\/node/g" $i.conf
    sed -i "s/6010/$port/g" $i.conf
    sed -i "s/LOCAL2/LOCAL$j/g" $i.conf
    sed -i "s/cache\/swift/cache\/swift$i/g" $i.conf
    done

    rm /etc/swift/account-server.conf
    rm /etc/swift/container-server.conf
    rm /etc/swift/object-server.conf

Lastly some house keeping and missing folders:

    cd /etc/swift
    mkdir -p /var/swift/keystone-signing
    chown -R swift:swift /var/swift/keystone-signing

Next we'll be creating our rings. Our partition power is set to 6 because we're making a *tiny* Swift installation. (The arguments are partition power, replication number, minimum rebuild hours). Replication does not have to be a full number - it can be a fraction (eg. 3.25 - it means some nodes will have a fourth copy)

    swift-ring-builder account.builder create 6 3 1
    swift-ring-builder container.builder create 6 3 1
    swift-ring-builder object.builder create 6 3 1

Be sure to change the IP below to your IP!!! (run `ip a` and locate your 10.0.0.x address)

    export MY_LOCAL_IP=10.0.0.0

    swift-ring-builder account.builder add z1-$MY_LOCAL_IP:6012/sdb1 100
    swift-ring-builder container.builder add z1-$MY_LOCAL_IP:6011/sdb1 100
    swift-ring-builder object.builder add z1-$MY_LOCAL_IP:6010/sdb1 100
    swift-ring-builder account.builder add z1-$MY_LOCAL_IP:6022/sdb2 100
    swift-ring-builder container.builder add z1-$MY_LOCAL_IP:6021/sdb2 100
    swift-ring-builder object.builder add z1-$MY_LOCAL_IP:6020/sdb2 100
    swift-ring-builder account.builder add z1-$MY_LOCAL_IP:6032/sdb3 100
    swift-ring-builder container.builder add z1-$MY_LOCAL_IP:6031/sdb3 100
    swift-ring-builder object.builder add z1-$MY_LOCAL_IP:6030/sdb3 100
    swift-ring-builder account.builder add z1-$MY_LOCAL_IP:6042/sdb4 100
    swift-ring-builder container.builder add z1-$MY_LOCAL_IP:6041/sdb4 100
    swift-ring-builder object.builder add z1-$MY_LOCAL_IP:6040/sdb4 100

Verify and rebalance:

    swift-ring-builder account.builder
    swift-ring-builder container.builder
    swift-ring-builder object.builder

    swift-ring-builder account.builder rebalance
    swift-ring-builder container.builder rebalance
    swift-ring-builder object.builder rebalance

    swift-ring-builder account.builder
    swift-ring-builder container.builder
    swift-ring-builder object.builder

    chown -R swift:swift /etc/swift
    chown -R swift:swift /var/cache/swift*
    service swift-proxy restart
    swift-init all restart

Now let's do a quick verification that it's working.

    swift stat

You should get feedback similar to:

       Account: AUTH_d196fd3acba649739e5dfbe5ebce108f
    Containers: 0
       Objects: 0
         Bytes: 0
    Content-Type: text/plain; charset=utf-8
    X-Timestamp: 1387325156.64803
    X-Put-Timestamp: 1387325156.64803

Lastly we can run `swift-recon --all` to get some interesting stats on our empty cluster.

Our all in one installation is finished. Try the Day to Day instructions
