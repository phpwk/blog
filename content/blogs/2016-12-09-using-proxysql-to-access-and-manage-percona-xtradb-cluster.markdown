---
title: Using ProxySQL to Access and Manage Percona XtraDB Cluster
date: 2016-12-09T10:00:54+07:00
excerpt: "We will use ProxySQL as a gateway to access our Percona XtraDB Cluster. ProxySQL is a high performance SQL proxy."
tags: [sysadmin, database, mysql, percona, proxysql]
featured_image: images/pxdbc-logo.png
featured_image_class: contain bg-center
header:
  overlay_image: pxdbc-logo.png
  overlay_filter: 0.7
  caption: "Photo credit: [**Percona**](https://www.percona.com)"
---

We will use ProxySQL as a gateway to access our Percona XtraDB Cluster. ProxySQL is a high-performance SQL proxy. ProxySQL runs as a daemon watched by a monitoring process. The process monitors the daemon and restarts it in case of a crash to minimize downtime. The daemon accepts incoming traffic from MySQL clients and forwards it to backend MySQL servers.

## Installing ProxySQL

### Node configuration

| Node   | Host     | IP            |
| :----- | :------- | :------------ |
| Node 1 | db1      | 192.168.50.31 |
| Node 2 | db2      | 192.168.50.32 |
| Node 3 | db3      | 192.168.50.33 |
| Node 4 | proxysql | 192.168.50.34 |

### Prerequisites

1. Install CentOS 7
2. Update CentOS 7
3. Open firewall for TCP ports 6033:

        ansible -i hosts -bkKv proxysql -m firewalld -a 'port=6033/tcp permanent=true state=enabled immediate=yes'

4. Disable or set SELinux to Permissive, set option below in **/etc/selinux/config** and **/etc/sysconfig/selinux**:

        SELINUX=disabled

and then reboot. 5. Remove another MySQL installation:

        ansible -bkKv -i hosts proxysql -m yum -a 'name=mysql-community-* state=removed'

### Installation from Percona Repository

1. Install Percona Repository:

        ansible -bkKv -i hosts proxysql -m command -a '/usr/bin/yum -y install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm creates=/etc/yum.repos.d/percona-release.repo'

2. Install ProxySQL:

        ansible -bkKv -i hosts proxysql -m yum -a 'name=proxysql state=present'

3. Change default credential configuration to **/etc/proxysql.cnf**:

        admin_credentials="admin:tempP455wd@"

4. Also change the default credential and host configuration in **/etc/proxysql-admin.cnf**:

        export PROXYSQL_USERNAME="admin"
        export PROXYSQL_PASSWORD="tempP455wd@"
        export PROXYSQL_HOSTNAME="127.0.0.1"
        export PROXYSQL_PORT="6032"

5. Disable _requiretty_ option in **/etc/sudoers**:

        Defaults    !requiretty

6. Start ProxySQL service:

        ansible -bkKv -i hosts proxysql -m systemd -a 'name=proxysql state=started'

## Manual Configuration

