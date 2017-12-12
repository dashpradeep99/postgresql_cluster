# Building A Highly Available Multi-Node PostgreSQL Cluster

## Overview
---
Building a highly avialable multi-node PostgreSQL cluster, using freely available software including [Pacemaker](http://clusterlabs.org/), [Corsync](http://corosync.github.io/corosync/), [Cman](http://www.sourceware.org/cluster/cman/) and [PostgresSQL](http://www.postgresql.org/) on [CentOS](http://www.centos.org/)

##### Infrastructure
Three node HotStandby HA cluster

![](https://raw.githubusercontent.com/smbambling/pgsql_ha_cluster/459a1525c2a75ffe632edc10c77d540ed5495aeb/postgresql_cluster_pacemaker.png)
## HA Cluster Installation
---
The required are avilable and included in the base/updates repositories for Centos 6.x.

From my readings and research it is also possible to use heartbeat 3.x with Pacemaker to achive similar results.  I've decided to go with Corosync as its backed by Red Hat and Suse and it looks to have more active development.  Not to memtion that the Pacemaker projects recommends you should use Corosync.

### Cluster Installation
---

**!!! Notice !!!**

> As of RedHat/CentOS 6.4 crmsh is no longer included in the default repositories.  If you want to use CRM vs PCS You can include the OpenSuse repositories [HERE](http://download.opensuse.org/repositories/network:/ha-clustering:/Stable/RedHat_RHEL-6/). More information on the crmsh can be found [HERE](https://savannah.nongnu.org/forum/forum.php?forum_id=7503)

In this turtoial we will add the openSUSE repository to our nodes.  Though I recommend building or copying these packages into a local repository for more controlled management.

Configure the openSUSE repository

````
sudo wget -4 http://download.opensuse.org/repositories/network:/ha-clustering:/Stable/RedHat_RHEL-6/network:ha-clustering:Stable.repo -O /etc/yum.repos.d/network_ha-clustering_Stable.repo
````

Limit the packages to be installed from the openSUSE repository

````
sudo echo "includepkgs=crmsh pssh" >> /etc/yum.repos.d/network_ha-clustering_Stable.repo
````

Now that we have the required repositories configured we need to install the needed packages.

**You will see multiple depenencies being pulled in**

````
sudo yum install pacemaker pcs corosync fence-agents crmsh cman css
````

### Cluster Configuration
---

The first step is to configure the underlying Cman/Corosync cluster ring communication between the nodes and setup Pacemaker to use Corosync as its communication mechanisum.

For secure communication Corosync requires an pre-shared authkey.  This key must be added to all nodes in the cluster.

To generate the authkey Corosync has a utility corosync-keygen. Invoke this command as the root users to generate the authkey. The key will be generated at /etc/corosync/authkey.  You only need to perform this action on **one** of the nodes in the cluster as we'll copy it to the other nodes

**Grab a cup of coffee this process takes a while to complete as it pulls from the more secure /dev/random. You don’t have to press anything on the keyboard it will still generate the authkey**

Once the key has been generated copy it to the other nodes in the cluster

````
sudo scp /etc/corosync/authkey root@node2:/etc/corosync/
sudo scp /etc/corosync/authkey root@node3:/etc/corosync/
````

In multiple examples and documents on the web they reference using the packmaker corosync plugin by adding a /etc/corosync/service.d/pcmk configure file on each node.  This is becoming deprecated and will show in the logs if you enable or use the corosync pacemaker plugin.  There is a small but important distinction that I stumbled upon, the pacemaker plugin has never been supported on RHEL systems.  

The real issue is that at some point it will no longer be supplied with the packages on RHEL systems.  Prior to 6.4 ( Though this is looking to change in 6.5 and above ), pacemaker only had a tech preview status for the plugin and using the CMAN plugin instead.

Reference this [wiki article](http://floriancrouzat.net/2013/04/rhel-6-4-pacemaker-1-1-8-adding-cman-support-and-getting-rid-of-the-plugin/)

Disable quorum in order to allow Cman/Corosync to complete startup in a standalone state.  

This need to be done on **ALL** nodes in the cluster.

````
sudo sed -i.sed "s/.*CMAN_QUORUM_TIMEOUT=.*/CMAN_QUORUM_TIMEOUT=0/g" /etc/sysconfig/cman
````

Define the cluster, where *pg_cluster* is the cluster name.  This will generate the cluster.conf configuration file.  

This only needs to be run on **one** node, we'll copy it to the other nodes.  For simplicity we will run this on node1 in the cluster

````
sudo ccs -f /etc/cluster/cluster.conf --createcluster pg_cluster
````

Create the cluster redundant ring(s).  The name used for each node in the cluster shoudl correspond to the nodes network hostname ``uname -n``.  

This only needs to be run on **one** node, we'll copy it to the other nodes.  For simplicity we will run this on node1 in the cluster

````
sudo ccs -f /etc/cluster/cluster.conf --addnode node1.example.com
sudo ccs -f /etc/cluster/cluster.conf --addnode node2.example.com
sudo ccs -f /etc/cluster/cluster.conf --addnode node3.example.com
````

Configure the fence_pcmk agent (supplied with Pacemaker) to redirect any fencing requests from CMAN components (such as dlm_controld) to Pacemaker. Pacemaker’s fencing subsystem lets other parts of the stack know that a node has been successfully fenced, thus avoiding the need for it to be fenced again when other subsystems notice the node is gone.  

This only needs to be run on **one** node, we'll copy it to the other nodes.  For simplicity we will run this on node1 in the cluster

````
sudo ccs -f /etc/cluster/cluster.conf --addmethod pcmk-redirect node1.example.com
sudo ccs -f /etc/cluster/cluster.conf --addmethod pcmk-redirect node2.example.com
sudo ccs -f /etc/cluster/cluster.conf --addmethod pcmk-redirect node3.example.com
````
````
sudo ccs -f /etc/cluster/cluster.conf --addfencedev pcmk agent=fence_pcmk
````
````
sudo ccs -f /etc/cluster/cluster.conf --addfenceinst pcmk node1.example.com pcmk-redirect port=node1.example.com
sudo ccs -f /etc/cluster/cluster.conf --addfenceinst pcmk node2.example.com pcmk-redirect port=node2.example.com
sudo ccs -f /etc/cluster/cluster.conf --addfenceinst pcmk node3.example.com pcmk-redirect port=node3.example.com
````

Enable secure communciation between nodes in the Corosync cluster using the corosync authkey generated above.  

This only needs to be run on **one** node, we'll copy it to the other nodes.  For simplicity we will run this on node1 in the cluster

````
sudo ccs -f /etc/cluster/cluster.conf --setcman keyfile="/etc/corosync/authkey" transport="udpu"
````

Before we copy the configuration to the other nodes in the cluster lets verify that the generated configuration values are vaild.  

This should be run on the same node as the pervious commmands.  For simplicity we will run this on node1 in the cluster

````
sudo ccs_config_validate -f /etc/cluster/cluster.conf
````

Start the Cman/Corosync cluster services

This only needs to be run on **one** node, we'll copy it to the other nodes.  For simplicity we will run this on node1 in the cluster

````
sudo /etc/init.d/cman start
Starting cluster:
   Checking if cluster has been disabled at boot...        [  OK  ]
   Checking Network Manager...                             [  OK  ]
   Global setup...                                         [  OK  ]
   Loading kernel modules...                               [  OK  ]
   Mounting configfs...                                    [  OK  ]
   Starting cman...                                        [  OK  ]
   Waiting for quorum...                                   [  OK  ]
   Starting fenced...                                      [  OK  ]
   Starting dlm_controld...                                [  OK  ]
   Tuning DLM kernel config...                             [  OK  ]
   Starting gfs_controld...                                [  OK  ]
   Unfencing self...                                       [  OK  ]
   Joining fence domain...                                 [  OK  ]
````

Note that starting Cman also starts the Corosync service.  This can be easily verified via the Corosync init script

````
sudo /etc/init.d/corosync status
corosync (pid  18376) is running...
````

Start the Pacemaker cluster service

This only needs to be run on **one** node, we'll copy it to the other nodes.  For simplicity we will run this on node1 in the cluster

````
sudo /etc/init.d/pacemaker start
Starting Pacemaker Cluster Manager                         [  OK  ]
````

Before continuing verify that all services have correctly started and are running.

````
sudo /etc/init.d/cman status
cluster is running.
````
````
sudo /etc/init.d/corosync status
corosync (pid  615) is running...
````
````
sudo /etc/init.d/pacemaker status
pacemakerd (pid  868) is running...
````

After the initial node has been successfully configured and services have started copy the cluster.conf to the other nodes in the cluster 

````
sudo scp /etc/cluster/cluster.conf node2.example.com:/etc/cluster/cluster.conf
sudo scp /etc/cluster/cluster.conf node3.example.com:/etc/cluster/cluster.conf
````

Start the Cman/Corosync services on additional nodes in the cluster.

````
sudo /etc/init.d/cman start
Starting cluster:
   Checking if cluster has been disabled at boot...        [  OK  ]
   Checking Network Manager...                             [  OK  ]
   Global setup...                                         [  OK  ]
   Loading kernel modules...                               [  OK  ]
   Mounting configfs...                                    [  OK  ]
   Starting cman...                                        [  OK  ]
   Waiting for quorum...                                   [  OK  ]
   Starting fenced...                                      [  OK  ]
   Starting dlm_controld...                                [  OK  ]
   Tuning DLM kernel config...                             [  OK  ]
   Starting gfs_controld...                                [  OK  ]
   Unfencing self...                                       [  OK  ]
   Joining fence domain...                                 [  OK  ]
````

Start the Pacemaker service on additional nodes in the cluster

````
sudo /etc/init.d/pacemaker start
Starting Pacemaker Cluster Manager                         [  OK  ]
````

Before continuing verify that all services have correctly started and are running on the additional nodes in the cluster.

````
sudo /etc/init.d/cman status
cluster is running.
````
````
sudo /etc/init.d/corosync status
corosync (pid  615) is running...
````
````
sudo /etc/init.d/pacemaker status
pacemakerd (pid  868) is running...
````

Verify that **ALL** nodes in the cluster are communicating.  The cman_tool and crm_mon commands can be used to get the status of the Cman/Corosync cluster ring and Pacemaker HA cluster respectivly.

View cluster ring Cman/Corosync status

````
sudo cman_tool nodes -a
Node  Sts   Inc   Joined               Name
   1   M      4   2014-04-09 08:30:22  node1.example.com
       Addresses: 10.4.10.60
   2   M      8   2014-04-09 08:44:01  node2.example.com
       Addresses: 10.4.10.61
   3   M     12   2014-04-09 08:44:08  node3.example.com
       Addresses: 10.4.10.62
````

View Pacemaker HA cluster status

````
sudo pcs status
Cluster name: pg_cluster
Last updated: Thu Apr 10 07:39:08 2014
Last change: Thu Apr 10 06:49:19 2014 via cibadmin on node1.example.com
Stack: cman
Current DC: node1.example.com - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
3 Nodes configured
0 Resources configured


Online: [ node1.example.com node2.example.com node3.example.com ]

Full list of resources:
````

#### Pacemaker Cluster Configuration

At this point we have configured the basic cluster communication ring.  All nodes are now communicating and reporting their heartbeat status to each of the nodes via Corosync.

Verify the Pacemaker cluster configuration.  Here you'll notice the cluster is complaining that STONITH (Shoot The Other Node In The Head) is not configured.  

````bash
sudo pcs cluster verify -V
   error: unpack_resources:     Resource start-up disabled since no STONITH resources have been defined
   error: unpack_resources:     Either configure some or disable STONITH with the stonith-enabled option
   error: unpack_resources:     NOTE: Clusters with shared data need STONITH to ensure data integrity
Errors found during check: config not valid
````

For now we are going to disable this and come back to it later in the tutorial.  This only needs to be run on one node of the cluster as they are syncing configurations between the nodes.

````bash
sudo pcs property set stonith-enabled=false
````

Verify the Pacemaker stonith property was correctly configured.

````bash
sudo pcs config
Cluster Name: pg_cluster
Corosync Nodes:

Pacemaker Nodes:
 node1.example.com node2.example.com node3.example.com

Resources:

Stonith Devices:
Fencing Levels:

Location Constraints:
Ordering Constraints:
Colocation Constraints:

Cluster Properties:
 cluster-infrastructure: cman
 dc-version: 1.1.10-14.el6_5.2-368c726
 stonith-enabled: false
````

Now verifying the Pacemaker cluster configuration again returns no errors.

````
sudo pcs cluster verify -V
````

#### Pacemaker IP Resources

Now that we have a basic cluster configuration setup we can focus on adding some resources for the cluster to manage.  The first resource to add is a cluster IP or "VIP" so that applications will be able to continuously communicate with the cluster regardless of where the cluster services are running.

**Notice** : Replace the **ip** and **cidr_netmask** parameters with the correct address for your cluster.

Create a IP resources "VIPs" using the **ocf:heartbeat:IPaddr2** resource agent 'script'.

Create Replication "VIP"

````bash
sudo pcs resource create pgrepvip ocf:heartbeat:IPaddr2 ip=10.4.10.63 cidr_netmask=24 iflabel="pgrepvip" op monitor interval=1s meta target-role="Started"
````

Create Client Access "VIP"

````bash
sudo pcs resource create pgclivip ocf:heartbeat:IPaddr2 ip=10.4.10.64 cidr_netmask=24 iflabel="pgclivip" op monitor interval=1s meta target-role="Started"
````

Verify the Pacemaker cluster resource has been correctly added to the cluster information base (CIB).

````bash
sudo pcs config
Cluster Name: pg_cluster
Corosync Nodes:

Pacemaker Nodes:
 node1.example.com node2.example.com node3.example.com

Resources:
 Resource: pgrepvip (class=ocf provider=heartbeat type=IPaddr2)
  Attributes: ip=10.4.10.63 cidr_netmask=24 iflabel=pgrepvip
  Meta Attrs: target-role=Started
  Operations: monitor interval=1s (pgrepvip-monitor-interval-1s)
 Resource: pgclivip (class=ocf provider=heartbeat type=IPaddr2)
  Attributes: ip=10.4.10.64 cidr_netmask=24 iflabel=pgclivip
  Meta Attrs: target-role=Started
  Operations: monitor interval=1s (pgclivip-monitor-interval-1s)

Stonith Devices:
Fencing Levels:

Location Constraints:
Ordering Constraints:
Colocation Constraints:

Cluster Properties:
 cluster-infrastructure: cman
 dc-version: 1.1.10-14.el6_5.2-368c726
 stonith-enabled: false
````

View the running status of the cluster.  Here we can see that both the IP resources "VIPs" are running. 

````
sudo pcs status
Cluster name: pg_cluster
Last updated: Thu Apr 10 08:04:14 2014
Last change: Thu Apr 10 07:53:03 2014 via cibadmin on node1.example.com
Stack: cman
Current DC: node1.example.com - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
3 Nodes configured
2 Resources configured


Online: [ node1.example.com node2.example.com node3.example.com ]

Full list of resources:

 pgrepvip       (ocf::heartbeat:IPaddr2):       Started node1.example.com
 pgclivip       (ocf::heartbeat:IPaddr2):       Started node2.example.com
````

The pgclivip IP resource "VIP" was started on node2, for simplicity we will move it to node1.

````
sudo pcs resource move pgclivip node1.example.com
````

Viewing the running status of the cluster again we can see both resources are now running on node1

````
sudo pcs status
Cluster name: pg_cluster
Last updated: Thu Apr 10 08:11:48 2014
Last change: Thu Apr 10 08:06:56 2014 via crm_resource on node1.example.com
Stack: cman
Current DC: node1.example.com - partition with quorum
Version: 1.1.10-14.el6_5.2-368c726
3 Nodes configured
2 Resources configured


Online: [ node1.example.com node2.example.com node3.example.com ]

Full list of resources:

 pgrepvip       (ocf::heartbeat:IPaddr2):       Started node1.example.com
 pgclivip       (ocf::heartbeat:IPaddr2):       Started node1.example.com
````

#### PostgreSQL Database Configuration

Before adding a Pacemaker pgsql resource to manage the PostgreSQL services, its recommended to setup the PostgreSQL cluster (The PostgreSQL internal cluster) with some basic streaming replication.

The version of PostgreSQL that is in the provided repositories on CentOS 6.5 is 8.4.20 which does not provide the needed streaming replication.  To work around this we will add PGDG (PostgreSQL Global Development Group) repository.  

As of this writing we are using PostgreSQL version 9.3.4

Configure the needed repository.  

This need to be done on **ALL** nodes in the cluster.

````
sudo wget http://yum.postgresql.org/9.3/redhat/rhel-6-x86_64/pgdg-centos93-9.3-1.noarch.rpm -O /tmp/pgdg-centos93-9.3-1.noarch.rpm

sudo rpm -Uvh /tmp/pgdg-centos93-9.3-1.noarch.rpm
````

With the correct repository configured install the recommended packages.

````
sudo yum install postgresql93-server postgresql93-contrib postgresql93-devel
````

Initialize the PostgreSQL database via initdb.  We only need to perform this on the **Master** node as we'll be transfering the database to the remaing nodes.  I'll be refering to these nodes as PostgreSQL replicas (node2, node3).  

We will use node1 as the **Master** from here on out.

````
sudo /etc/init.d/postgresql-9.3 initdb
````

Once the initialization is succesfull you'll see the PostgreSQL data directory populated.  On CentOS this is located in /var/lib/pgsql/{version (9.3)}/data

````
ls /var/lib/pgsql/9.3/data/
base  global  pg_clog  pg_hba.conf  pg_ident.conf  pg_log  pg_multixact  pg_notify  pg_serial  pg_snapshots  pg_stat  pg_stat_tmp  pg_subtrans  pg_tblspc  pg_twophase  PG_VERSION  pg_xlog  postgresql.conf
````

When the database was initialized via initdb it configued permissions in the pg_hba.conf.  This uses the not so popular ident scheme to determine if a user is allowed to connect to the database.

**ident**: An authentication schema that relies on the currently logged in user. If you’ve su -s to postgres and then try to login as another user, ident will fail (as it’s not the currently logged in user).

This can be a sore spot if your not aware how it was configured and will give an error if trying to create a database with a user that is not currently logged into the system.
> createdb: could not connect to database postgres: FATAL:  Ident authentication failed for user "myUser"

To avoid this headache modify the pg_hba.conf file to move from the ident scheme to the md5 scheme.  

This needs to be modified on the **Master** node1

````
sudo sed -i 's/\ ident/\ md5/g' /var/lib/pgsql/9.3/data/pg_hba.conf
````
````
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
````

Modify the pg_hba.conf to allow the repclias to connect to the **Master**.  Again in this example we are adding a basic connection line.  It is recommended that you tune this based on your infrasturture for proper secutiry.  

This needs to be modified on the **Master** node1

````
cat << EOF >> /var/lib/pgsql/9.3/data/pg_hba.conf

# Allowing Replicas to connect in for streaming replication
host    replicator     all             all                     trust
EOF
````

To assit with the replication process and some basic security, a seperate replication user account should be created.

````
sudo runuser -l postgres -c "psql -c \"CREATE USER replicator REPLICATION LOGIN ENCRYPTED PASSWORD 'replaceme';\""
````

Configure the bind address the PostgreSQL will listen on.    This needs to be set to * so the PostgreSQL service will listen on any address.  PostgreSQL will scan for new addresses and automatically bind to them as they appear on the node.  This is required to allow PostgreSQL to start listening on the **VIP** address in the event of node failover.

Modify the postgresql.conf with your favorite text editor and modify the listen_addresses parameter or add an additional parameter to the end of the configureation file. 

This needs to be modified on the **Master** node1

````
sudo echo "listen_addresses = '*'" >> /var/lib/pgsql/9.3/data/postgresql.conf
````

Configure PostgreSQL steaming replication.  This example uses a simple archive command its recommned to use a wrapper script to assist in syncing the archived WAL logs to the other nodes in the cluster.  These settings are very basic and will need to be tuned based on your infrastructure.  

This needs to be modified on the **Master** node1

````
cat << EOF >> /var/lib/pgsql/9.3/data/postgresql.conf
wal_level = hot_standby
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/9.2/archive/%f'
max_wal_senders = 3
wal_keep_segments = 100
hot_standby = on
EOF
````

The PostgreSQL has the concept of archiving for its WAL logs.  Its recommended to create a seperate archive directory, this will be used to store and recover archived WAL logs.  In this example we will create this in the currnet PostgreSQL {version} directory, you can create this anywhere.

This need to be done on **ALL** nodes in the cluster.

````
sudo runuser -l postgres -c 'mkdir /var/lib/pgsql/9.3/archive'
````

Start the PostgreSQL service and check for erros in /var/lib/pgsql/9.3/pg_log/*.

````
sudo /etc/init.d/postgresql-9.3 start
Starting postgresql-9.3 service:                           [  OK  ]
````
````
tail -F -n3 /var/lib/pgsql/9.3/data/pg_log/postgresql-Fri.log
< 2014-04-11 07:09:26.780 EDT >LOG:  database system was shut down at 2014-04-11 06:29:14 EDT
< 2014-04-11 07:09:26.793 EDT >LOG:  database system is ready to accept connections
< 2014-04-11 07:09:26.793 EDT >LOG:  autovacuum launcher started
````

Clone the PostgreSQL database cluster from the **Master** node to the replicas (node2, node3).  We are using a modern version of PostgreSQL that include pg_basebackup, which makes the process 1000000000 time simpilar.  

You can also use pg_start_backup, rsync and pg_stop_backup to perform a more manaul cloning.

On the replica nodes (node2, node3) run the pg_basebackup command

````
sudo runuser -l postgres -c 'pg_basebackup -D /var/lib/pgsql/9.3/data -l `date +"%m-%d-%y"`_initial_cloning -P -h node1.example.com -p 5432 -U replicator -W -X stream'
````

To avoid any confusion with troubleshooting remove the log files that were transfered during the pg_basebackup process.  This needs to be done on both replicas

````
sudo rm /var/lib/pgsql/9.3/data/pg_log/*
````

In order for the replicas to connect to the **Master** for streaming replication a recovery.conf file must exist in the PostgreSQL data directory.

Create a recovery.conf file on both replicas (node2, node3)

````
cat << EOF >> /var/lib/pgsql/9.3/data/recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=10.4.10.63 port=5432 user=replicator application_name=`hostname`'
restore_command = 'cp /var/lib/pgsql/9.2/archive/%f "%p"'
EOF
````

Start the PostgreSQL service on both of the replicas (node2, node3)

````
sudo /etc/init.d/postgresql-9.3 start
````

On the **Master** verify and view the active replica connections and their status.  You'll notice the sync_state is async for both nodes, this is because we have not set the standby_node_names parameter on the master to let it know what nodes it should attempt to perform synchronous replication with.  You'll also notice the state is streaming, this is continually sending changes to the replicas/slaves without waiting for WAL segments to fill and then be shipped.

````
sudo runuser -l postgres -c "psql -c \"SELECT application_name, client_addr, client_hostname, sync_state, state, sync_priority, replay_location FROM pg_stat_replication;\""
    application_name    | client_addr | client_hostname | sync_state |   state   | sync_priority | replay_location
------------------------+-------------+-----------------+------------+-----------+---------------+-----------------
 node2.example.com | 10.4.10.61  |                 | async      | streaming |             0 | 0/40000C8
 node3.example.com | 10.4.10.62  |                 | async      | streaming |             0 | 0/40000C8
````








