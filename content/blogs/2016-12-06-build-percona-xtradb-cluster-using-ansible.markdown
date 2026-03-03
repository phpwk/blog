---
title: Build Percona XtraDB Cluster using Ansible
date: 2016-12-06T17:04:47+07:00
excerpt: "Here we will try to install Percona XtraDB Cluster on a three nodes cluster configuration with the help from Ansible configuration management."
tags: [sysadmin, mysql, percona, ansible]
featured_image: images/pxdbc-logo.png
featured_image_class: contain bg-center
header:
  overlay_image: pxdbc-logo.png
  overlay_filter: 0.7
  caption: "Photo credit: [**Percona**](https://www.percona.com)"
---

Percona XtraDB Cluster is a database clustering solution for MySQL. It ensures
high availability, prevents downtime and data loss, and provides linear
scalability for a growing environment. Here we will try to install Percona
XtraDB Cluster on a three nodes cluster configuration with the help from
Ansible configuration management.

## Installing Percona XtraDB Cluster on Red Hat Enterprise Linux and CentOS 7

### Node configuration

| Node   | Host | IP            |
| :----- | :--- | :------------ |
| Node 1 | db1  | 192.168.50.31 |
| Node 2 | db2  | 192.168.50.32 |
| Node 3 | db3  | 192.168.50.33 |

### Prerequisites

1. Install CentOS 7
2. Update CentOS 7
3. Open firewall for TCP ports 3306, 4444, 4567, 4568:

```bash
ansible -i hosts -bkKv db-cluster -m firewalld -a 'port=3306/tcp permanent=true
state=enabled immediate=yes'
ansible -i hosts -bkKv db-cluster -m firewalld -a 'port=4444/tcp permanent=true
state=enabled immediate=yes'
ansible -i hosts -bkKv db-cluster -m firewalld -a 'port=4567/tcp permanent=true
state=enabled immediate=yes'
ansible -i hosts -bkKv db-cluster -m firewalld -a 'port=4568/tcp permanent=true
state=enabled immediate=yes'
```

1. Disable or set SELinux to Permissive, set option below in **/etc/selinux/config** and **/etc/sysconfig/selinux**:

        SELINUX=disabled

    and then reboot.

2. Remove another MySQL installation:

        ansible -bkKv -i hosts db-cluster -m yum -a 'name=mysql-community-* state=removed'

### Installation from Percona Repository

1. Install Percona Repository:

        ansible -bkKv -i hosts db-cluster -m command -a '/usr/bin/yum -y install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm creates=/etc/yum.repos.d/percona-release.repo'

2. Install Percona-XtraDB-Cluster:

        ansible -bkKv -i hosts db-cluster -m yum -a 'name=Percona-XtraDB-Cluster-57 state=present'

3. Start Percona XtraDB Cluster server service:

        ansible -bkKv -i hosts db-cluster -m systemd -a 'name=mysql state=started'

4. For each nodes, find the Percona XtraDB Cluster server temporary root password:

        sudo grep 'temporary password' /var/log/mysqld.log

5. For each nodes, use the temporary password to log in as **root**:

        mysql -u root -p

6. For each nodes, change the password for the **root** account:

        mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'tempP455wd@';
        Query OK, 0 rows affected (0.00 sec)

        mysql> exit
        Bye

7. Stop Percona XtraDB Cluster server service:

        ansible -bkKv -i hosts db-cluster -m systemd -a 'name=mysql state=stopped'

## Configuring Nodes for Write-Set Replication

1. Make sure to stop Percona XtraDB Cluster server service:

        ansible -bkKv -i hosts db-cluster -m systemd -a 'name=mysql state=stopped'

2. Add the following configuration to **/etc/my.cnf** on the first node:

        wsrep_provider=/usr/lib64/galera3/libgalera_smm.so

        wsrep_cluster_name=db-cluster
        wsrep_cluster_address=gcomm://192.168.50.31,192.168.50.32,192.168.50.33

        wsrep_node_name=db1
        wsrep_node_address=192.168.50.31

        wsrep_sst_method=xtrabackup-v2
        wsrep_sst_auth=sstuser:tempP455wd@

        pxc_strict_mode=ENFORCING

        binlog_format=ROW
        default_storage_engine=InnoDB
        innodb_autoinc_lock_mode=2

