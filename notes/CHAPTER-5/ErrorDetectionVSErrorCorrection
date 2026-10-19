## Why error detection is not enough
> The usual way of detecting corrupted data is by computing a checksum for the data when it first enters the system, and again whenever it is transmitted across a channel that is unreliable and hence capable of corrupting the data.
> The data is deemed to be corrupt if the newly generated checksum doesn’t exactly match the original.
> This technique doesn’t offer any way to fix the data, it is merely error detection. (**And this is a reason for not using low-end hardware; in particular, be sure to use ECC memory.**)


1. What checksums can and cannot do.
2. How memory errors can corrupt data.
3. How ECC memory helps prevent or detect those errors.

A checksum can help detect the corruption if the data is checked against a previously calculated checksum. However, the checksum itself cannot reconstruct the original data.

Instead of relying only on software to detect corruption after it happens, we should also **use hardware that helps prevent memory errors from corrupting data in the first place**.

This workflow illustrates an important limitation of checksums: **checking data at one point in time does not guarantee that it will remain unchanged at every subsequent stage of processing.**

If the corrupted data is later written to another file or transmitted elsewhere, the corruption could propagate.
ECC memory helps reduce this risk by **detecting and correcting** certain memory errors.

```
                        Disk
                         |
                         v
                  Read correct data
                         |
                         v
                 Data enters RAM
                         |
                         v
                   Memory error
                         |
                         v
                 Corrupted data
                         |
                         v
               Hadoop processes it
```
### What is ECC memory?
ECC stands for Error-Correcting Code. ECC memory is RAM that includes additional error-detection and error-correction capabilities.

This is a memory error while ordinary RAM generally does not provide ECC's ability to detect and correct such errors.

```
Original: 10110110
Stored:   10100110
              ^
         Bit changed because of a hardware issue
```
If one bit changes because of a memory error, ECC can detect the inconsistency and, with common ECC implementations, correct the single-bit error automatically. Actual ECC implementations use mathematical codes rather than simply comparing the data with a saved copy.

Many server-class ECC implementations can correct single-bit errors and detect (but not necessarily correct) certain multi-bit errors. The precise capabilities depend on the memory system.

The important point is that ECC can correct certain memory errors before corrupted data propagates further through the system.
