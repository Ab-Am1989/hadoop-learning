# Anatomy of a File Write
## Datanode Failure During File Writing

If any datanode fails while data is being written to it, then the following actions are taken:

At first remember client maintains two queues:

| Queue      | Meaning                               |
| ---------- | ------------------------------------- |
| data queue | packets waiting to send               |
| ack queue  | packets sent but not yet acknowledged |

#### Scenario

Suppose client sends packets:

```
P1 P2 P3 P4
```

Then **data flow** would be:

```text
Client → DN1 → DN2 → DN3
```
Consequently **acknowledgment flow** would become:

```
DN3 → DN2 → DN1 → Client
```

Pipeline becomes broken:

```
Client → DN1 → X → DN3
```
Then **DN3** consider as downstream node

#### Step 1: Close Pipline

Old pipeline discarded using broken connection immediately.

#### Step 2: Requeue Unacknowledged Packets

packets in **ack queue** are added to front of **data queue**

**Suppose State Was**

| Packet | State              |
| ------ | ------------------ |
| P1     | acknowledged       |
| P2     | acknowledged       |
| P3     | sent but not ACKed |
| P4     | sent but not ACKed |
| P5     | not sent yet       |

**Queues**

Ack queue:

```
P3 P4
```

Data queue:

```
P5
```
Requeue happens because after DN2 crash maybe DN3 never never received P3/P4. We cannot trust partially completed writes.

Then new data queue becomes:

```markdown
P3 P4 P5
```

#### Step 3: Partial Block Gets New Identity

**current block on good datanodes is given new identity tells NameNode Old incomplete replica version is obsolete"**

Because replicas are now inconsistent.

| Node | Data             |
| ---- | ---------------- |
| DN1  | P1 P2 P3 P4      |
| DN2  | crashed halfway  |
| DN3  | maybe only P1 P2 |

Suppose failed DN2 later recovers. It may contain stale, incomplete, corrupted partial block.

The new block identity tells NameNode:

**Delete old stale replica later.**

#### Step 4: Remove Failed Node

Before

```markdown
Client → DN1 → DN2 → DN3
```
After

```markdown
Client → DN1 → DN3
```
#### Step 5: Rebuild Pipeline

Remaining healthy nodes reconnect and Pipeline repaired.

#### Step 6: Continue Writing

Client resends packets through repaired pipeline:

```
P3 P4 P5 ...
```
Application usually notices nothing.

**But Replication Factor Is Now Wrong**

**Important Insight**
The client does NOT directly create replacement replica.

Client only:

keeps write alive
repairs pipeline

NameNode later restores desired replication asynchronously.