3. Use the same configuration for **/etc/my.cnf** on the other nodes with some modification for second node:

        wsrep_node_name=db2
        wsrep_node_address=192.168.50.32

4. And modification for third node:

        wsrep_node_name=db3
        wsrep_node_address=192.168.50.33

### Configuration Reference

wsrep_provider

Specify the path to the Galera library. Red Hat or CentOS: **/usr/lib64/galera3/libgalera_smm.so**

wsrep_cluster_name

Specify the logical name for your cluster. It must be the same for all nodes in your cluster.

wsrep_cluster_address

Specify the IP addresses of nodes in your cluster. At least one is required for a node to join the cluster, but it is recommended to list addresses of all nodes. This way if the first node in the list is not available, the joining node can use other addresses. No addresses are required for the initial node in the cluster. However, it is recommended to specify them and _properly bootstrap the first node_. This will ensure that the node is able to rejoin the cluster if it goes down in the future.

wsrep_node_name

Specify the logical name for each individual node. If this variable is not specified, the host name will be used.

wsrep_node_address

Specify the IP address of this particular node.

wsrep_sst_method

By default, Percona XtraDB Cluster uses Percona XtraBackup for State Snapshot Transfer (_SST_). Setting **wsrep_sst_method=xtrabackup-v2** is highly recommended. This method requires a user for SST to be set up on the initial node. Provide SST user credentials with the **wsrep_sst_auth variable**.

wsrep_sst_auth

Specify authentication credentials for SST as **<sst_user>:<sst_pass>**. You must create this user when _Bootstrapping the First Node_ and provide necessary privileges for it:

      mysql> CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'tempP455wd@';
      mysql> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO
        'sstuser'@'localhost';
      mysql> FLUSH PRIVILEGES;

pxc_strict_mode

_PXC Strict Mode_ is enabled by default and set to **ENFORCING**, which blocks the use of experimental and unsupported features in Percona XtraDB Cluster.

binlog_format

Galera supports only row-level replication, so set **binlog_format=ROW**.

default_storage_engine

Galera fully supports only the InnoDB storage engine. It will not work correctly with MyISAM or any other non-transactional storage engines. Set this variable to **default_storage_engine=InnoDB**.

innodb_autoinc_lock_mode

Galera supports only interleaved (**2**) lock mode for InnoDB. Setting the traditional (**0**) or consecutive (**1**) lock mode can cause replication to fail due to unresolved deadlocks. Set this variable to **innodb_autoinc_lock_mode=2**.

## Bootstrapping the First Node

After you _configure all PXC nodes_, initialize the cluster by bootstrapping the first node. The initial node should be the one that contains all your data, which you want to be replicated to other nodes. Bootstrapping implies starting the node without any known cluster addresses. If the **wsrep_cluster_address** variable is empty, Percona XtraDB Cluster assumes that this is the first node and initializes the cluster.

1. Start the first node using the following command instead of changing configuration:

        [user@db1 ~]$ sudo systemctl start mysql@bootstrap.service

    This command runs in bootstrap mode with **wsrep_cluster_address=gcomm://**. This tells the node to initialize the cluster with **wsrep_cluster_conf_id** set to **1**, after we _add another nodes_ to the cluster, we can then restart this node as normal, and it will use standard configuration again.

2. Check that the first node has been properly initialized:

        mysql@db1> show status like 'wsrep%';
        +----------------------------+--------------------------------------+
        | Variable_name              | Value                                |
        +----------------------------+--------------------------------------+
        | wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
        | ...                        | ...                                  |
        | wsrep_local_state          | 4                                    |
        | wsrep_local_state_comment  | Synced                               |
        | ...                        | ...                                  |
        | wsrep_cluster_size         | 1                                    |
        | wsrep_cluster_status       | Primary                              |
        | wsrep_connected            | ON                                   |
        | ...                        | ...                                  |
        | wsrep_ready                | ON                                   |
        +----------------------------+--------------------------------------+
        40 rows in set (0.01 sec)

    The previous output shows that the cluster size is 1 node, it is the primary component, the node is in **Synced** state, it is fully connected and ready for write-set replication.

3. Before _adding other nodes_ to the new cluster, we need to create user for _SST_ and provide necessary privileges for it. The credentials must match those specified when _Configuring Nodes for Write-Set Replication_.

        mysql@db1> CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'tempP455wd@';
        mysql@db1> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
        mysql@db1> FLUSH PRIVILEGES;

