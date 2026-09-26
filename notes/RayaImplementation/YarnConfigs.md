# Yarn Configurations Explanation

## General HA Configuration

### Enable ResourceManager HA

#### Necessity of using RecourceManager HA
If it crashes:

- Existing containers usually continue running.
- NodeManagers are still alive.
- But no new containers can be allocated.
- No new applications can be submitted.
- Scheduling stops.

#### How to enable
```
<property>
  <name>yarn.resourcemanager.ha.enabled</name>
  <value>true</value>
</property>
```
#### Why can't both be active

Hadoop guarantees exactly one RM is active at any time because both ResourceManagers might allocate the same CPU, allocate the same memory, schedule conflicting containers. That would corrupt the cluster.

With HA:

```
        Active RM
       /
Clients
       \
        Standby RM
```

#### ZooKeeper's Role
ZooKeeper acts as the coordinator. Both RMs maintain a session with ZooKeeper, it keeps track of:

- who currently owns leadership
- whether the Active RM is still alive

ZooKeeper does not perform scheduling. It only coordinates which RM is Active.
Look at [Zookeep Section](#zookeeper) for further clarification.

#### Standby Resource Manager

In reality, the Standby RM is a **fully running ResourceManager** with one crucial restriction: **it is not allowed to make scheduling decisions.**

1. The Standby does not disconnect from ZooKeeper. It keeps its session alive so it can react immediately if leadership becomes available.

2. The Standby ResourceManager registers a watch on that znode. A watch is a notification mechanism that tells ZooKeeper, **"Notify me whenever this znode changes or is deleted."**

3. It recovers application state. The Active RM knows:
    - application IDs
    - queues
    - priorities
    - running containers
    - application attempts
    - completion status

    If RM1 crashes without persisting this information, there is no way to recover it. To prevent this, Hadoop uses a ***ResourceManager State Store***[^1] Whenever an application is submitted, accepted, or completed, or when containers are allocated, the Active ResourceManager persists enough metadata for another ResourceManager to recover its state. The Standby ResourceManager continuously reads this persisted state (or loads it when it starts). As a result, when the Standby becomes the Active ResourceManager, it can resume operation without starting from scratch.
4. Standby ResourceManager does not schedule resources only the Active RM is allowed to:
    - allocate containers
    - start containers
    - preempt containers
    The Standby deliberately keeps the scheduler passive.

#### Running Applications Behaviour

Containers are usually unaffected because they run on the NodeManagers, which continue operating even if the ResourceManager fails.

If the active ResourceManager fails during job submission, the client library reads the [yarn.resourcemanager.ha.rm-ids](#rm-ids) configuration to discover the configured ResourceManagers. It then automatically **retries the request against another ResourceManager**. As a result, the application usually succeeds without requiring any action from the user.

### Cluster ID

This is simply the ***logical name*** of the RM cluster. Every RM participating in HA must have exactly the same cluster ID.

```
<property>
  <name>yarn.resourcemanager.cluster-id</name>
  <value>production</value>
</property>
```

### RM IDs

This setting informs Hadoop that the cluster has two ResourceManagers.

```
<property>
  <name>yarn.resourcemanager.ha.rm-ids</name>
  <value>rm1,rm2</value>
</property>
```
These are IDs—not necessarily hostnames (although many deployments use the hostname as the ID).

### ZooKeeper
This is probably the most important HA property. The ResourceManagers use ZooKeeper to coordinate failover. ZooKeeper ensures only **one RM becomes Active at a time**.

```
<property>
  <name>yarn.resourcemanager.zk-address</name>
  <value>zk1:2181,zk2:2181,zk3:2181</value>
</property>
```

#### Leader Election
Before describing the leader election process, let's first become familiar with the concept of a ***znode***:

A znode (ZooKeeper node) is the basic data structure in ZooKeeper. Think of it like a file in a filesystem, except it can also store a small amount of data.
For example, ZooKeeper might contain:


```console
├── hadoop
│   ├── rmstore
│   └── leader
├── kafka
│   ├── brokers
│   └── controller
└── zookeeper

```

Here:

- /hadoop is a znode.
- /hadoop/leader is another znode.
- /kafka/brokers is another znode.

Each znode can have **children** (like directories), **store a small amount of data** (typically up to about 1 MB), have **permissions and metadata**. There are two types of znode:

- **Persistent znode:** Even if the client disconnects, It remains until someone explicitly deletes it.
- **Ephemeral znode:** An ephemeral znode exists only as long as the client's ZooKeeper session is alive.

```
RM1 connects
    ↓
create /yarn/leader   (ephemeral)
```

  If RM1 crashes, ZooKeeper detects the session has expired and automatically removes the znode:

```
/yarn/leader
    ↓
deleted automatically
```
That's why ephemeral znodes are ideal for representing "I'm alive."

With that background, let's return to the **leader election** process.
At first it's worth noting that there is no real election process involved. It's more accurately described as leader acquisition through distributed locking.

The sequence is:

1. RM1 connects to ZooKeeper.
2. RM2 connects to ZooKeeper.
3. Both attempt to acquire leadership.
4. ZooKeeper guarantees that only one succeeds atomically.

```
RM1 ---- create /yarn/leader ----> ZooKeeper

RM2 ---- create /yarn/leader ----> ZooKeeper
```

ZooKeeper processes these requests one at a time. Suppose RM1's request arrives first:
```
RM1: SUCCESS
RM2: NodeAlreadyExists
```

Result:

```
RM1 → Active
RM2 → Standby
```

No voting takes place.

#### Failover Detection

1. As mentioned earlier, an ephemeral znode is automatically deleted when the client session ends. Each ResourceManager (RM) maintains a heartbeat with ZooKeeper. Now, suppose RM1 crashes and its heartbeats stop. After the ZooKeeper session timeout expires, ZooKeeper determines that the leader is no longer available and automatically deletes its ephemeral znode.

2. ZooKeeper now tells the remaining RM that leadership is available.

### Automatic Failover
This setting informs Hadoop to switch automatically if the Active RM dies.

```
<property>
  <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
  <value>true</value>
</property>
```

Without it:

```
RM1 dies
    ↓
Administrator runs
    ↓
yarn rmadmin -transitionToActive rm2
```

With automatic failover:

```
RM1 dies
    ↓
ZooKeeper detects failure
    ↓
RM2 becomes Active
```

### ResourceManager Schedulare

This property determines which scheduling algorithm is used to allocate containers. The available scheduling algorithms were discussed extensively in Chapter 3.

The corresponding property is configured as follows:

```
<property>
  <name>yarn.resourcemanager.scheduler.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

## Per-ResourceManager Properties

One ResourceManager JVM starts several servers, each listening on a different port and serving a different purpose.

|         ResourceManager JVM          |
|:------------------------------------:|
|  RPC Server (8032)                   |
|  Scheduler RPC Server (8030)         |
|  ResourceTracker RPC Server (8031)   |
|  Admin RPC Server (8033)             |
|  Embedded HTTP Server (8088)         |

### Hadoop RPC(Remote Procedure Call)

Make calling a method on a remote machine look somewhat like calling a local method

```
                Machine A                         Machine B

                Client                            CalculatorImpl
                  |                                      |
                  |                                      |
                  +---------- network ------------------>+
```

The ResourceManager has an interface, which defines operations such as:

```Java
interface ApplicationClientProtocol {

    SubmitApplicationResponse submitApplication(
        SubmitApplicationRequest request
    );

    GetApplicationReportResponse getApplicationReport(
        GetApplicationReportRequest request
    );

    KillApplicationResponse forceKillApplication(
        KillApplicationRequest request
    );
}
```

The client might conceptually do:
```Java
client.submitApplication(request);
```

As the ResourceManager is on another machine, the RPC framework handles the networking part.

```
              Client machine                 RM machine

              +----------------+             +----------------+
              | YARN client    |             | ResourceManager|
              |                |             |                |
              | submitApp()    |             |                |
              +-------+--------+             +--------+-------+
                      |                               |
                      |        Hadoop RPC             |
                      +------------------------------>|
                                                      |
                                                submitApplication()
```
Hadoop gives the client a **proxy implementing** ‍‍‍```ApplicationClientProtocol```. The proxy is simply an object in the client's JVM. It implements the same interface: ```ApplicationClientProtocol```.

The client does:
```
client.submitApplication(request);
```

The proxy intercepts that call. It creates an RPC request containing information conceptually like as follows which gets serialized into byte and send to ResourceManager.

```
method:
    submitApplication

request:
    applicationId = ...
    applicationName = ...
    queue = ...
    ...
```

The ResourceManager has an RPC server listening there.

```
Client JVM                         ResourceManager JVM

            client.submitApplication() # Interface
                   |
                   v
               RPC Proxy # Implements the Interface
                   |
                   | serialized request
                   |
                   +---------------------------->
                                                :8032
                                                  |
                                              RPC Server
                                                  |
                                                  v
                                     ApplicationClientProtocol
                                          implementation
                                                  |
                                                  v
                                          ResourceManager
```

### RPC Addresses

**Applications** (yarn jar, spark-submit, MapReduce client) and **NodeManagers** use it to communicate with the ResourceManager.

When you run Spark Application, the Spark client eventually sends RPCs here.

```Bash
spark-submit ...
```

**Typical operations:**

- submit application
- kill application
- get application report
- get cluster metrics

Internally, this endpoint implements the **ApplicationClientProtocol** (?) interface.

The corresponding property is configured as follows:

```
<property>
  <name>yarn.resourcemanager.address.rm1</name>
  <value>rm1:8032</value>
</property>
```

### Scheduler Address

This endpoint is used by the scheduler. Once an application starts, its **ApplicationMaster** uses this endpoint to negotiate resources.

Remember the lifecycle:

```
  Client

      ↓

  ResourceManager

      ↓

  Container allocated

      ↓

  ApplicationMaster starts
```

For example if ApplicationMaster needs 10 more containers, Those requests go to the Scheduler RPC server. Internally this endpoint implements **ApplicationMasterProtocol**.

Keeping RPC and Scheduler separate simplifies authorization, scalability, and the protocol design.

The corresponding property is configured as follows:

```
<property>
  <name>yarn.resourcemanager.scheduler.address.rm1</name>
  <value>rm1:8030</value>
</property>
```

### Resource Tracker Address

This endpoint is used by the NodeManagers. Each NodeManager periodically sends heartbeats to this address, providing the ResourceManager with information about node health, available resources, and container status. Without these heartbeats, the ResourceManager would not know which nodes are alive, how much memory is available, or which containers have completed. A heartbeat contains information such as:

- available memory
- available CPUs
- container status
- node health

Internally this endpoint implements **ResourceTracker**.

The corresponding property is configured as follows:

```
<property>
  <name>yarn.resourcemanager.resource-tracker.address.rm1</name>
  <value>rm1:8031</value>
</property>
```

### Admin Address

Administrative commands connect here and uses this endpoint. For expamle:

```console
yarn rmadmin \
    -refreshNodes

yarn rmadmin \
    -transitionToActive rm1
```
Internally it implements **ResourceManagerAdministrationProtocol**.

The corresponding property is configured as follows:

```
<property>
  <name>yarn.resourcemanager.admin.address.rm1</name>
  <value>rm1:8033</value>
</property>
```

### Web UI(HTTP,HTTPS)

This is different. It is an embedded HTTP server instead of RPC.

Example:
```
http://rm1:8088
```
It shows:

- running applications
- queues
- nodes
- scheduler
- metrics

The corresponding propertied are configured as follows:

```
<property>
  <name>yarn.resourcemanager.webapp.address.rm1</name>
  <value>rm1:8088</value>
</property>
```
Same thing, except **HTTPS**.
```
<property>
  <name>yarn.resourcemanager.webapp.https.address.rm1</name>
  <value>rm1:8090</value>
</property>
```
## Other Yarn Configurations

There are actually four different groups here:

- YARN SharedCache
- NodeManager configuration
- Log aggregation
- Web UI

The idea is to avoid repeatedly transferring the same application files to nodes.

### YARN SharedCache

This enables the YARN SharedCache. The idea is to avoid repeatedly transferring the same application files to nodes.

Imagine 100 jobs all use ```my-big-library.jar```

Without SharedCache:
```
Job 1 → upload JAR → NodeManager
Job 2 → upload JAR → NodeManager
Job 3 → upload JAR → NodeManager
...
```
With SharedCache:



                                        SharedCache
                                            |
                                     my-big-library.jar
                                            |
                                  +---------+---------+
                                  |         |         |
                                 NM1       NM2       NM3


#### Overview

The YARN Shared Cache provides the facility to upload and manage shared application resources **to HDFS** in a safe and scalable manner.

YARN applications can leverage resources uploaded by other applications or previous runs of the same application **without having to re­upload and localize identical files multiple times.** This will save network resources and reduce YARN application startup time.

You may think what is the nesseccessity of the HDFS? Resources could be store in and transfered through ApplicationMaster or even ResourceManger directly to the  NodeManagers. I asked this in the chat bot in this link.

#### Architecture

The shared cache feature consists of 4 major components:

  1.  The shared cache client.
  2.  The HDFS directory that acts as a cache.
  3. The shared cache manager (aka. SCM).
  4. The localization service and uploader.

##### The Shared Cache Client

YARN application developers and users, should interact with the shared cache using the shared cache client. This client is responsible for **interacting with the shared cache manager**, **computing the checksum of application resources**, and **claiming application resources in the shared cache**.

###### Calculate a checksum for the resource
  A resource is identified by its checksum, rather than by its original filename or path.
  Hadoop's SharedCacheClient provides getFileChecksum() specifically for this purpose.

###### Interacting with the shared cache manager
  The client sends the checksum, together with the application's ApplicationId[^2], to the SharedCacheManager.

  Conceptually:

  ```
  Application
       |
       | SharedCacheClient.use(appId, checksum)
       v
  SharedCacheManager
       |
       | Is this resource in the Shared Cache?
       |
       +---- No ----> null
       |
       +---- Yes ---> URL/path of cached resource
  ```
###### Claim the resource for the application
The word "claim" is important.

Claiming a resource does not mean that the client downloads the file or physically copies it to the application container.

Instead, the application tells the SharedCacheManager:

    "My application is going to use the resource identified by this checksum."

The SharedCacheManager records that the application is currently using the resource. Once the client receives the cached resource URL, that resource is guaranteed to remain usable for the lifetime of that application.

The actual resource is still stored in the Shared Cache's HDFS directory. Later, YARN's normal NodeManager localization mechanism makes that resource available to the container that needs it.

Example

Suppose an application has:

    /home/user/lib/C.jar

The process is approximately:
```
1. Application has C.jar
          |
          v
2. SharedCacheClient calculates checksum
          |
          v
   checksum = abc123...
          |
          v
3. Client calls use(appId, "abc123...")
          |
          v
4. SharedCacheManager checks its metadata
          |
          +---- resource exists
          |
          v
5. SCM returns:
   hdfs:///sharedcache/.../C.jar
          |
          v
6. Application uses that URL as a YARN LocalResource
          |
          v
7. NodeManager localizes C.jar
          |
          v
8. Container can use C.jar
```

The important distinction is:

- SharedCacheClient  = identifies and claims the resource

- SharedCacheManager = manages the cache metadata and coordinates access

- HDFS Shared Cache = stores the actual resource

- NodeManager localization = makes the resource available on the node

###### Related YARN Configuration

```XML
  <property>
    <name>yarn.sharedcache.client-server.address</name>
    <value>{{ hadoop_yarn_resourcemanagers | first }}:8045</value>
  </property>
```


##### The Shared Cache HDFS Directory

**The shared cache HDFS directory stores all of the shared cache resources**. It is protected by HDFS permissions and is globally readable, but writing is restricted to a trusted user. This HDFS directory **is only modified by the shared cache manager and the resource uploader** on the node manager. Resources are spread across a set of subdirectories using the resources’s checksum:

```
/sharedcache/a/8/9/a896857d078/foo.jar
/sharedcache/5/0/f/50f11b09f87/bar.jar
/sharedcache/a/6/7/a678cb1aa8f/job.jar
```

You may wonder why HDFS is necessary in this case. The resources could be stored in the ApplicationMaster or even the ResourceManager and transferred directly to the NodeManagers. I raised this question in the [chatbot link](https://https://chatgpt.com/s/t_6ab5463de9388191b669b5761b19e8fd).

##### Shared Cache Manager (SCM)
The shared cache manager is responsible for **serving requests from the client and managing the contents of the shared cache**. It looks after both the meta data as well as the persisted resources in HDFS. It is made up of two major components, a **back end store and a cleaner service**. The SCM runs as a separate daemon process that **can be placed on any node in the cluster**. This allows for administrators to start/stop/upgrade the SCM without affecting other YARN components (i.e. the resource manager or node managers).

The **back end store** is responsible for **maintaining and persisting metadata about the shared cache**. This includes **the resources in the cache**, **when a resource was last used** and **a list of applications that are currently using the resource**. The implementation for the backing store is **pluggable** and **it currently uses an in-memory store that recreates its state after a restart**.

The cleaner service maintains the persisted resources in HDFS by ensuring that resources that are no longer used are removed from the cache. It scans the resources in the cache periodically and **evicts resources if they are both stale and there are no live applications currently using the application.**

YARN application developers and users, should interact with the shared cache using the shared cache client. This client is responsible for **interacting with the shared cache manager**, **computing the checksum of application resources**, and **claiming application resources in the shared cache**.

```
                         YARN cluster

     +------------------+       +----------------------+
     | ResourceManager  |       | SharedCacheManager   |
     |                  |       |                      |
     | scheduling       |       | metadata             |
     | applications     |       | client requests      |
     +------------------+       | cleanup              |
                                +----------+-----------+
                                           |
                                           |
                                    HDFS SharedCache
                                           |
                              +------------+------------+
                              |            |            |
                             DN1          DN2          DN3
```
Notice the SCM isn't transporting the Resource bytes, **The actual bytes come from HDFS**.

###### SCM backend store

The backend store is not the HDFS cache itself. Instead, it stores metadata about those resources, as mentioned above.

| checksum  | resource | last-used | applications   | HDFS location |
| --------- | -------- | --------- | -------------- | ------------- |
| abc123... | C.jar    | 10:32     | App_01, App_17 |/sharedcache/a/8/9/a896857.../C.jar |
| def456... | D.jar    | 09:15     | App_08         |/sharedcache/5/0/f/50f11b.../D.jar |

The active metadata database is held in RAM rather than being continuously persisted to a conventional external database. Since the actual resources remain in HDFS, if the SCM crashes or restarts, Hadoop can rebuild the metadata store by examining the existing SharedCache resources in HDFS. [The Hadoop API documentation for InMemorySCMStore explicitly describes this bootstrap behavior.](https://hadoop.apache.org/docs/r3.1.0/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-sharedcachemanager/apidocs/org/apache/hadoop/yarn/server/sharedcachemanager/store/InMemorySCMStore.html?utm_source=chatgpt.com)

###### Cleaner Service

The cleaner periodically examines cached resources therefore prevents the HDFS SharedCache from growing forever.

```
                            Resource
                                |
                      +---------+---------+
                      |                   |
                   stale?             currently used?
                      |                   |
                      |                   |
                     YES                  NO
                      |                   |
                      +---------+---------+
                                |
                                v
                             EVICT
                                |
                                v
                      delete from HDFS
```
SCM can have an **AppChecker** component that determines whether an application is still running. Hadoop provides a ```RemoteAppChecker``` implementation that queries the ResourceManager remotely
[The Hadoop API documents AppChecker specifically as the mechanism the cleaner uses to determine whether an application is running.
](https://hadoop.apache.org/docs/r3.1.0/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-sharedcachemanager/apidocs/org/apache/hadoop/yarn/server/sharedcachemanager/package-summary.html)

##### The Shared Cache uploader and localization
The shared cache uploader **is a service that runs on the node manager** and **adds resources to the shared cache**. It is responsible for **verifying a resources checksum**, **uploading the resource to HDFS** and **notifying the shared cache manager that a resource has been added to the cache**. It is important to note that the uploader service is asynchronous from the container launch and does not block the startup of a yarn application. In addition adding things to the cache is done in a best effort way and does not impact running applications. Once the uploader has placed a resource in the shared cache, **YARN uses the normal node manager localization mechanism to make resources available to the application**.

```
                         +----------------------+
                         | SharedCacheManager   |
                         |                      |
                         |  Client protocol     |
                         |  Admin protocol      |
                         |                      |
                         | +------------------+ |
                         | | Backend Store    | |
                         | | metadata         | |
                         | +------------------+ |
                         |          |           |
                         |          v           |
                         | +------------------+ |
                         | | Cleaner Service  | |
                         | +------------------+ |
                         +----------+-----------+
                                    |
                         metadata / coordination
                                    |
                                    v
                         +----------------------+
                         |   HDFS SharedCache   |
                         |                      |
                         |  C.jar               |
                         |  D.jar               |
                         |  X.zip               |
                         +----------+-----------+
                                    |
                              localization
                                    |
               +--------------------+--------------------+
               |                    |                    |
              NM1                  NM2                  NM3
               |                    |                    |
           local copy            local copy            local copy
               |                    |                    |
           Container            Container            Container
```

The YARN Shared Cache provides the facility to upload and manage shared application resources **to HDFS** in a safe and scalable manner.

YARN applications can leverage resources uploaded by other applications or previous runs of the same application **without having to re­upload and localize identical files multiple times.** This will save network resources and reduce YARN application startup time.




[^1]:The ResourceManager State Store is not a separate service. It's an abstraction (interface) that Hadoop uses to save and recover the ResourceManager's state. We need to set "yarn.resourcemanager.recovery.enabled" property value true to enable State Store.Common choices include:
    - ZooKeeper-based state store
    - Filesystem state store
    - LevelDB state store

[^2]:A unique identifier that YARN assigns to each submitted application.
