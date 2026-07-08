# Apache Slider
**“I already have a distributed application — run it on YARN”**

You want an HBase cluster.

**Without Slider:**

```
SSH node1
start HBase

SSH node2
start regionserver

monitor manually
```

**With Slider:**


slider create hbase-cluster

```
Slider:

Requests containers
Starts HBase nodes
Tracks failures
Restarts failed nodes
```

You can later say:

> scale 3 → 10 nodes

Slider adjusts YARN containers.

Think **Slider ≈ Kubernetes Deployment** for YARN

You focus on how many nodes, Slider handles Where to run.
