# Getting Started with TDP

Launch various virtual TDP big data infrastructures in less than 10 commands

# TL;DR
```bash
ansible-playbook deploy-infrastructure.yml deploy-ca.yml deploy-kerberos.yml deploy-zookeeper.yml deploy-users.yml deploy-hdfs-yarn-mapreduce.yml deploy-postgres.yml deploy-ranger.yml deploy-hive.yml -K
```
### Requirements
- ansible >= 2.9.6
- vagrant >= 2.29

## Deploy TDP

**Infrastructure**

The below command will create the core infrastructure expected by all tdp deployments documented in the guide.

```bash
ansible-playbook deploy-infrastructure.yml
```
It spawns and lightly configures a set of 8 virtual machines at static IPs below:

* worker-01 192.168.32.10
* worker-02 192.168.32.11
* worker-03 192.168.32.12
* master-01 192.168.32.13
* master-02 192.168.32.14
* master-03 192.168.32.15
* edge-01 192.168.32.16

**Note that:**
- To change the static IPs, update **both** the `Vagrantfile` and the `tdp-hosts` files
- To edit the resources assigned to the vms, update the `Vagrantfile`

*Check the status of the created VMs with the command `vagrant status`, and ssh to them with the cammand `vagrant ssh <target vm name>`*

**Certificate Authority and Certificates**

Creates a certificate authority at the `[ca]` ansible group and distributes signed certificates and keys to each VM.

```
ansible-playbook deploy-ca.yml
```

*The certificates will also be downloaded to the `files/certs` local project folder.*

***Kerberos***

Launches a KDC at the `[kdc]` ansible group and installs kerberos clients on each of the VMs.

```
ansible-playbook deploy-kerberos.yml
```

*After this, you can login as the kerberos admin from any VM with the command `kinit admin/admin` and the passwork `admin`.*

**Create Cluster Users**

The below command creates:
  - Unix users *tdp-user* and *tdp-admin* on each node of the cluster
  - A kerberos principal named `<user>/<fqdn>@<realm>` with keytabs at `/home/<user>/.ssh/<user>.kerberos.keytab`
  - All users are added to the users group
  - Users with 'admin' in the name will also be added to the group 'tdp-admin'

```
ansible-playbook deploy-users.yml
```

*Additional users can be added to the ansible-playbook parameter **users** in the `deploy-users.yml` if required*

**Zookeeper**

Deploys Apache ZooKeeper to the the `[zk]` ansible group and starts a 3 node Zookeeper quorum.
  
```
ansible-playbook deploy-zookeeper.yml
```

*Run `echo stat | nc localhost 2181` from any node in the `[zk]` group to see it's zookeeper status.*

**Launch HDFS, Yarn & MapReduce**

Launches a high availability hdfs distributed filesystem. 

```
ansible-playbook deploy-hdfs-yarn-mapreduce.yml -K
```

The following code snippets demonstrate that:
  - The namenode kerberos principal can create an appropriate hdfs user directory for tdp-user:
    
    - *From master-01.tdp:*

      ```bash
      kinit -kt /etc/security/keytabs/nn.service.keytab nn/master-01.tdp@REALM.TDP
      /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf.nn dfs -mkdir -p /user/tdp-user
      /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf.nn dfs -chown -R tdp-user:tdp-user /user/tdp-user
      ```

  - That tdp-user can access and write to their hdfs user directory:

    - *From edge-01.tdp:*

        ```bash
        su tdp-user
        kinit -kt /home/tdp-user/.ssh/tdp-user.principal.keytab tdp-user/edge-01.tdp@REALM.TDP
        echo "This is the first line." | /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf dfs -put - /user/tdp-user/testFile
        echo "This is the second (appended) line." | /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf dfs -appendToFile - /user/tdp-user/testFile
        /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf dfs -cat /user/tdp-user/testFile
        ```

  - That writes using the tdp-user from edge-01.tdp can be read from master-01.tdp:

    - *On master-01.tdp:*

      ```bash
      su tdp-user
      kinit -kt /home/tdp-user/.ssh/tdp-user.principal.keytab tdp-user/master-01.tdp@REALM.TDP
      /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf.nn dfs -cat /user/tdp-user/testFile
      ```

