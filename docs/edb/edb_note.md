# EDB Cluster Installation

## 1. Installation preparation
## 2. Register a trail account from EDB portal

+   EDB Account: `edb.hua.2022@gmail.com`    
+   Sign-in & Repository Username: `Wah1111`  
+   Repository Password: `PMIS3LwpRo5iX2tW`  

## 3 Preparation for the Virtual machine

+  Master:  Node01 : 192.168.99.146  (Master node)  
+  Standby: Node02 : 192.168.99.147  (Standby node)  
+  Witness: Node03 : 192.168.99.148  (Witness node)  


### 3.1 Update/edit as follows for static IP configuration:


```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens33

```
Append the below two rows into ifcfg-ens33 to apply the fixed IP address
+  IPADDR=192.168.99.146 (147,148)  
+  PREFIX=24  

Save and close the file. To restart networking service

```
systemctl restart network
```

### 3.2 Change the Hostname:
```shell
vi /etc/hosts
```

192.168.99.146 node01 node01.edb.demo  
192.168.99.147 node02 node02.edb.demo  
192.168.99.148 node03 node03.edb.demo  

### 3.4 Turn off the firewall on CentOS
view the current status of the Firewalld service
```
systemctl status firewalld
```

disable the Firewalld service to start automatically on system reboot
```
systemctl diable firewalld
```

> Or adding the Postgresql service in the firewalld configuration to allow requests from the standby node to master
```
# firewall-cmd --add-service=postgresql --permanent
# firewall-cmd --reload
```


## 4 Installation 

### 4.1 Installthe EDB Postgres Advanced Server via Yum on both Node01 and Node02

``` shell
yum -y install https://yum.enterprisedb.com/edbrepos/edb-repo-latest.noarch.rpm

# Replace 'USERNAME:PASSWORD' below with your username and password for the EDB repositories
# Visit https://www.enterprisedb.com/user to get your username and password
# sed -i "s@<username>:<password>@Wah1111:PMIS3LwpRo5iX2tW@" /etc/yum.repos.d/edb.repo
sed -i "s@<username>:<password>@USERNAME:PASSWORD@" /etc/yum.repos.d/edb.repo

# Install EPEL repository
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# Install selected packages
yum -y install edb-as12-server 

# Initialize Database cluster
PGSETUP_INITDB_OPTIONS="-E UTF-8" /usr/edb/as12/bin/edb-as-12-setup initdb

# Start Database server
systemctl start edb-as-12

# Connect to the database server
# sudo su - enterprisedb
# psql postgres
```


### 2.3 Setup the Streaming Replication

#### A. update the client authentication configuration file
Update the **/var/lib/edb/as12/data/pg_hba.conf** client authentication configuration file  to allow standy node connect to master onde as shown.


```
host    replication     all             0.0.0.0/0               md5
```

#### B. Backup and data directory and delete it on Slave node

**1.** figure out the data direcotry on the slave node02

```
postgres=#show data_directory;


ostgres=# show data_directory;
     data_directory
------------------------
 /var/lib/edb/as12/data
(1 row)

-bash-4.2$ cp -R /var/lib/edb/as12/data /var/lib/edb/as12/data_bak
-bash-4.2$ rm -rf /var/lib/edb/as12/data 


```

**2.** Make a base backup of the master server   
```

[root@node02 data]# su - enterprisedb 
-bash-4.2$pg_basebackup -h node01 -U enterprisedb -p 5444 -D /var/lib/edb/as12/data -P -Fp -Xs -v -R -C -S replicate_solt_a
Password:
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/A000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created replication slot "replicate_solt_a"
66058/66058 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/A000100
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: base backup completed


```
**Reference:**
In the following command, the option:

pg_basebackup takes a base backup of a running PostgreSQL server.

**Usage:**  
  pg_basebackup [OPTION]...


```
Options controlling the output:
  -D, --pgdata=DIRECTORY receive base backup into directory  
  -F, --format=p|t       output format (plain (default), tar)
  -r, --max-rate=RATE    maximum transfer rate to transfer data directory
                         (in kB/s, or use suffix "k" or "M")
  -R, --write-recovery-conf
                         write configuration for replication
  -T, --tablespace-mapping=OLDDIR=NEWDIR
                         relocate tablespace in OLDDIR to NEWDIR
      --waldir=WALDIR    location for the write-ahead log directory
  -X, --wal-method=none|fetch|stream
                         include required WAL files with specified method
  -z, --gzip             compress tar output
  -Z, --compress=0-9     compress tar output with given compression level

General options:
  -c, --checkpoint=fast|spread
                         set fast or spread checkpointing
  -C, --create-slot      create replication slot
  -l, --label=LABEL      set backup label
  -n, --no-clean         do not clean up after errors
  -N, --no-sync          do not wait for changes to be written safely to disk
  -P, --progress         show progress information
  -S, --slot=SLOTNAME    replication slot to use
  -v, --verbose          output verbose messages
  -V, --version          output version information, then exit
      --no-slot          prevent creation of temporary replication slot
      --no-verify-checksums
                         do not verify checksums
  -?, --help             show this help, then exit

Connection options:
  -d, --dbname=CONNSTR   connection string
  -h, --host=HOSTNAME    database server host or socket directory
  -p, --port=PORT        database server port number
  -s, --status-interval=INTERVAL
                         time between status packets sent to server (in seconds)
  -U, --username=NAME    connect as specified database user
  -w, --no-password      never prompt for password
  -W, --password         force password prompt (should happen automatically)
```

**3.** Validate the streaming replication

3.1. Execute the command on the Master node01   

```
[root@node01 data]# su - enterprisedb
Last login: Tue Dec  7 06:40:33 PST 2021 on pts/0
-bash-4.2$ psql postgres
psql (12.9.13)

postgres=# select pid,state,client_addr,sync_priority,sync_state from pg_stat_replication;

  pid  |   state   |  client_addr   | sync_priority | sync_state
-------+-----------+----------------+---------------+------------
 50235 | streaming | 192.168.99.147 |             0 | async
 
 
postgres=# SELECT * FROM pg_replication_slots;
    slot_name     | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn
------------------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------
 replicate_solt_a |        | physical  |        |          | f         | t      |      50235 |      |              | 0/B000060   |

```

3.2. Execute the command on the Slave node02   

```
postgres=# \x on
Expanded display is on.
postgres=# select * from pg_stat_wal_receiver;
-[ RECORD 1 ]---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 3131
status                | streaming
receive_start_lsn     | 0/B000000
receive_start_tli     | 1
received_lsn          | 0/B001880
received_tli          | 1
last_msg_send_time    | 08-DEC-21 00:28:00.870025 -08:00
last_msg_receipt_time | 08-DEC-21 00:28:00.870489 -08:00
latest_end_lsn        | 0/B001880
latest_end_time       | 07-DEC-21 07:55:03.035988 -08:00
slot_name             | replicate_solt_a
sender_host           | node01
sender_port           | 5444
conninfo              | user=enterprisedb password=******** dbname=replication host=node01 port=5444 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
postgres=# \x off
Expanded display is off.

```




**Reference URL**   
>  1. Step By Step Streaming Replication – PostgreSQL 12   
https://huzefapatel.com/blogs/step-by-step-streaming-replication-postgresql-12/  
>  2. How To Configure PostgreSQL 12 Streaming Replication in CentOS 8  
https://www.tecmint.com/configure-postgresql-streaming-replication-in-centos-8/  
>  3. PostgreSQL HOT STANDBY using Stream replication with testing  
https://developer.aliyun.com/article/14712
>  4.pg_basebackup物理备份日常使用姿势   
https://www.modb.pro/db/50395

