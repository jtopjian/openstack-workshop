# Prerequisites

## Packages

Install `apt` tools and the Havana `apt` repo:

    $ sudo apt-get install python-software-properties
    $ sudo add-apt-repository cloud-archive:havana
    $ sudo apt-get update

Install MySQL:

    $ sudo apt-get install mysql-server

Install the Python MySQL library:

    $ sudo apt-get install python-mysqldb

Install RabbitMQ:

    $ sudo apt-get install rabbitmq-server

Install both `wget` and `curl`:

    $ sudo apt-get install wget curl

## Information Gathering

Note the IP address on `eth0`:

    $ ip a | grep eth0

Note the hostname:

    $ hostname

## Configure MySQL

Create databases for the various OpenStack components:

    $ mysql -u root -p
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

