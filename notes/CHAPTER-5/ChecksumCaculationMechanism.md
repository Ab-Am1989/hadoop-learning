### Automatic protection

> separate checksum is created for every dfs.bytes-per-checksum bytes of data.

HDFS doesn't create just one checksum for the entire file, it divides the data into smaller chunks for checksum calculation. It calculates a separate checksum for each chunk **rather than calculating only one checksum for the entire file**.

The size of each chunk is controlled by the following configuration property which its default value is 512 bytes.

```XML
dfs.bytes-per-checksum
```

For example, imagine you write a 1,536-byte file to HDFS. HDFS divides the data into three 512-byte chunks:


```
File: 1,536 bytes
       |
       v
+------------------+
| Chunk 1: 512 B   | ---> Checksum 1 (4 bytes)
+------------------+
| Chunk 2: 512 B   | ---> Checksum 2 (4 bytes)
+------------------+
| Chunk 3: 512 B   | ---> Checksum 3 (4 bytes)
+------------------+
```

If corruption is detected, it can identify the affected checksum chunk rather than treating the entire file as one indivisible unit.

```
Read chunk
     |
     v
Calculate checksum
     |
     v
Compare with stored checksum
     |
     +---- Match ------> Data passes the check
     |
     +---- Mismatch ---> Corruption detected
```

The checksum detects corruption; it doesn't repair the data itself. If another valid replica is available, HDFS can use it to recover the correct data.


### Checksum Algorthim

HDFS uses CRC-32C, a checksum algorithm that produces a 32-bit checksum.

So, for every 512 bytes of data, HDFS needs an additional 4 bytes for the checksum.

\[
512 + 4 = 516 \text{ bytes}
\]

\[
\text{Overhead} = \frac{4}{512} \times 100
\]

### Blocks VS Chunks

Note that these 512-byte chunks are checksum units, not HDFS blocks. An HDFS block is typically much larger, such as 128 MB.

```
File: 256 MB
       |
       v
+---------------------------+
| HDFS Block 1 (128 MB)     |
|                           |
|  Chunk 1: 512 bytes       | ---> Checksum 1 (4 bytes)
|  Chunk 2: 512 bytes       | ---> Checksum 2 (4 bytes)
|  Chunk 3: 512 bytes       | ---> Checksum 3 (4 bytes)
|  ...                      |
|  Many more chunks         |
+---------------------------+
| HDFS Block 2 (128 MB)     |
|                           |
|  Chunk 1: 512 bytes       | ---> Checksum 1 (4 bytes)
|  Chunk 2: 512 bytes       | ---> Checksum 2 (4 bytes)
|  Chunk 3: 512 bytes       | ---> Checksum 3 (4 bytes)
|  ...                      |
|  Many more chunks         |
+---------------------------+
```

The checksum metadata file corresponds to the block's data file. The checksums are not simply appended to the end of the block's data.

```
DataNode
   |
   +-- Block data file
   |      |
   |      +-- Data chunk 1 (512 bytes)
   |      +-- Data chunk 2 (512 bytes)
   |      +-- Data chunk 3 (512 bytes)
   |
   +-- Block checksum metadata file
          |
          +-- Checksum 1 (4 bytes)
          +-- Checksum 2 (4 bytes)
          +-- Checksum 3 (4 bytes)
```
