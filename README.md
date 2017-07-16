-----------
Description
-----------

Configures MariaDB Galera cluster consisting of three CentOS 7 nodes.
The Galera Cluster performs a synchronous replication amongst identical MariaDB
servers (nodes), and according to http://galeracluster.com/documentation-webpages/gettingstarted.html
focuses on data consistency. The transactions are either applied to the databases on all
the nodes, through certification-based replication, or to none of the nodes.
The write operations are limited by the slowest node, while read operations
scale with the number of nodes. A typically recommended setup uses only one node
in the cluster for writes, and an odd total number of nodes of identical performance.
The odd number of nodes facilitates dealing with split-brain failures.

Tested on Ubuntu 14.04 Vagrant host.


----------------------
Configuration overview
----------------------

The private 192.168.10.0/24 network is associated with the Vagrant eth1 interfaces.
eth0 is used by Vagrant.  Vagrant reserves eth0 and this cannot be currently changed
(see https://github.com/mitchellh/vagrant/issues/2093).

                 ---------------------------------------------------------
                 |                     Vagrant HOST                      |
                 ---------------------------------------------------------
                  |     |                 |     |                 |     |
              eth1| eth0|             eth1| eth0|             eth1| eth0|
    192.168.10.100|     |    192.168.10.10|     |    192.168.10.1N|     |
                 -----------             ---------               ---------
                 | client0 |             | node0 |     ...       | nodeN |
                 -----------             ---------               ---------
                                          |     |                 |     |                          
                                   ------------ |          ------------ | 
                                   | /dev/sdc | |          | /dev/sdc | | 
                                   | swap     | |          | swap     | | 
                                   ------------ |          ------------ | 
                                ------------------      ------------------
                                | /dev/sdb       |      | /dev/sdb       |
                                | /var/lib/mysql |      | /var/lib/mysql |
                                ------------------      ------------------

------------
Sample Usage
------------

Install VirtualBox https://www.virtualbox.org/ and Vagrant
https://www.vagrantup.com/downloads.html

Start three servers and install MariaDB packages with:

        $ git clone https://github.com/marcindulak/vagrant-mariadb-galera-tutorial-centos7.git
        $ cd vagrant-mariadb-galera-tutorial-centos7
        $ vagrant up

The above setup follows loosely the instructions from http://galeracluster.com/documentation-webpages/gettingstarted.html and https://mariadb.com/kb/en/mariadb/getting-started-with-mariadb-galera-cluster/
The MariaDB nodes started by Vagrant have the `/var/lib/mysql` and the additional `swap` 
partitions already created, the MariaDB and Galera software installed and secured
with `mysql_secure_installation`, and the mandatory Galera options (see http://galeracluster.com/documentation-webpages/configuration.html) set in `/etc/my.cnf.d/server.cnf`
on each of the nodes. The latter consist of `wsrep_on=ON`, `wsrep_provider=/usr/lib64/galera/libgalera_smm.so`, `wsrep_cluster_address=gcomm://node0,node1,node2`, `binlog_format=row`, `default_storage_engine=InnoDB`, `innodb_autoinc_lock_mode=2`, `bind-address=192.168.10.1N`. In addition to these `wsrep_node_address=192.168.10.1N` and `wsrep_node_name=nodeN` are set appropriately per each node and the Galera State Snapshot Transfer (SST) method and credentials are set `wsrep_sst_method=rsync` `wsrep_sst_auth=sst:SST_PASSWORD`.
List the contents of `/etc/my.cnf.d/server.cnf` with:

        $ vagrant ssh node0 -c "sudo su - -c 'cat /etc/my.cnf.d/server.cnf'"

The `/etc/my.cnf.d/server.cnf` contains the `sst` user's password and therefore its read access must be limited to `mysql:mysql`.

Under certain circumstances a node in Galera Cluster may
need to cache a lot of data during a transaction, and therefore a margin of swap space is recommended,
otherwise MariaDB instance may crash. In this setup an additional swap `/dev/sdc` partition is used.

Galera Cluster does not work with MyISAM or similar nontransactional storage engines,
only InnoDB is supported. This means that MariaDB system tables are not replicated (e.g. mysql.users is not).
Running `mysql_secure_installation` or other write operations on the system tables need to
be performed on all the nodes belonging to the cluster separately and is not automatically replicated.

After having the servers running configure the Galera cluster.

- bootstrap the cluster:

        $ vagrant ssh node0 -c "sudo su - -c 'galera_new_cluster'"

  Every time this command is executed a new cluster and history UUID is created. This very first
  node used to initialize the so called primary component.

- check the setup by getting the cluster size of 1:

        $ vagrant ssh node0 -c "echo \"SHOW STATUS LIKE 'wsrep_cluster_size'\" | mysql --user=root --password=ROOT_PASSWORD" | grep wsrep_cluster_size | grep -q 1

- add the remaining nodes to the cluster created above:

        $ vagrant ssh node1 -c "sudo su - -c 'systemctl start mariadb'"
        $ vagrant ssh node2 -c "sudo su - -c 'systemctl start mariadb'"

  Note that while adding nodes it is sufficient to provide URL of one cluster member node, it does not need
  to be the node used to initialize the primary component. You could alternatively use
  `vagrant ssh node2 -c "sudo su - -c 'mysqld --user=mysql --wsrep_cluster_address=gcomm://node1'"`, however it will block.

- verify the cluster consists now of three nodes (this time the wsrep_cluster_size is checked using an alternative query):

        $ vagrant ssh node2 -c "echo \"SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME='wsrep_cluster_size'\" | mysql --user=root --password=ROOT_PASSWORD" | tail -n -1 | grep -q 3

- and other checks:

        $ vagrant ssh node2 -c "echo \"SHOW STATUS LIKE 'wsrep_cluster_status'\" | mysql --user=root --password=ROOT_PASSWORD" | grep wsrep_cluster_status | grep -q Primary
        $ vagrant ssh node2 -c "echo \"SHOW STATUS LIKE 'wsrep_connected'\" | mysql --user=root --password=ROOT_PASSWORD" | grep wsrep_connected | grep -q ON
        $ vagrant ssh node2 -c "echo \"SHOW STATUS LIKE 'wsrep_local_state'\" | mysql --user=root --password=ROOT_PASSWORD" | grep wsrep_local_state | grep -q 4

Verify the cluster operations:

- install the client machine and grant all database privileges to the `vagrant` user on all the Galera cluster nodes.
  As mentioned the system tables are not replicated and the operation needs to be performed on all nodes:

        $ vagrant ssh client0 -c "sudo su - -c 'yum -y install mariadb'"
        $ vagrant ssh node0 -c "echo \"GRANT ALL PRIVILEGES ON *.* to vagrant@'client0' IDENTIFIED BY 'vagrant';\" | mysql --user=root --password=ROOT_PASSWORD"
        $ vagrant ssh node1 -c "echo \"GRANT ALL PRIVILEGES ON *.* to vagrant@'client0' IDENTIFIED BY 'vagrant';\" | mysql --user=root --password=ROOT_PASSWORD"
        $ vagrant ssh node2 -c "echo \"GRANT ALL PRIVILEGES ON *.* to vagrant@'client0' IDENTIFIED BY 'vagrant';\" | mysql --user=root --password=ROOT_PASSWORD"

- test connection, create and populate the `cluster` table in the `vagrant` database, remotely from `client0` on `node2`:

        $ vagrant ssh client0 -c "echo \"SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME='wsrep_cluster_size'\" | mysql --host=node2 --user=vagrant --password=vagrant"
        $ vagrant ssh client0 -c "echo \"CREATE DATABASE vagrant;\" | mysql --host=node1 --user=vagrant --password=vagrant"
        $ vagrant ssh client0 -c "echo \"CREATE TABLE vagrant.cluster (id INT NOT NULL AUTO_INCREMENT, node VARCHAR(5), PRIMARY KEY(id));\" | mysql --host=node0 --user=vagrant --password=vagrant"
        $ vagrant ssh client0 -c "echo \"INSERT INTO vagrant.cluster (node) VALUES ('node2');\" | mysql --host=node2 --user=vagrant --password=vagrant"

- verify the replication on all the nodes:

        $ vagrant ssh node0 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q node2
        $ vagrant ssh node1 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q node2
        $ vagrant ssh node2 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q node2

Simulate a failure of one of the nodes. We expect the cluster in quorum and operating normally.
See http://stackoverflow.com/questions/23236871/what-is-the-defined-behavior-of-a-3-node-galera-cluster-after-one-node-dies

- simulate a fault of one node, populate the `cluster` table on one of the remaining Galera nodes:

        $ ! vagrant ssh node2 -c "sudo su - -c 'shutdown -h now'"
        $ sleep 30  # give time for virtualbox to detect VM is down
        $ vagrant ssh node0 -c "echo \"SHOW STATUS LIKE 'wsrep_cluster_size'\" | mysql --user=root --password=ROOT_PASSWORD" | grep wsrep_cluster_size | grep -q 1
        $ vagrant ssh client0 -c "echo \"INSERT INTO vagrant.cluster (node) VALUES ('fail1');\" | mysql --host=node0 --user=vagrant --password=vagrant"
        $ vagrant ssh node0 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail1
        $ vagrant ssh node1 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail1

- start the failed node (this does NOT add it to the Galera cluster):

        $ vagrant up

- assert the `node0` and `node1` are still running (see http://galeracluster.com/documentation-webpages/restartingcluster.html), and have the `-1` sequence number:

        $ vagrant ssh node0 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "\-1"
        $ vagrant ssh node1 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "\-1"
        $ vagrant ssh node2 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "6"

- add the failed node to the cluster:

        $ vagrant ssh node2 -c "sudo su - -c 'systemctl start mariadb'"

- and verify the replication:

        $ vagrant ssh node2 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail1

Simulate a failure of two the nodes. We expect (really?) the cluster (the remaining node) to operate in read mode only.
See http://serverfault.com/questions/581941/mariadb-how-to-handle-2-3-node-failure-in-multi-master-cluster

- simulate a fault of two nodes, populate the `cluster` table on the remaining Galera node:

        $ ! vagrant ssh node2 -c "sudo su - -c 'shutdown -h now'"
        $ ! vagrant ssh node1 -c "sudo su - -c 'shutdown -h now'"
        $ sleep 30  # give time for virtualbox to detect VM is down
        $ vagrant ssh node0 -c "echo \"SHOW STATUS LIKE 'wsrep_cluster_size'\" | mysql --user=root --password=ROOT_PASSWORD" | grep wsrep_cluster_size | grep -q 2
        $ vagrant ssh client0 -c "echo \"INSERT INTO vagrant.cluster (node) VALUES ('fail2');\" | mysql --host=node0 --user=vagrant --password=vagrant"
        $ vagrant ssh node0 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail2

- start the nodes (this does NOT add them to the Galera cluster):

        $ vagrant up

- assert the `node0` is still running (see http://galeracluster.com/documentation-webpages/restartingcluster.html), it has the `-1` sequence number:

        $ vagrant ssh node0 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "\-1"
        $ vagrant ssh node1 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "7"  # shouldn't this be 7 or -1, seems to be either randomly
        $ vagrant ssh node2 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "7"

- add the failed nodes to the cluster:

        $ vagrant ssh node1 -c "sudo su - -c 'systemctl start mariadb'"
        $ vagrant ssh node1 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail2
        $ vagrant ssh node2 -c "sudo su - -c 'systemctl start mariadb'"
        $ vagrant ssh node2 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail2

Simulate the whole cluster crash.

- simulate again a fault of two nodes, populate the `cluster` table on the remaining Galera node, and this time fail also the last node:

        $ ! vagrant ssh node2 -c "sudo su - -c 'shutdown -h now'"
        $ ! vagrant ssh node1 -c "sudo su - -c 'shutdown -h now'"
        $ sleep 30  # give time for virtualbox to detect VM is down
        $ vagrant ssh node0 -c "echo \"SHOW STATUS LIKE 'wsrep_cluster_size'\" | mysql --user=root --password=ROOT_PASSWORD" | grep wsrep_cluster_size | grep -q 1
        $ vagrant ssh client0 -c "echo \"INSERT INTO vagrant.cluster (node) VALUES ('fail3');\" | mysql --host=node0 --user=vagrant --password=vagrant"
        $ vagrant ssh node0 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail3
        $ ! vagrant ssh node0 -c "sudo su - -c 'shutdown -h now'"
        $ sleep 30  # give time for virtualbox to detect VM is down

- start the nodes (this does NOT start the Galera cluster):

        $ vagrant up

- assert the `node0` is the one with the most advanced state (see http://galeracluster.com/documentation-webpages/restartingcluster.html), it has the highest sequence number:

        $ vagrant ssh node0 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "9"
        $ vagrant ssh node1 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "8"
        $ vagrant ssh node2 -c "sudo su - -c 'cat /var/lib/mysql/grastate.dat'" | grep "seqno:" | grep -q "8"

- recover the InnoDB table space to a consistent state on `node0` and start the cluster:

        $ vagrant ssh node0 -c "sudo su - -c 'mysqld --user=mysql --wsrep-recover'"
        $ vagrant ssh node0 -c "sudo su - -c 'galera_new_cluster'"

- start the remaining nodes:

        $ vagrant ssh node1 -c "sudo su - -c 'systemctl start mariadb'"
        $ vagrant ssh node2 -c "sudo su - -c 'systemctl start mariadb'"

- verify the replication:

        $ vagrant ssh node0 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail3
        $ vagrant ssh node1 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail3
        $ vagrant ssh node2 -c "echo \"SELECT node FROM vagrant.cluster;\" | mysql --user=root --password=ROOT_PASSWORD" | grep -q fail3

When done, destroy the test machines with:

        $ vagrant destroy -f


------------
Dependencies
------------

None


-------
License
-------

BSD 2-clause


----
Todo
----

--------
Problems
--------

1. If you get "VBoxManage: error: Could not find a controller named 'IDE Controller'" (see https://github.com/redhat-imaging/imagefactory/issues/393)
   prepend `CONTROLLER=IDE` to all vagrant commands, e.g. `CONTROLLER=IDE vagrant up`

