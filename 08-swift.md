# Swift

#### The OpenStack Object Storage Service

Swift is a highly available, distributed, eventually consistent object/blob store. Swift is designed to store lots of data efficiently, safely, and cheaply.

Swift is not a filesystem or a regular volume but a store of objects. An object can be anything you want.

In many ways you can think of Swift as a bank (not in terms of security, but in terms of direct access) - you're not allowed in the bank vault and need to talk to a teller (proxy) in order to do anything with what is in the vault. You have a list (ring) of what should be in the vault and what areas of the vault it should be in (partition and zones) but you don't actually care about the details. You only care that you want your object, and the bank vault, teller and worker processes take care of all the day to day operations.

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

    $ sudo apt-get update
    $ sudo apt-get install swift swauth swift-account swift-container swift-object swift-proxy memcached python-keystoneclient python-swiftclient python-webob xfsprogs

Like our other services the packages have created a system user but we also need an OpenStack user and to tell Keynote that Swift actually exists. This should be very familiar:

    $ keystone user-create --name swift --tenant services --pass password --email root@localhost
    $ keystone user-role-add --user swift --tenant services --role admin

Obtain admin's tenant ID by running:

    $ keystone tenant-list

Then add the following to`/etc/keystone/default_catalog.templates`:

    catalog.RegionOne.object_store.name = Swift Service
    catalog.RegionOne.object_store.publicURL = http://cc-ip:8080/v1/AUTH_$(tenant_id)s
    catalog.RegionOne.object_store.adminURL = http://cc-ip:8080/
    catalog.RegionOne.object_store.internalURL = http://cc-ip:8080/v1/AUTH_$(tenant_id)s

Then be sure to restart Keystone (`service keystone restart`)

### Storage

Next we need to have somewhere to store everything. We're going to use a 3GB disk image (loopback) device. (XFS as the disk format **needs** to support xattr)

    $ sudo lvcreate -n swift -L 5G cinder-volumes
    $ sudo mkfs.xfs -f /dev/mapper/cinder--volumes-swift
    $ sudo mkdir -p /srv/node/swift
    $ sudo echo "/dev/mapper/cinder--volumes-swift /srv/node/swift xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab
    $ sudo mount /srv/node/swift
    $ sudo chown -R swift:swift /srv/node

### rsync

Swift utilizes `rsync` extensively to send copies around to each node, so we need to set up `rsyncd` to be sitting and listening for swift uploads.

    $ sudo sed -i 's/RSYNC_ENABLE=false/RSYNC_ENABLE=true/g' /etc/default/rsync

    $ cat <<EOF | sudo tee /etc/rsyncd.conf
    uid = swift
    gid = swift
    log file = /var/log/rsyncd.log
    pid file = /var/run/rsyncd.pid
    address = ${ip}

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

As always, when you make changes to configuration files restart the service.

    $ sudo service rsync restart

You can check that `rsync` is working by running `rsync rsync://pub@localhost/`.

### Swift Proxy

We then have to configure the proxy node. The proxy service is what everything talks to get/put anything into the object-store.

First, set up the proxy server's configuration file. Copy and paste the following code block to set up a basic proxy-server config:

    $ cat <<EOF | sudo tee /etc/swift/proxy-server.conf
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

    $ cat <<EOF | sudo tee /etc/swift/swift.conf
    [swift-hash]
    swift_hash_path_suffix = YOUR_RANDOM_SALT
    EOF

Lastly some house keeping and missing folders:

    $ sudo mkdir -p /var/swift/keystone-signing
    $ sudo chown -R swift:swift /var/swift/keystone-signing

Next we'll be creating our rings. Our partition power is set to 6 because we're making a *tiny* Swift installation. (The arguments are partition power, replication number, minimum rebuild hours). Replication does not have to be a full number - it can be a fraction (eg. 3.25 - it means some nodes will have a fourth copy)

    $ cd /etc/swift
    $ sudo swift-ring-builder account.builder create 6 1 1
    $ sudo swift-ring-builder container.builder create 6 1 1
    $ sudo swift-ring-builder object.builder create 6 1 1

Be sure to change the IP below to your IP!!! (run `ip a` and locate your 10.0.0.x address)

    $ export CC_IP=<cc ip>
    $ sudo swift-ring-builder account.builder add z1-$CC_IP:6002/swift 100
    $ sudo swift-ring-builder container.builder add z1-$CC_IP:6001/swift 100
    $ sudo swift-ring-builder object.builder add z1-$CC_IP:6000/swift 100

Verify and rebalance:

    $ sudo swift-ring-builder account.builder
    $ sudo swift-ring-builder container.builder
    $ sudo swift-ring-builder object.builder

    $ sudo swift-ring-builder account.builder rebalance
    $ sudo swift-ring-builder container.builder rebalance
    $ sudo swift-ring-builder object.builder rebalance

    $ sudo chown -R swift:swift /etc/swift
    $ sudo chown -R swift:swift /var/cache/swift*
    $ sudo service swift-proxy restart
    $ sudo swift-init all restart

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
