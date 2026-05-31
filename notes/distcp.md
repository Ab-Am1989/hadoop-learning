# Parallel Copying with distcp
## Comparing two ways to copy a file inside HDFS
Suppose you run:

```console
hadoop fs -cp /data/bigfile /backup/bigfile
```

The client machine running the command becomes part of the data path.

```markdown
Source DataNode
      ↓
    Client
      ↓
Destination DataNodes
```

The client:

1. Reads the file from HDFS.
2. Receives all bytes over the network.
3. Writes those bytes back into HDFS.

It is not considered suitable for **large files** because the client becomes bottleneck and consumes bandwidth, consumes CPU and takes longer.

Instead of it distcp (Distributed Copy) launches a **MapReduce-style** job that copies data inside the cluster.

```console
DataNode A
    ↓
Mapper running near data
    ↓
DataNode B
```
**The copy is distributed across cluster nodes. The client mostly just submits the job and it does not sit in the middle of the data transfer.**

Then Even if you're copying **one huge file**, DistCp may still be better.

## DistCp Is a MapReduce Job
When you run:

```console
hadoop distcp hdfs:///src hdfs:///dst
```
- Hadoop doesn't start a single process that copies all files. Instead, it launches a MapReduce job.

- copying files doesn't require:
    - aggregation
    - grouping
    - sorting

Each file can be copied independently. So DistCp uses **Maps only** and **No reducers are needed**.
