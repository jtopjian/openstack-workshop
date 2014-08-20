# Prerequisites

## Information Gathering

Note the IP address on `eth0`:

    $ ip a | grep eth0

Note the hosts and IP addresses you will be working with:

    $ cat /etc/hosts


## Packages

Install `apt` tools and the Havana `apt` repo:

    $ sudo apt-get install python-software-properties
    $ sudo add-apt-repository cloud-archive:icehouse
    $ sudo apt-get update

Install MySQL:

    $ sudo apt-get install mysql-server

Install the Python MySQL library:

    $ sudo apt-get install python-mysqldb

Install RabbitMQ:

    $ sudo apt-get install rabbitmq-server

Install both `wget` and `curl`:

    $ sudo apt-get install wget curl

## Configure MySQL

Create a local `/root/.my.cnf` file to store MySQL credentials:

    [client]
    user=root
    host=localhost
    password=password

Create databases for the various OpenStack components:

    $ mysql
    mysql> create database keystone;
    mysql> create database glance;
    mysql> create database nova;
    mysql> create database cinder;
    mysql> create database neutron;

    mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON glance.*   TO 'glance'@'localhost'   IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON nova.*     TO 'nova'@'localhost'     IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON cinder.*   TO 'cinder'@'localhost'   IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON neutron.*  TO 'neutron'@'localhost'  IDENTIFIED BY 'password';

    mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'145.100.180.%' IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON glance.*   TO 'glance'@'145.100.180.%'   IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON nova.*     TO 'nova'@'145.100.180.%'     IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON cinder.*   TO 'cinder'@'145.100.180.%'   IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON neutron.*  TO 'neutron'@'145.100.180.%'  IDENTIFIED BY 'password';

Have MySQL listen on all interfaces, rather than just localhost:

  * Edit `/etc/mysql/my.cnf`, search for `bind-address` and change it to `0.0.0.0`
  * restart MySQL with `/etc/init.d/mysql restart`


