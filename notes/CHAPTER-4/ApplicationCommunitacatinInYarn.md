## Application Communicatation In YARN Anatomy

Notice from Figure 4-2 that YARN itself does not provide any way for the parts of the
application (client, master, process) to communicate with one another. Most nontrivial
YARN applications use some form of remote communication (such as Hadoop’s RPC
layer) to pass status updates and results back to the client, but these are specific to the
application.

The key point is:

> YARN allocates resources and launches processes, but YARN does NOT define how those processes talk to each other.
YARN is a resource manager, not a communication framework.

**YARN only does:**
- allocate container
- start process
- monitor process
- stop process

### Example: Spark

```console
spark-submit app.py
```

**Flow:**

<<<<<<< HEAD
> Client
>  ↓
> YARN
>  ↓
> Spark Driver (ApplicationMaster)
>  ↓
> Executors
=======
```
 Client
  ↓
 YARN
  ↓
 Spark Driver (ApplicationMaster)
  ↓
 Executors
```
>>>>>>> dd5cb14 (Yarn added and catogerized notes)

**Executors need to:**

- receive tasks
- send results
- send heartbeat
- exchange shuffle data

YARN only allocates machines and Spark provides:

<<<<<<< HEAD
> Driver ←→ Executor RPC
> Executor ←→ Executor shuffle
=======
```
 Driver ←→ Executor RPC
 Executor ←→ Executor shuffle
```
>>>>>>> dd5cb14 (Yarn added and catogerized notes)