1. Install MySQL client (if doesn't exist):

        ansible -bkKv -i hosts proxysql -m yum -a 'name=Percona-XtraDB-Cluster-client-57 state=present'

2. Login to ProxySQL Admin Module:

        [user@proxysql ~]$ mysql -u admin -ptempP455wd@ -h 127.0.0.1 -P 6032

        Welcome to the MySQL monitor.  Commands end with ; or \g.
        Your MySQL connection id is 2
        Server version: 5.5.30 (ProxySQL Admin Module)

        Copyright (c) 2009-2016 Percona LLC and/or its affiliates
        Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

        Oracle is a registered trademark of Oracle Corporation and/or its
        affiliates. Other names may be trademarks of their respective
        owners.

        Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

        mysql@proxysql>

3. Check ProxySQL databases and tables:

        mysql@proxysql> SHOW DATABASES;
        +-----+---------+-------------------------------+
        | seq | name    | file                          |
        +-----+---------+-------------------------------+
        | 0   | main    |                               |
        | 2   | disk    | /var/lib/proxysql/proxysql.db |
        | 3   | stats   |                               |
        | 4   | monitor |                               |
        +-----+---------+-------------------------------+
        4 rows in set (0.00 sec)

        mysql@proxysql> SHOW TABLES;
        +--------------------------------------+
        | tables                               |
        +--------------------------------------+
        | global_variables                     |
        | mysql_collations                     |
        | mysql_query_rules                    |
        | mysql_replication_hostgroups         |
        | mysql_servers                        |
        | mysql_users                          |
        | runtime_global_variables             |
        | runtime_mysql_query_rules            |
        | runtime_mysql_replication_hostgroups |
        | runtime_mysql_servers                |
        | runtime_scheduler                    |
        | scheduler                            |
        +--------------------------------------+
        12 rows in set (0.00 sec)

4. **Note**: ProxySQL configuration can run from three different areas:
    - MEMORY (your current working place)
    - RUNTIME (the production settings)
    - DISK (durable configuration, saved inside an SQLITE database)

When you change a parameter, you change it in MEMORY area. That is done by design to allow you to test the changes before pushing to production (RUNTIME), or save them to disk.

### Adding cluster nodes to ProxySQL

1. To configure the backend Percona XtraDB Cluster nodes in ProxySQL, insert corresponding records into the **mysql_servers** table.
2. **Note**: ProxySQL uses the concept of _hostgroups_ to group cluster nodes. This enables you to balance the load in a cluster by routing different types of traffic to different groups. There are many ways you can configure hostgroups (for example master and slaves, read and write load, etc.) and a every node can be a member of multiple hostgroups.
3. This example adds three Percona XtraDB Cluster nodes to the default hostgroup (0), which receives both write and read traffic:

        mysql@proxysql> INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (0,'192.168.50.31',3306);
        mysql@proxysql> INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (0,'192.168.50.32',3306);
        mysql@proxysql> INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (0,'192.168.50.33',3306);

4. Use the following command to see the nodes:

        mysql@proxysql> SELECT * FROM mysql_servers;

        +--------------+---------------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+
        | hostgroup_id | hostname      | port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms |
        +--------------+---------------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+
        | 0            | 192.168.50.31 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |
        | 0            | 192.168.50.32 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |
        | 0            | 192.168.50.33 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |
        +--------------+---------------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+
        3 rows in set (0.00 sec)

### Creating ProxySQL Monitoring User

1. To enable monitoring of Percona XtraDB Cluster nodes in ProxySQL, create a user with **USAGE** privilege on any node in the cluster and configure the user in ProxySQL.

        mysql@db1> CREATE USER 'proxysql'@'%' IDENTIFIED BY 'tempP455wd@';
        mysql@db1> GRANT USAGE ON *.* TO 'proxysql'@'%';

2. Configure the monitoring user in ProxySQL:

        mysql@proxysql> UPDATE global_variables SET variable_value='proxysql' WHERE variable_name='mysql-monitor_username';
        mysql@proxysql> UPDATE global_variables SET variable_value='tempP455wd@' WHERE variable_name='mysql-monitor_password';

3. Use **LOAD** command to load this configuration at runtime:

        mysql@proxysql> LOAD MYSQL VARIABLES TO RUNTIME;

4. To save these changes to disk so they can persist after ProxySQL is shutdown, use **SAVE** command:

        mysql@proxysql> SAVE MYSQL VARIABLES TO DISK;

5. Check monitoring logs to ensure that monitoring is enabled:

        mysql@proxysql> SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 6;
        +---------------+------+------------------+----------------------+---------------+
        | hostname      | port | time_start_us    | connect_success_time | connect_error |
        +---------------+------+------------------+----------------------+---------------+
        | 192.168.50.31 | 3306 | 1469635762434625 | 1695                 | NULL          |
        | 192.168.50.32 | 3306 | 1469635762434625 | 1779                 | NULL          |
        | 192.168.50.33 | 3306 | 1469635762434625 | 1627                 | NULL          |
        | 192.168.50.31 | 3306 | 1469635642434517 | 1557                 | NULL          |
        | 192.168.50.32 | 3306 | 1469635642434517 | 2737                 | NULL          |
        | 192.168.50.33 | 3306 | 1469635642434517 | 1447                 | NULL          |
        +---------------+------+------------------+----------------------+---------------+
        6 rows in set (0.00 sec)

6. Check that ProxySQL is able to connect and ping the nodes:

        mysql> SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 6;
        +---------------+------+------------------+-------------------+------------+
        | hostname      | port | time_start_us    | ping_success_time | ping_error |
        +---------------+------+------------------+-------------------+------------+
        | 192.168.50.31 | 3306 | 1469635762416190 | 948               | NULL       |
        | 192.168.50.32 | 3306 | 1469635762416190 | 803               | NULL       |
        | 192.168.50.33 | 3306 | 1469635762416190 | 711               | NULL       |
        | 192.168.50.31 | 3306 | 1469635702416062 | 783               | NULL       |
        | 192.168.50.32 | 3306 | 1469635702416062 | 631               | NULL       |
        | 192.168.50.33 | 3306 | 1469635702416062 | 542               | NULL       |
        +---------------+------+------------------+-------------------+------------+
        6 rows in set (0.00 sec)

7. Enable monitoring of these nodes by loading them at runtime:

        mysql@proxysql> LOAD MYSQL SERVERS TO RUNTIME;

### Creating ProxySQL Client User

ProxySQL must have users that can access backend nodes to manage connections.

1. To add a user, insert credentials into mysql_users table:

        mysql@proxysql> INSERT INTO mysql_users (username,password) VALUES ('sbuser','sbP455wd@');
        Query OK, 1 row affected (0.00 sec)

2. Load the user into runtime space:

        mysql@proxysql> LOAD MYSQL USERS TO RUNTIME;

3. While in the runtime, the user password is not hashed. To facilitate the support of hashed passwords, ProxySQL v1.2.3 introduced a new global boolean variable, **admin-hash_password**, enabled by default. When **admin-hash_password=true**, password are automatically hashed at _RUNTIME_ only when running _LOAD MYSQL USERS TO RUNTIME_. Passwords in mysql_users tables are yet not automatically hashed.[2]

        mysql@proxysql> SELECT username,password FROM mysql_users;
        +----------+-----------+
        | username | password  |
        +----------+-----------+
        | sbuser   | sbP455wd@ |
        +----------+-----------+
        1 row in set (0.00 sec)

4. Nonetheless, it is easily possible to hash the passwords in mysql*users table, both in-memory and on-disk. It is enough to copy users from \_RUNTIME*, for example running _SAVE MYSQL USERS FROM RUNTIME_ after _LOAD MYSQL USERS TO RUNTIME_, and then _SAVE MYSQL USERS TO DISK_ (recommended).[2]

        mysql@proxysql> SAVE MYSQL USERS FROM RUNTIME;
        Query OK, 0 rows affected (0.00 sec)

        mysql@proxysql> SELECT username,password FROM mysql_users;
        +----------+-------------------------------------------+
        | username | password                                  |
        +----------+-------------------------------------------+
        | sbuser   | *00CEF9910CD14484D09244483B96CF51F6294D40 |
        +----------+-------------------------------------------+
        1 row in set (0.00 sec)

        mysql@proxysql> SAVE MYSQL USERS TO DISK;
        Query OK, 0 rows affected (0.29 sec)

5. Try to login with the new user to check if the user has been set up correctly:

        [user@proxysql ~]$ mysql -u sbuser -psbP455wd@ -h 127.0.0.1 -P 6033

        Welcome to the MySQL monitor.  Commands end with ; or \g.
        Your MySQL connection id is 1491
        Server version: 5.5.30 (ProxySQL)

        Copyright (c) 2009-2016 Percona LLC and/or its affiliates
        Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

        Oracle is a registered trademark of Oracle Corporation and/or its
        affiliates. Other names may be trademarks of their respective
        owners.

        Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

        mysql@proxysql>

6. To provide read/write access to the cluster for ProxySQL, add this user on one of the Percona XtraDB Cluster nodes:

        mysql@db2> CREATE USER 'sbuser'@'192.168.50.34' IDENTIFIED BY 'sbP455wd@';
        Query OK, 0 rows affected (0.01 sec)

        mysql@db2> GRANT ALL ON sbtest.* TO 'sbuser'@'192.168.50.34';
        Query OK, 0 rows affected (0.00 sec)

### Adding Galera Support

Default ProxySQL cannot detect a node which is not in **Synced** state. To monitor status of Percona XtraDB Cluster nodes, use the **proxysql_galera_checker** script. The script is located here: **/usr/bin/proxysql_galera_checker**. To use this script, load it into ProxySQL Scheduler.

1. The following example shows how you can load the script for default ProxySQL configuration:

        mysql@proxysql> INSERT INTO scheduler(id,active,interval_ms,filename,arg1,arg2,arg3,arg4,arg5)
            VALUES
            (1,1,10000,'/usr/bin/proxysql_galera_checker',0,0,3,1,'/var/lib/proxysql/proxysql_galera_checker.log');

2. To load the scheduler changes into the runtime space:

        mysql@proxysql> LOAD SCHEDULER TO RUNTIME;

3. To make sure that the script has been loaded, check the runtime_scheduler table:

        mysql@proxysql> SELECT * FROM runtime_scheduler\G
        *************************** 1. row ***************************
                 id: 1
             active: 1
        interval_ms: 10000
           filename: /usr/bin/proxysql_galera_checker
               arg1: 0
               arg2: 0
               arg3: 3
               arg4: 1
               arg5: /var/lib/proxysql/proxysql_galera_checker.log
            comment:
        1 row in set (0.00 sec)

4. To check the status of available nodes, run the following command:

        mysql@proxysql> SELECT hostgroup_id,hostname,port,status FROM mysql_servers;
        +--------------+---------------+------+--------+
        | hostgroup_id | hostname      | port | status |
        +--------------+---------------+------+--------+
        | 0            | 192.168.50.31 | 3306 | ONLINE |
        | 0            | 192.168.50.32 | 3306 | ONLINE |
        | 0            | 192.168.50.33 | 3306 | ONLINE |
        +--------------+---------------+------+--------+
        3 rows in set (0.00 sec)

5. **Note**: Each node can have the following status:
    - **ONLINE**: backend node is fully operational.
    - **SHUNNED**: backend node is temporarily taken out of use, because either too many connection errors hapenned in a short time, or replication lag exceeded the allowed threshold.
    - **OFFLINE_SOFT**: new incoming connections aren’t accepted, while existing connections are kept until they become inactive. In other words, connections are kept in use until the current transaction is completed. This allows to gracefully detach a backend node.
    - **OFFLINE_HARD**: existing connections are dropped, and new incoming connections aren’t accepted. This is equivalent to deleting the node from a hostgroup, or temporarily taking it out of the hostgroup for maintenance.

### Testing Cluster with sysbench

1. Install **sysbench** from Percona software repositories:

          ansible -bkKv -i hosts proxysql -m yum -a 'name=sysbench state=present'

    **Note**: **sysbench** requires ProxySQL client user credentials that you created in _Creating ProxySQL Client User_.

2. Create the database that will be used for testing on one of the Percona XtraDB Cluster nodes:

        mysql@db3> CREATE DATABASE sbtest;

3. Populate the table with data for the benchmark on the ProxySQL node:

        [user@proxysql ~]$ sysbench --report-interval=5 --num-threads=4 \
          --num-requests=0 --max-time=20 \
          --test=/usr/share/doc/sysbench/tests/db/oltp.lua \
          --mysql-user='sbuser' --mysql-password='sbP455wd@' \
          --oltp-table-size=10000 --mysql-host=127.0.0.1 --mysql-port=6033 \
          prepare

4. Run the benchmark on the ProxySQL node:

        root@proxysql:~# sysbench --report-interval=5 --num-threads=4 \
          --num-requests=0 --max-time=20 \
          --test=/usr/share/doc/sysbench/tests/db/oltp.lua \
          --mysql-user='sbuser' --mysql-password='sbP455wd@' \
          --oltp-table-size=10000 --mysql-host=127.0.0.1 --mysql-port=6033 \
          run

5. ProxySQL stores collected data in the **stats** schema:

        mysql@proxysql> SHOW TABLES FROM stats;
        +--------------------------------+
        | tables                         |
        +--------------------------------+
        | stats_mysql_query_rules        |
        | stats_mysql_commands_counters  |
        | stats_mysql_processlist        |
        | stats_mysql_connection_pool    |
        | stats_mysql_query_digest       |
        | stats_mysql_query_digest_reset |
        | stats_mysql_global             |
        +--------------------------------+

6. For example, to see the number of commands that run on the cluster:

        mysql@proxysql> SELECT * FROM stats_mysql_commands_counters;
        +-------------------+---------------+-----------+-----------+-----------+---------+---------+----------+----------+-----------+-----------+--------+--------+---------+----------+
        | Command           | Total_Time_us | Total_cnt | cnt_100us | cnt_500us | cnt_1ms | cnt_5ms | cnt_10ms | cnt_50ms | cnt_100ms | cnt_500ms | cnt_1s | cnt_5s | cnt_10s | cnt_INFs |
        +-------------------+---------------+-----------+-----------+-----------+---------+---------+----------+----------+-----------+-----------+--------+--------+---------+----------+
        | ALTER_TABLE       | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        | ANALYZE_TABLE     | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        | BEGIN             | 2212625       | 3686      | 55        | 2162      | 899     | 569     | 1        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        | CHANGE_MASTER     | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        | COMMIT            | 21522591      | 3628      | 0         | 0         | 0       | 1765    | 1590     | 272      | 1         | 0         | 0      | 0      | 0       | 0        |
        | CREATE_DATABASE   | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        | CREATE_INDEX      | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        ...
        | DELETE            | 2904130       | 3670      | 35        | 1546      | 1346    | 723     | 19       | 1        | 0         | 0         | 0      | 0      | 0       | 0        |
        | DESCRIBE          | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        ...
        | INSERT            | 19531649      | 3660      | 39        | 1588      | 1292    | 723     | 12       | 2        | 0         | 1         | 0      | 1      | 2       | 0        |
        ...
        | SELECT            | 35049794      | 51605     | 501       | 26180     | 16606   | 8241    | 70       | 3        | 4         | 0         | 0      | 0      | 0       | 0        |
        | SELECT_FOR_UPDATE | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        ...
        | UPDATE            | 6402302       | 7367      | 75        | 2503      | 3020    | 1743    | 23       | 3        | 0         | 0         | 0      | 0      | 0       | 0        |
        | USE               | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        | SHOW              | 19691         | 2         | 0         | 0         | 0       | 0       | 1        | 1        | 0         | 0         | 0      | 0      | 0       | 0        |
        | UNKNOWN           | 0             | 0         | 0         | 0         | 0       | 0       | 0        | 0        | 0         | 0         | 0      | 0      | 0       | 0        |
        +-------------------+---------------+-----------+-----------+-----------+---------+---------+----------+----------+-----------+-----------+--------+--------+---------+----------+
        52 rows in set (0.00 sec)

### Automatic Fail-over

ProxySQL will automatically detect if a node is not available or not synced with the cluster.

1. You can check the status of all available nodes by running:

        mysql@proxysql> SELECT hostgroup_id,hostname,port,status FROM mysql_servers;
        +--------------+---------------+------+--------+
        | hostgroup_id | hostname      | port | status |
        +--------------+---------------+------+--------+
        | 0            | 192.168.50.31 | 3306 | ONLINE |
        | 0            | 192.168.50.32 | 3306 | ONLINE |
        | 0            | 192.168.50.33 | 3306 | ONLINE |
        +--------------+---------------+------+--------+
        3 rows in set (0.00 sec)

2. To test problem detection and fail-over mechanism, shut down Node 3:

        ansible -bkKv -i hosts db3 -m systemd -a 'name=mysql state=stopped'

3. ProxySQL will detect that the node is down and update its status to **OFFLINE_SOFT**:

        mysql@proxysql> SELECT hostgroup_id,hostname,port,status FROM mysql_servers;
        +--------------+---------------+------+--------------+
        | hostgroup_id | hostname      | port | status       |
        +--------------+---------------+------+--------------+
        | 0            | 192.168.50.31 | 3306 | ONLINE       |
        | 0            | 192.168.50.32 | 3306 | ONLINE       |
        | 0            | 192.168.50.33 | 3306 | OFFLINE_SOFT |
        +--------------+---------------+------+--------------+
        3 rows in set (0.00 sec)

4. Now start Node 3 again:

        ansible -bkKv -i hosts db3 -m systemd -a 'name=mysql state=stopped'

5. The script will detect the change and mark the node as ONLINE:

        mysql@proxysql> SELECT hostgroup_id,hostname,port,status FROM mysql_servers;
        +--------------+---------------+------+--------+
        | hostgroup_id | hostname      | port | status |
        +--------------+---------------+------+--------+
        | 0            | 192.168.50.31 | 3306 | ONLINE |
        | 0            | 192.168.50.32 | 3306 | ONLINE |
        | 0            | 192.168.50.33 | 3306 | ONLINE |
        +--------------+---------------+------+--------+
        3 rows in set (0.00 sec)

## References

1. [Percona XtraDB Cluster 5.7 Documentation](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/index.html)
2. [Passwords management](https://github.com/sysown/proxysql/wiki/Passwords-management)
3. [How to set up read-write split in Galera Cluster using ProxySQL](http://severalnines.com/blog/how-set-read-write-split-galera-cluster-using-proxysql)
4. [MySQL Documentation](http://dev.mysql.com/doc/)
5. [Ansible Documentation](http://docs.ansible.com/ansible/)
