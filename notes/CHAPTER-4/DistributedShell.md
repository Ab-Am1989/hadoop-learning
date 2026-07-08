# Distributed Shell
Distributed Shell is not really a framework, it is mostly a sample YARN application.

It Runs shell commands across containers.

Example:

```
run "hostname"
on 50 machines
```

**Example:**

```Bash
run "hostname"
on 50 machines
```
Flow
```
Client
 ↓
Submit to YARN
 ↓
ApplicationMaster starts
 ↓
Request 50 containers
 ↓
Execute shell commands

```
Then Containers execute:

```
hostname
```

Internally Distributed Shell demonstrates:

```
submit application
request containers
launch commands
receive status
stop application
```

It is educational.

People read its source to learn, how do I build my own YARN application?

Think:

```
Distributed Shell ≈ "Hello World"
for YARN
```
