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

    If RM1 crashes without persisting this information, there is no way to recover it. To prevent this, Hadoop uses a ***ResourceManager State Store***.[^1] Whenever an application is submitted, accepted, or completed, or when containers are allocated, the Active ResourceManager persists enough metadata for another ResourceManager to recover its state. The Standby ResourceManager continuously reads this persisted state (or loads it when it starts). As a result, when the Standby becomes the Active ResourceManager, it can resume operation without starting from scratch.
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

## Per-ResourceManager Properties
### RPC Addresses

**Applications** and **NodeManagers** use it to communicate with the ResourceManager.
```
<property>
  <name>yarn.resourcemanager.address.rm1</name>
  <value>rm1:8032</value>
</property>
```
Completing ...

### Scheduler Address

This endpoint is used by the scheduler. Applications submit requests here.

```
<property>
  <name>yarn.resourcemanager.scheduler.address.rm1</name>
  <value>rm1:8030</value>
</property>
```

Completing ...

### Resource Tracker Address
NodeManagers connect here. Every heartbeat from every node goes to this address.

```
<property>
  <name>yarn.resourcemanager.resource-tracker.address.rm1</name>
  <value>rm1:8031</value>
</property>
```
Without this endpoint, the RM cannot know:

- available memory
- available CPUs
- container status
- node health

Completing ...

### Admin Address

```
<property>
  <name>yarn.resourcemanager.admin.address.rm1</name>
  <value>rm1:8033</value>
</property>
```
Administrative commands connect here. For expamle:

```console
yarn rmadmin
```
uses this endpoint.

Completing ...

### Web UI(HTTP,HTTPS)
This is the YARN web UI.
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
[^1]: The ResourceManager State Store is not a separate service. It's an abstraction (interface) that Hadoop uses to save and recover the ResourceManager's state. We need to set "yarn.resourcemanager.recovery.enabled" property value true to enable State Store.Common choices include:
    - ZooKeeper-based state store
    - Filesystem state store
    - LevelDB state store
