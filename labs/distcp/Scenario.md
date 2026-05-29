## Copy Local File into HDFS
```console
hadoop fs -put dir1 dir2
```
if dir2 already exist:

```console
hadoop fs -put -f dir1 dir2
```

## Distributed Copy in HDFS

```console
hadoop distcp dir2 dir3
```

## Distributed Copy and Update Changed Files
```console
hadoop distcp -update dir2 dir3
```
