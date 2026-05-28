# Simple Hadoop Training Lab
Follow [the instructions](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Pseudo-Distributed_Operation) to set up a single-node cluster; however, I preferred the pseudo-distributed setup.

Both topics are covered in the document. The following notes only highlight some potentially confusing setup cases that may help during installation.

## Configuration files
Note that all configuration files are located in the ```etc``` directory inside the extracted Hadoop directory. The most common:

```console
etc/hadoop/hadoop-env.sh
```
Set the ```JAVA_HOME``` and other required environment variables permanently so they are available in every session.

```console
etc/hadoop/core-site.xml
```
Add following property to set hdfs filesytem scheme

```java
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>

```
And the replication factor property in the following config file:

```console
etc/hadoop/hdfs-site.xml
```
```java
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

## Prerequisites
1. The document requires passwordless SSH access to the NameNode (*```
localhost in this setup```*). Verify that ```ssh localhost``` works without a password prompt; otherwise, you may encounter errors while starting the cluster.

2. Create the following directories if they are not already present.
    ```console
    /tmp/hadoop-abamini/dfs/data
    /tmp/hadoop-abamini/dfs/name
    /tmp/hadoop-abamini/dfs/namesecondary
    ```
3. Add HADOOP_Home to your PATH
    ```console
    export HADOOP_HOME=/opt/hadoop/hadoop-3.5.0
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
    ```
## Start the Cluster
```console
$HADOOP_HOME/sbin/start-dfs.sh
```
## troubleshoot

Run jps command if cluster has been run correctly you'll see following process is running:
```console
SecondaryNameNode
DataNode
NameNode
```

To follow logs for other components use related wildcard:
```console
tail -n 50 $HADOOP_HOME/logs/hadoop-*-namenode-*.log
```
If you encounter log messages related to Namenode formatting issues...
```console
bin/hdfs namenode -format
```
