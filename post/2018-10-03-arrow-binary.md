# Investigating performance of Apache Arrow in-memory 

This will be a multipart blog series where I am looking closely at the 
performance of Apache Arrow. For the context, please refer to the 
discussion on Apache mailing lists [here](https://lists.apache.org/thread.html/943d41c49245d39cd932316f70404712658c30b55a09bc11fa9a2b58@%3Cdev.crail.apache.org%3E), 
and [here](https://lists.apache.org/thread.html/b299033a02b6d8f6f5b8dee1a9fcce221323ce90e9695bf0978b6ba0@%3Cdev.crail.apache.org%3E). 
The key theme of this investigation is what does it take for Apache 
Arrow to deliver 100+ Gbps performance on modern networking and 
storage hardware. Naturally, the performance depends upon the type of schema and the 
runtime environment (here: Java/JVM) as well.  


The benchmark code is available 
at [https://github.com/animeshtrivedi/benchmarking-arrow](https://github.com/animeshtrivedi/benchmarking-arrow). 
Its README.md might not be up-to date yet, so please have a look at the source code when in doubt.

 ## Index 
 1. [Introduction](#introduction)
 2. [Setting up the benchmark](#setting-up-the-benchmark)
 3. [Benchmarking Caution](#benchmarking-caution)
     - 3.1 [NUMA](#numa)
     - 3.2 [JVM GC settings](#jvm-gc-settings)     
 4. [Establishing a single thread performance](#establishing-a-single-thread-performance)
     - 4.1 [The default out of the box performance](#the-default-out-of-the-box-performance)
     - 4.2 [Using the Holder API](#using-the-holder-api)
     - 4.3 [Optimizing the batch size](#optimizing-the-batch-size)
 5. [Core scalability of the performance](#core-scalability-of-the-performance)
 6. [Conclusions](#conclusions)
 7. [Next Steps](#next-steps)
 
## Introduction 

For this investigation, we make our life simple by starting with a simple schema with one column 
of `byte[]`, whose size we will control. We will look at multi-column schema and different column 
types later. We further simplify the setup by holding all data in-memory, hence without any 
 dependency on external network or storage devices. 
 
 Our benchmarking sequence (for a single thread) is : 
   * Generate data in the Arrow format (certain number of rows, size of the column)  [see: `ArrowDataGenerator` class]
   * Write the data out to an in-memory (on or off-heap) location [see: `MemoryChannel` class] 
   * Read and materialize `byte[]` elements from the stored in-memory Arrow data as fast as possible [see `ArrowReader` class]
   
And we will be benchmarking the last step to measure the performance.  

### Setting up the benchmark 

To run the benchmark, follow these instructions: 
```commandline
$git clone https://github.com/animeshtrivedi/benchmarking-arrow
$cd benchmarking-arrow
$ mvn package # build your benchmarking jar 
$ mvn dependency:copy-dependencies # copy all the dependencies 
```
After that to run the program you can 
```commandline
$java -cp ./target/benchmark-arrow-1.0.jar:./target/dependency/* com.github.animeshtrivedi.anoc.Main -h 
```
It will show you all the options that we are going to use later. 

## Benchmarking Caution
I have found these setting after a long trial and error setup. So I am just going to summarize 
them here quickly. 
### NUMA 
In our experience we have found that Linux kernel has a bit unpredictable NUMA behavior when 
dealing with thread migrations and memory allocation. I found that it makes significant 
(up-to 30-40%) difference in the performance when ensuring that there are no unnecessary 
thread migrations and bad NUMA memory allocations. For this reason, I use `numactl` tool to 
pin the benchmark thread on core 0 with memory allocation from NUMA node 0 as well. You can 
do this by  
```commandline
numactl -C 0 -m 0 [cmd] 
```

Furthermore, it is worth noticing the NUMA options for the JVM: 
 ```commandline
java -XX:+UseNUMA
```
[https://docs.oracle.com/javase/8/docs/technotes/guides/vm/performance-enhancements-7.html#numa](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/performance-enhancements-7.html#numa)

Linux specific NUMA settings: https://lwn.net/Articles/568870/ (and check `cat /proc/sys/kernel/numa_balancing`). 
I recommend to switch off the numa balancing code by echoing zero into it. 

You can measure the cross node memory traffic by looking at `node-load-misses` and `node-store-misses`
counters in the `perf`. See their explanation here: [https://www.spinics.net/lists/linux-perf-users/msg03351.html](https://www.spinics.net/lists/linux-perf-users/msg03351.html) 

### JVM GC settings 
As we are going to be running benchmarks with threads, and lots of memory, in my adhoc 
evaluation I have found to use the next generation of G1C garbage collector 
([https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html))
and large young memory region (`-Xmn`) useful. I restricted my setting with 120G to make sure 
it stays within a single NUMA node. For 16 core experiments, I used 240GB (young). YMMV. 

```-XX:+UseG1GC -Xmn120G -Xmx256G```

## Establishing a single thread performance 

We run these experiments on a dual-socket Sandy Bridge Xeon system (8 x 2 = 16 cores) with 
256GB of memory, split between 2 NUMA nodes with each having 8 cores and 128GB of memory. 

### The default out of the box performance 
So without any comprehensive parameter sweeping, we start with the baseline performance for 
a 1KiB (`-s 1024`) binary array (`-n binary`), with 50M rows (`-r 50000000`), and 100K rows as
the batch size (`-g 100000`). We choose the binary format for now to eliminate any 
type specific overheads in the JVM. I think all parameters are self explanatory except 
perhaps the batch size. Arrow read and write files in the granularity of "some" batch size. 
It is identified as number of rows. So, 100K batch size x 1KB of row size = 100 MB of the 
batch size. This size is chosen as an educated guess, because my previous benchmark that 
converted parquet files to arrow files for benchmarking used this size from the parquet files. 
The parquet file use this size to match the underlying block size of the storage system such 
as HDFS (the default block size in HDFS is 128MB). We _will_ revise this number later.
 
I am just going to show the parameter that I passed to the command line as shown in the previous 
section.
 
```commandline
$[...] -t ArrowMemBench -p 1 -r 50000000 -c 1 -g 100000 -n binary -s 1024
...  
-----------------------------------------------------------------------
Total bytes: 51200000000(47GiB) bandwidth 9.72 Gbps
-----------------------------------------------------------------------
$
```
So we got 9.72 Gbps/core. That is a good start. 

### Using the Holder API 
Now, if we see how we are consuming the data in our benchmark: 
```java
protected void consumeBinary(VarBinaryVector vector) {
  int length;
  int valCount = vector.getValueCount();
  for(int i = 0; i < valCount; i++){
    if(!vector.isNull(i)){
      binaryCount+=1;
      length = vector.get(i).length;
      this.checksum+=length;
      this.binarySizeCount+=length;
    }
  }
}
```
This is getting a `byte[]` object after each call and then just discarding it. In a real-world 
setting perhaps one would do some more useful work with the byte array. However, at this point 
we are just interested in the length of the array. Anything more (like calculating the checksum)
would be too intensive and will dilute the benchmarking focus. 

However, we can optimize this by using the `Holder` object API in the arrow code. With the `holder` 
object, the result is left in place and you can materialize it when you want, but you eliminate 
one copy perhaps. A binary holder object looks like: 
```java
public final class NullableVarBinaryHolder implements ValueHolder {
    public static final int WIDTH = 4;
    public int isSet;
    public int start;
    public int end;
    public ArrowBuf buffer;
    // more ... 
}
```
So instead of giving us the `byte[]`, the holder object is going to keep the data in `ArrowBuf` and 
let us know the `start` and `end` from where we can read the data. So, lets use it (see the `-e` flag): 

```commandline
$[...] -t ArrowMemBench -p 1 -r 50000000 -c 1 -g 100000 -n binary -s 1024 -e 
...                                                                       ^^ (notice here!)
-----------------------------------------------------------------------
Total bytes: 51200000000(47GiB) bandwidth 16.50 Gbps
-----------------------------------------------------------------------
$
```  
16.50 Gbps, a major improvement! 
 
***Note:*** A few word about the holder API - it is benefetial to use the holder API specially for 
binary data. However, for basic primitive types such as integers, or floats - as far as I can 
tell - the API is only going to improve the number of object one has to allocate. However, it will 
not help to optimize if you have to materialize the object at all or not. I am yet to verify this 
experimentally. 

### Optimizing the batch size 

So far we have looked at the batch size of 100k rows, which with a column size of 1024 bytes, 
gives us about 100MB of buffer size that Arrow has to allocate to read the batches. We can further
think of optimizing it and making it close to the cache size (~10-20MB L3 size on modern Xeon 
processors). 

```commandline
$[...] -t ArrowMemBench -p 1 -r 50000000 -c 1 -g 10000 -n binary -s 1024 -e 
...                                               ^^^^ (notice here!)     
-----------------------------------------------------------------------
Total bytes: 51200000000(47GiB) bandwidth 28.82 Gbps
-----------------------------------------------------------------------
$
```  
28.82 Gbps! Now if we quickly look at the profile of two runs with 100MB and 10MB batch sizes, 
we see the difference in the cache profile. 

These numbers are obtained by `perf` doing a system-wide profile when running the reading 
benchmark, 3 runs, profiled for 3 seconds each. 
**100MB run**
```concept
 Performance counter stats for 'system wide' (3 runs):

             2,199      context-switches                                              ( +-  3.69% )
                10      cpu-migrations                                                ( +- 11.63% )
         1,444,307      page-faults                                                   ( +-  3.55% )
    12,062,141,644      cycles                                                        ( +-  2.27% )  (24.84%)
     8,732,319,102      stalled-cycles-frontend   #   72.39% frontend cycles idle     ( +-  3.45% )  (33.23%)
     6,586,224,482      stalled-cycles-backend    #   54.60% backend cycles idle      ( +-  3.38% )  (41.61%)
     6,991,502,072      instructions              #    0.58  insn per cycle         
                                                  #    1.25  stalled cycles per insn  ( +-  5.58% )  (50.00%)
     1,109,409,046      branches                                                      ( +-  5.09% )  (58.38%)
        10,698,302      branch-misses             #    0.96% of all branches          ( +- 16.68% )  (66.77%)
       186,228,557      cache-references                                              ( +-  2.99% )  (75.15%)
       114,510,721      cache-misses              #   61.489 % of all cache refs      ( +-  3.86% )  (75.23%)
           137,219      node-load-misses                                              ( +-  4.85% )  (75.23%)
            76,909      node-store-misses                                             ( +-  6.09% )  (75.20%)
        84,642,739      node-loads                                                    ( +-  5.13% )  (16.51%)
        80,668,594      node-stores                                                   ( +-  4.04% )  (16.52%)

       3.004971832 seconds time elapsed                                          ( +-  0.01% )

```

**10MB run**  
```concept
 Performance counter stats for 'system wide' (3 runs):

             1,394      context-switches                                              ( +-  6.00% )
                11      cpu-migrations                                                ( +- 12.50% )
            61,704      page-faults                                                   ( +-  3.61% )
    12,045,929,891      cycles                                                        ( +-  0.28% )  (24.86%)
     8,974,257,416      stalled-cycles-frontend   #   74.50% frontend cycles idle     ( +-  1.75% )  (33.24%)
     6,243,038,136      stalled-cycles-backend    #   51.83% backend cycles idle      ( +-  2.07% )  (41.63%)
     7,980,935,088      instructions              #    0.66  insn per cycle         
                                                  #    1.12  stalled cycles per insn  ( +-  3.65% )  (50.02%)
     1,304,910,031      branches                                                      ( +-  4.64% )  (58.41%)
        17,511,876      branch-misses             #    1.34% of all branches          ( +- 18.26% )  (66.80%)
       374,485,482      cache-references                                              ( +-  3.20% )  (75.18%)
       115,439,365      cache-misses              #   30.826 % of all cache refs      ( +-  4.86% )  (75.22%)
           209,857      node-load-misses                                              ( +-  3.08% )  (75.22%)
           103,764      node-store-misses                                             ( +-  3.10% )  (75.18%)
       173,314,393      node-loads                                                    ( +-  4.39% )  (16.52%)
       165,636,949      node-stores                                                   ( +-  3.68% )  (16.52%)

       3.004484679 seconds time elapsed                                          ( +-  0.01% )

```

Almost 2x performance improvement in the cache hit rate. If we go further down to 1MB batch size, 
the performance suffers massively, down from 31.32 Gbps to 11.19 Gbps. And looking at the profile, 
we can second guess why: 
**1MB run**
```concept
 Performance counter stats for 'system wide' (3 runs):

             1,428      context-switches                                              ( +-  7.95% )
                 9      cpu-migrations                                                ( +- 65.11% )
               136      page-faults                                                   ( +- 23.05% )
    12,201,425,465      cycles                                                        ( +-  1.62% )  (24.82%)
     6,580,415,038      stalled-cycles-frontend   #   53.93% frontend cycles idle     ( +-  3.42% )  (33.20%)
     3,780,207,662      stalled-cycles-backend    #   30.98% backend cycles idle      ( +-  3.70% )  (41.58%)
    15,454,944,840      instructions              #    1.27  insn per cycle         
                                                  #    0.43  stalled cycles per insn  ( +-  0.90% )  (49.96%)
     2,939,884,628      branches                                                      ( +-  2.27% )  (58.34%)
        30,118,504      branch-misses             #    1.02% of all branches          ( +-  7.31% )  (66.72%)
       173,786,996      cache-references                                              ( +-  6.14% )  (75.11%)
        22,131,380      cache-misses              #   12.735 % of all cache refs      ( +-  6.73% )  (75.23%)
            96,552      node-load-misses                                              ( +-  6.70% )  (75.24%)
            56,009      node-store-misses                                             ( +- 14.01% )  (75.22%)
        56,870,596      node-loads                                                    ( +-  7.51% )  (16.51%)
         3,140,630      node-stores                                                   ( +-  3.68% )  (16.51%)

       3.005626026 seconds time elapsed                                          ( +-  0.03% )

```
The number of instructions increases significantly from 7.9B (10MB case) to 15.4B (1MB case). 
I am not yet entirely sure why this is happening (**Q1**).    

## Core scalability of the performance 

Having established the best case performance for 10MB case for a single thread (28.82 Gbps), we 
now look at the performance scalability with respect to the number of cores. So, if we just 
scale this performance for 1, 2, 4, 8, 16 cores in my system, this is what I get: 

**Table 1 - core scalability**

| Cores       |  Bandwidth | 
| :------------- |:-------------|
| 1	 | 28.82 Gbps |  
| 2  | 52.11 Gbps |
| 4  | 75.86 Gbps |
| 8 | 91.82 Gbps|
| 16 | 168.83 Gbps|

I can only run 1 core experiment with 50M rows each because other runs get to the limit of 
127GB/NUMA node configuration in my system. I don't want to generate any cross-node traffic. 
So, further runs are scaled proportionally to make sure that num_row/thread x cores is less 
than 128 GB. 

So, the performance looks good. With the 16 cores, we managed to drive around 170 Gbps, it is 
still not ideal scaling performance, but something I can live with. 

## Conclusions 

Here are some short take away messages: 
  * Use the holder API if possible.
  * Find the right batch size for your application 
  * Pay attention to the NUMA (core, memory, NIC) settings 
  * Pay attention to the GC profile and settings  

## Next steps 
  * **Q1** What is the correlation among the cache profile, block size, and the performance? As far I 
  can measure, the cache hit improves with small block sizes, but performance degrades. Why?
  
  * **Q2** What is the sensitivity to the column size. In the current analysis we just choose 1024 as 
    an arbitrary column size with a (intended) goal to see how far can we push the performance.
    
  * **Q3** How does performance reported for `byte[]` translate to reading real world data like 
  TPC-DS SQL tables?
   
  * **Q4** What is the performance with storage and networking devices? 
  
  * **Q5** Arrow uses netty's memory manager and buffer code. What is the right setting for those
  for large-scale experiments? 
  
And perhaps more, as I continue my investigation.