4. After initializing the cluster, you can _add other nodes_.

## Adding Nodes to Cluster

New nodes that are _properly configured_ are provisioned automatically. When you start a node with the address of at least one other running node in the **wsrep_cluster_address** variable, it automatically joins the cluster and synchronizes with it.

**Note:** Any existing data and configuration will be overwritten to match the data and configuration of the DONOR node. Do not join several nodes at the same time to avoid overhead due to large amounts of traffic when a new node joins.

1. By default, Percona XtraDB Cluster uses Percona XtraBackup for State Snapshot Transfer (_SST_). This requires the following:

- Set the **wsrep_sst_method** variable to **xtrabackup-v2** and provide SST user credentials with the **wsrep_sst_auth** variable.
- Create a user for SST on the initial node.

1. Start the second node using the following command:

        [user@db2 ~]$ sudo systemctl start mysql.service

    After the server starts, it should receive _SST_ automatically.

2. Check the status of second node:

        mysql@db2> show status like 'wsrep%';
        +----------------------------+--------------------------------------+
        | Variable_name              | Value                                |
        +----------------------------+--------------------------------------+
        | wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
        | ...                        | ...                                  |
        | wsrep_local_state          | 4                                    |
        | wsrep_local_state_comment  | Synced                               |
        | ...                        | ...                                  |
        | wsrep_cluster_size         | 1                                    |
        | wsrep_cluster_status       | Primary                              |
        | wsrep_connected            | ON                                   |
        | ...                        | ...                                  |
        | wsrep_ready                | ON                                   |
        +----------------------------+--------------------------------------+
        40 rows in set (0.01 sec)

    Previous output shows that the new node has been successfully added to the cluster. Cluster size is now 2 nodes, it is the primary component, and it is fully connected and ready to receive write-set replication. If the state of the second node is **Synced** as in the previous example, then the node received full SST, is synchronized with the cluster, and you can proceed to add the next node.

3. **Note:** If the state of the node is Joiner, it means that SST hasn’t finished. Do not add new nodes until all others are in Synced state.

4. Add the third node as usual:

        [user@db3 ~]$ sudo systemctl start mysql.service

5. Check the status of third node:

        mysql@db3> show status like 'wsrep%';
        +----------------------------+--------------------------------------+
        | Variable_name              | Value                                |
        +----------------------------+--------------------------------------+
        | wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
        | ...                        | ...                                  |
        | wsrep_local_state          | 4                                    |
        | wsrep_local_state_comment  | Synced                               |
        | ...                        | ...                                  |
        | wsrep_cluster_size         | 1                                    |
        | wsrep_cluster_status       | Primary                              |
        | wsrep_connected            | ON                                   |
        | ...                        | ...                                  |
        | wsrep_ready                | ON                                   |
        +----------------------------+--------------------------------------+
        40 rows in set (0.01 sec)

    Previous output shows that the new node has been successfully added to the cluster. Cluster size is now 3 nodes, it is the primary component, and it is fully connected and ready to receive write-set replication. When you add all nodes to the cluster, you can _verify replication_ by running queries and manipulating data on nodes to see if these changes are synchronized accross the cluster.

## Verifying Replication

The following procedure can be used to verify replication by doing some operation each in different node.

1. Create a new database on the second node:

        mysql@db2> CREATE DATABASE percona;
        Query OK, 1 row affected (0.01 sec)

2. Create a new table on the third node:

        mysql@db3> USE percona;
        Database changed

        mysql@db3> CREATE TABLE example (node_id INT PRIMARY KEY, node_name VARCHAR(30));
        Query OK, 0 rows affected (0.05 sec)

3. Insert a record on the first node:

        mysql@db1> INSERT INTO percona.example VALUES (1, 'percona1');
        Query OK, 1 row affected (0.02 sec)

4. Retrieve rows from that table on the second node:

        mysql@db2> SELECT * FROM percona.example;
        +---------+-----------+
        | node_id | node_name |
        +---------+-----------+
        |       1 | percona1  |
        +---------+-----------+
        1 row in set (0.00 sec)

## References

1. [Percona XtraDB Cluster 5.7 Documentation](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/index.html)
2. [Ansible Documentation](http://docs.ansible.com/ansible/)
3. [MySQL Documentation](http://dev.mysql.com/doc/)
