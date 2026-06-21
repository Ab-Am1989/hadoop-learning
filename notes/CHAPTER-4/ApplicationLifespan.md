## Application Lifespan

A YARN application is the **runtime environment** YARN creates to execute jobs.
A YARN application is basically:
```
ApplicationMaster + Containers + resources
```
The question is:
>When should YARN create a new application?

### Prerequsits
#### Job Concept in MapReduce

When a user runs:

```console
hadoop jar wordcount.jar input output
```
That is ONE user job and YARN creates **ONE application**. Inside that application, MapReduce executes:

```text
User Job (WordCount for example)
   ↓
Map phase
   ↓
Shuffle / Sort
   ↓
Reduce phase
```

**Notice:** Map and Reduce are tasks, not jobs.

***Example:***
Input:
```
a b c
a b
a
```
Map phase produces:
```
(a,1)
(b,1)
(c,1)

(a,1)
(b,1)

(a,1)
```
Shuffle groups:
```
a → [1,1,1]
b → [1,1]
c → [1]
```
Reduce phase outputs:
```
a → 3
b → 2
c → 1
```
All of them are ***ONE JOB*** and Inside that there may be many map and reduce tasks.

```
Job
├── Map Task 1
├── Map Task 2
├── Map Task 3
├── Map Task 4
│
├── Shuffle
│
├── Reduce Task 1
└── Reduce Task 2
```
Sometimes people chain jobs.

```
Job1 → aggregate logs
↓
Job2 → calculate statistics
↓
Job3 → generate report
```
That is called a workflow and contains multiple Jobs.

#### Job Concept in Spark
Suppose we want count HTTP 500 errors per service:

```
df = spark.read.parquet("/logs")

df.filter(df.status == 500)\
  .groupBy("service")\
  .count()\
  .show()
```
1 TB of logs will be split into 4 × 250 GB volumes in our example.

**Stage**: A set of parallel tasks that can run without exchanging data.

```python
df.filter(df.status == 500)
```

Each partition can filter independently, no machine needs another machine. then:

```
Stage 1

Task1 → Filter Partition1
Task2 → Filter Partition2
Task3 → Filter Partition3
Task4 → Filter Partition4
```
```
Imagine the Result is:
Machine 1:
Srv a → 10
Srv b → 25
Srv c → 15
Srv d → 20

Machine 2:
Srv b → 10
Srv d → 30

Machine 3:
Srv a → 5
Srv c → 10
Srv d → 20
Srv f → 5

Machine 4:
Srv b → 5
Srv c → 10
Srv d → 15
```

Then:
```python
groupBy("service")
```
data must move between machines which calls **Shuffle** and **create new Stage**.

```
Stage 2 (Group + Count)
Srv a → [10,5] → 15
Srv b → [25,10,5] → 40
Srv c → [15,10,10] → 35
Srv d → [20,30,20,15] → 85
Srv f → [5] → 5
```
Therefore, the job can be represented by the following schema:
```
Job
├── Stage1
│   ├── Task
│   ├── Task
│   └── Task
│
└── Stage2
    ├── Task
    └── Task
```
### Application Lifespan Models:
#### Model 1: One application per user job (MapReduce style)

YARN performs the following steps for each job:
```
Create Application #1
 ↓
Start ApplicationMaster
 ↓
Launch containers
 ↓
Run job
 ↓
Destroy everything
```
```
Job1: [Start====Finish]

Job2:           [Start====Finish]

Job3:                      [Start====Finish]
```
Advantage:

- Simple isolation.

Disadvantage

- Every job pays startup cost:
   - allocate AM
   - start containers
   - initialize framework

#### Model 2: One application per session/workflow (Spark style)
You start:

```
spark-shell
```

YARN:
```
Create one application
 ↓
Start Driver
 ↓
Allocate executors
```
But YARN does NOT create new applications for each job.
```
Spark Application
   │
   ├── Job 1
   ├── Job 2
   ├── Job 3
   └── Job 4
```
```
Application:
[=========================]

Job1:
[====]

Job2:
      [===]

Job3:
           [======]
```
This is faster because containers remain alive in addition Spark can cache Data in execter memory.
```
Job1:

Read 1TB → compute

Job2:

Reuse cached result
```
**No reread from HDFS.**

#### Model 3: Long-running shared application (service style)

Instead of:

```
user → application
```

it becomes:
```
many users → one application
```
Application stays alive continuously.

```
Timeline:

Application:
[===================================]

User1 Query:
   [--]

User2 Query:
        [---]

User3 Query:
             [--]
```
Example: Impala wants SQL queries to return quickly.

````
Impala daemon
      ↓
Long-running AM
      ↓
Request resources immediately
````
#### Why not always use long-running applications?
| Model               | Example   | Startup Cost | Resource Usage |
| ------------------- | --------- | -----------: | -------------: |
| One app per job     | MapReduce |         High |            Low |
| One app per session | Spark     |       Medium |         Medium |
| Long-running shared | Impala    |     Very low |           High |