**Postgres**

Deploys postgres instance to `[postgres]` ansible group. Listens for request on from all IPs but but only trusts those specified in the /etc/hosts file.

The DBA user **postgres** is created with the password **postgres**.

```bash
ansible-playbook deploy-postgres.yml
```


**Ranger**

Creates a suitably configured postgres database to the `[postgresql]` ansible group, then deploys Ranger to the `[ranger_admin]` ansible group.

*Note that any changes to the `[ranger_admin]` hosts should be also be reflected in the `[hadoop client group`]*
  

```
ansible-playbook deploy-ranger.yml
```

The Ranger UI can be accessed at the address https://<master-02.tdp ip>:6182/login.jsp and the user `admin` and password `RangerAdmin123` (assuming default *ranger_admin_password* parameter). You may bneed to import the `root.pem` certificate authority into your browser to access.

**Hive**

Deploys hive to the `[hive_s2]` ansible group. HDFS filesystem is created and the service is launched.

```
ansible-playbook deploy-hive.yml
```

*Execute the following code blocks to execute some hive queries using beeline:*

The following code snippets:
  - Create an hdfs user directory for tdp-user (this block is is the same as in the deploy hdfs example above):

    - *From master-01.tdp:*

        ```bash
        kinit -kt /etc/security/keytabs/nn.service.keytab nn/master-01.tdp@REALM.TDP
        /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf.nn dfs -mkdir -p /user/tdp-user
        /opt/tdp/hadoop/bin/hdfs --config /etc/hadoop/conf.nn dfs -chown -R tdp-user /user/tdp-user
        ```

  - Authenticate as tdp-user from one of the hive_s2 nodes and enter the beeline client interface:
  
    - *From master-02.tdp:*

        ``bash
        su tdp-user
        export hive_truststore_password=Truststore123!
        # Either via zookeeper
        kinit -kt /home/tdp-user/.ssh/tdp-user.principal.keytab tdp-user/master-02.tdp@REALM.TDP
        /opt/tdp/hive/bin/hive --config /etc/hive/conf.s2 --service beeline -u "jdbc:hive2://master-01.tdp:2181,master-02.tdp:2181,master-03.tdp:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;sslTrustStore=/etc/ssl/certs/truststore.jks;trustStorePassword=${hive_truststore_password}"
        # Or directly to a hiveserver
        /opt/tdp/hive/bin/hive --config /etc/hive/conf.s2 --service beeline -u "jdbc:hive2://master-02.tdp:10001/;principal=hive/_HOST@REALM.TDP;transportMode=http;httpPath=cliservice;ssl=true;sslTrustStore=/etc/ssl/certs/truststore.jks;trustStorePassword=${hive_truststore_password}"
        ```
  
  - As there is no ranger user sync configured in this cluster, add the tdp-user manually in the ranger UI at `https://master-02.tdp:6182/index.html` with `admin` and `RangerAdmin123` user and password (assuming default settings used). 
      - Go to *RangerUI > Settings > users/Groups/Roles > Add New User* and create tdp-user
      - Go to  *RangerUI > Service Manager > hive-tdp Policies* and create a policy to allow tdp-user full permissions to database tdp_user_db
      - Go to  *RangerUI > Service Manager > hdfs-tdp Policies* and create a policy to allow tdp-user full read, write and execute permissions in  `/user/tdp-user` hdfs dir
      - Go to  *RangerUI > Service Manager > hdfs-tdp Policies* and create a policy to hive user read, write and execute permissions in  `/user` hdfs dir

From the beeline client, execute the following code blocks to interact with Hive:

```bash
# Create the database
CREATE DATABASE tdp_user_db;
USE tdp_user_db;

# Examine the database
SHOW DATABASES;
SHOW TABLES;

# Modify the database
CREATE TABLE IF NOT EXISTS table1
 (col1 int COMMENT 'Integer Column',
 col2 string COMMENT 'String Column'
 );

# Examine the database
SHOW TABLES;

# Modify the database table
INSERT INTO TABLE table1 VALUES (1, 'one'), (2, 'two');

# Examine the database table
SELECT * FROM table1;
```