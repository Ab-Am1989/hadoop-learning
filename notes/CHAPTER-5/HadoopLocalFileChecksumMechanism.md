## Hadoop's local-file checksum mechanism

This part is explaining how Hadoop protects files stored on your local filesystem (such as files on your Linux machine), rather than files stored in HDFS.

The main idea is that Hadoop's LocalFileSystem calculates and verifies checksums for local files, just as HDFS does for data stored in a cluster. However, the implementation is different.

### What is LocalFileSystem?

Hadoop provides an abstraction called FileSystem. It allows applications to work with files without needing to know all the details of the underlying storage system.

| Filesystem | Where data is stored | Checksum mechanism |
|---|---|---|
| `LocalFileSystem` | Local filesystem, such as Linux's ext4 | Hadoop calculates and verifies checksums |
| `RawLocalFileSystem` | Local filesystem | Hadoop's additional checksum mechanism is disabled |
| `DistributedFileSystem` | HDFS cluster | HDFS calculates and verifies checksums |

The **URI scheme** (file:// or hdfs://) determines which filesystem implementation Hadoop uses, according to its configuration.

local file:
```
file:///home/user/data.txt
```

HDFS file:
```
hdfs://namenode:8020/data/data.txt
```

### client-side checksumming

***client-side*** means that Hadoop's filesystem implementation running in the application performs the checksum calculations and verification.

When you use Hadoop's LocalFileSystem to write the file, it creates two files:

```
/home/user/
    |
    +-- data.txt
    |
    +-- .data.txt.crc # auxiliary metadata
```

Notice that this is different from HDFS, where checksum metadata is maintained alongside the block data on DataNodes.

**Notice:** Checksum verification applies when the file is accessed through the checksum-aware LocalFileSystem. If a program accesses the file directly through ordinary operating-system file operations, Hadoop's checksum verification is not automatically performed.

### Disable checksum calculation

Your Linux machine might use ext4 or XFS to store files. These filesystems provide their own storage management and integrity features, although ordinary ext4 and XFS configurations do not provide end-to-end checksumming of file contents in the same way that Hadoop's checksum layer does.

Some other filesystems, such as ZFS and Btrfs, provide native data checksumming.

If the underlying storage system already provides the required integrity protection, an application might choose to avoid Hadoop's additional checksum layer although **Disabling Hadoop's checksums does not automatically mean that another system will detect or correct corruption.**

| Feature | `LocalFileSystem` | `RawLocalFileSystem` |
|---|---|---|
| Reads and writes local files | Yes | Yes |
| Creates Hadoop `.crc` files | Yes | No |
| Verifies Hadoop checksums when reading | Yes | No |
| Provides Hadoop's additional checksum layer | Yes | No |

#### fs.file.impl configuration

Hadoop uses filesystem implementation settings to determine which implementation handles a particular URI scheme.

Here, file refers to local filesystem URIs:

```
file:///home/user/data.txt
```

By setting ```fs.file.impl``` to ```RawLocalFileSystem```, you tell Hadoop to use that implementation for local filesystem URIs in the relevant configuration context.

This means Hadoop will use the raw local filesystem implementation rather than its checksum-aware LocalFileSystem implementation.

> To do this globally in an application...

It means that the configuration affects local filesystem access throughout the application that uses that configuration.

It does not mean that checksums are disabled for every Hadoop application across the entire cluster.

Other applications using different configurations may continue to use LocalFileSystem.
