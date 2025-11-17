# Understanding Hadoop: A Beginner's Guide to Big Data Storage and Processing

Hadoop is a framework for storing and processing large amounts of data across multiple computers. It's designed to handle data that's too big to fit on a single machine.

## The Two Main Components of Hadoop

Hadoop has two core parts that work together:

**HDFS (Hadoop Distributed File System)** - This is the storage layer. It manages how and where data is stored across multiple machines.

**YARN (Yet Another Resource Negotiator)** - This is the resource management layer. It decides which computers run which jobs and allocates computing resources.

On top of YARN, you can run different processing frameworks like MapReduce, Spark, or Tez.

---

## HDFS: How Hadoop Stores Data

### The Main Components

**NameNode**

The NameNode is the master server that manages the file system. It keeps track of:
- Which files exist in the system
- How files are divided into blocks
- Where each block is stored
- File permissions and directory structure

The NameNode stores two important files:
- **fsimage** - A complete snapshot of the file system at a point in time
- **edit log** - A record of all changes made since the last snapshot

The NameNode doesn't store the actual data. It only stores information about the data.

**DataNode**

DataNodes are worker servers that store the actual data. Each DataNode:
- Stores blocks of data on its local hard drives
- Creates, deletes, and replicates blocks when instructed by the NameNode
- Sends regular heartbeat messages to the NameNode to confirm it's working
- Sends block reports listing all the blocks it currently stores

### How Data is Divided and Stored

HDFS splits large files into smaller pieces called blocks. The default block size is usually 128MB or 256MB.

Each block is treated as an independent file on the DataNode's disk. This means a single large file is broken into multiple blocks that can be stored on different machines.

**Replication for Safety**

Every block is copied and stored on multiple DataNodes. The default is 3 copies. This provides:
- Fault tolerance - if one machine fails, the data still exists elsewhere
- Better read performance - clients can read from the nearest copy

**Default Placement Strategy:**
1. First copy goes on the node where the client is writing (if the client is in the cluster)
2. Second copy goes on a DataNode in a different rack
3. Third copy goes on another DataNode in the same rack as the second copy

This strategy protects against both individual machine failures and entire rack failures.

### Reading and Writing Data

**Reading a file:**
1. The client asks the NameNode for the locations of all blocks in the file
2. The NameNode returns a list of DataNodes holding each block
3. The client reads directly from the DataNodes, usually choosing the closest one

**Writing a file:**
1. The client asks the NameNode to create a new file
2. The NameNode creates a record and assigns block locations
3. The client writes data to the first DataNode in the pipeline
4. That DataNode forwards the data to the second DataNode
5. The second DataNode forwards to the third
6. Each DataNode confirms successful storage back to the NameNode

### Handling Failures

**If a DataNode fails:**
- The NameNode detects the failure through missing heartbeats
- It marks all blocks on that DataNode as under-replicated
- It instructs other DataNodes to create new copies to restore the replication count

**If the NameNode fails:**
- In older setups without High Availability, the entire cluster becomes unavailable
- Modern setups use High Availability with an active NameNode and a standby NameNode
- The standby takes over if the active NameNode fails

### Useful Commands

Here are some basic commands for working with HDFS:

- Upload a file: `hdfs dfs -put localfile /user/hadoop/`
- List files: `hdfs dfs -ls /user/hadoop/`
- Check file system: `hdfs fsck / -files -blocks -locations`
- See where blocks are stored: `hdfs fsck /path -files -blocks -locations`

---

## YARN: How Hadoop Manages Computing Resources

YARN is the system that manages computing resources and schedules jobs across the cluster.

### The Main Components

**ResourceManager (RM)**

This is the master daemon that manages resources for the entire cluster. It:
- Tracks available CPU and memory on each machine
- Allocates resources to applications
- Monitors the health of NodeManagers

The ResourceManager has two main parts:
- **Scheduler** - Decides which applications get resources and when
- **ApplicationsManager** - Accepts job submissions and starts ApplicationMasters

**NodeManager (NM)**

One NodeManager runs on each worker machine. It:
- Manages containers on that specific machine
- Monitors resource usage (CPU, memory, disk)
- Reports status back to the ResourceManager through heartbeats
- Launches and stops containers as instructed

**ApplicationMaster (AM)**

Each application gets its own ApplicationMaster. The ApplicationMaster:
- Negotiates with the ResourceManager for resources
- Requests specific numbers of containers with specific resource requirements
- Manages the execution of all tasks in the application
- Monitors task progress and handles failures
- Releases resources when the application completes

**Containers**

A container is a collection of resources (CPU cores and memory) allocated on a single NodeManager. 

Each container:
- Has a specific resource allocation (for example, 2 CPU cores and 4GB memory)
- Runs a single process or task
- Is monitored by the NodeManager
- Is released when its task completes

### Scheduling Options

YARN offers different schedulers for different needs:

- **FIFO Scheduler** - Processes jobs in the order they arrive
- **Capacity Scheduler** - Guarantees each organization or team a minimum share of cluster resources
- **Fair Scheduler** - Distributes resources equally across all running jobs

### How an Application Runs

Here's what happens when you submit a job:

1. The client submits the application to the ResourceManager
2. The ResourceManager allocates a container and starts the ApplicationMaster
3. The ApplicationMaster determines what resources it needs
4. The ApplicationMaster requests containers from the ResourceManager
5. The ResourceManager grants containers on available NodeManagers
6. The ApplicationMaster launches tasks in those containers
7. Tasks execute and write their results
8. The ApplicationMaster monitors progress and handles any failures
9. When all tasks complete, the ApplicationMaster releases resources
10. The ApplicationMaster reports completion to the ResourceManager

---

## Data Locality: Why Hadoop Moves Computation to Data

Traditional systems move data to where the computation happens. Hadoop does the opposite - it moves the computation to where the data already exists.

### Why This Matters

Reading large amounts of data over the network is slow and expensive. It's much faster to run the processing on the same machine that already has the data stored locally.

### Locality Levels

YARN tries to schedule tasks according to these preferences:

1. **Node-local** - The task runs on the same machine that stores the data block. This is fastest because there's no network transfer.

2. **Rack-local** - The task runs on a different machine in the same rack. This requires network transfer but only within the rack.

3. **Off-rack** - The task runs on a machine in a different rack. This requires network transfer across racks and is the slowest option.

### Benefits

- Reduces network traffic significantly
- Improves overall cluster performance
- Allows the cluster to process more data simultaneously
- Reduces bottlenecks on network switches

### How It Works in Practice

When a MapReduce job runs, the scheduler tries to place map tasks on machines that have the input data blocks stored locally. If that's not possible, it tries rack-local placement. Only as a last resort does it place tasks on remote racks.

---

## Input Splits and Map Tasks

Input splits determine how many map tasks will run for a job.

### The Basic Rule

One input split creates one map task. The number of input splits determines the number of mappers.

### What is an Input Split?

An input split is a logical division of the input data. It's not the same as an HDFS block, though they often align.

The InputFormat class determines how to create splits. Different InputFormats handle this differently based on the file type.

### Default Behavior for Text Files

For regular text files, the system typically:
- Creates splits that align with HDFS block boundaries
- Ensures splits don't break records (lines) in half
- Extends a split boundary if necessary to include a complete line

### Example Calculation

Let's say you have:
- A single file that's 300MB
- HDFS block size is 128MB

HDFS will store this as:
- Block 1: bytes 0-127MB
- Block 2: bytes 128-255MB
- Block 3: bytes 256-300MB (44MB)

The InputFormat will typically create 3 input splits, resulting in 3 map tasks.

### Split Size Configuration

You can control split size with these parameters:
- `mapreduce.input.fileinputformat.split.maxsize` - Maximum split size
- `mapreduce.input.fileinputformat.split.minsize` - Minimum split size

The actual split size is calculated as:
`max(minSize, min(blockSize, maxSize))`

### The Small Files Problem

If you have many small files, each file might become its own split. This creates too many mappers.

For example:
- 10,000 files of 1KB each
- Default behavior: 10,000 mappers
- Problem: High overhead from starting and managing so many tasks

**Solutions:**
- Use CombineFileInputFormat to combine multiple small files into one split
- Pack files into a single container format like SequenceFile, ORC, or Parquet
- Process and merge small files before loading into HDFS

### Compressed Files

Compression affects how files can be split:

**Splittable compression formats** (like bzip2):
- Files can be divided into multiple splits
- Multiple mappers can process the file in parallel
- Good performance for large files

**Non-splittable compression formats** (like gzip):
- The entire file becomes one split
- Only one mapper can process the file
- Poor parallelization for large files

---

## MapReduce Job Execution Flow

Here's what happens during a complete MapReduce job:

### Step 1: Job Submission
You submit your job code (packaged as a JAR file) to the ResourceManager along with:
- Input file paths
- Output directory
- Configuration parameters

### Step 2: ApplicationMaster Starts
The ResourceManager allocates a container and launches the ApplicationMaster for your job.

### Step 3: Split Calculation
The ApplicationMaster:
- Examines all input files
- Calculates input splits based on the InputFormat
- Determines the number of map tasks needed

### Step 4: Resource Request
The ApplicationMaster requests containers from the ResourceManager. It prefers containers on machines that have the input data stored locally.

### Step 5: Map Phase
Map tasks begin executing:
- Each mapper reads its assigned input split
- Processes each record in the split
- Emits intermediate key-value pairs
- Writes output to local disk (not HDFS)
- Partitions output by which reducer should receive it

### Step 6: Shuffle Phase
Reducers retrieve their data:
- Each reducer pulls its portion of data from all mappers
- Data is transferred over the network
- This is called the shuffle phase

### Step 7: Sort and Merge
Each reducer:
- Sorts all the data it received by key
- Merges data from multiple mappers

### Step 8: Reduce Phase
Reducers execute:
- Run the reduce function on each key and its values
- Write final output to HDFS
- Output is written with replication for durability

### Step 9: Cleanup
- The ApplicationMaster releases all containers
- Reports job completion to the ResourceManager
- The client receives job completion status

---

## Performance and Tuning Tips

### HDFS Configuration

**Block Size**
- Larger blocks (256MB) reduce metadata overhead
- Fewer blocks mean less memory needed on the NameNode
- Good for large files and high-throughput workloads
- Smaller blocks give better parallelism for smaller files

**Replication Factor**
- Default is 3 copies
- Provides good balance of durability and storage cost
- Reduce to 2 if storage space is limited and you can accept slightly higher risk
- Increase above 3 for critical data or read-heavy workloads

**NameNode Memory**
- The NameNode stores all metadata in RAM
- More files and blocks require more memory
- Plan approximately 1GB RAM per million blocks
- Monitor memory usage and increase as cluster grows

### MapReduce Configuration

**Controlling Number of Mappers**
- Tune split size to control mapper count
- Too many mappers waste resources on task startup overhead
- Too few mappers limit parallelism
- Find balance based on your data and cluster size

**Setting Number of Reducers**
- Set explicitly with `mapreduce.job.reduces`
- Too many reducers waste resources
- Too few reducers create bottlenecks
- Start with approximately: number of nodes × 0.95

**Compression**
- Enable compression for map output: `mapreduce.map.output.compress=true`
- Reduces network traffic during shuffle
- Trades CPU time for I/O savings
- Usually beneficial for shuffle-heavy jobs

**Speculative Execution**
- Hadoop can launch duplicate tasks if one is running slowly
- Helps complete jobs faster when some tasks lag
- Can waste resources by running duplicate work
- Consider disabling if cluster resources are constrained

**Container Resources**
- Configure memory per container carefully
- Set `yarn.scheduler.minimum-allocation-mb` appropriately
- Avoid allocating too much or too little memory
- Match container size to your task requirements

---

## Common Problems and Solutions

### Problem 1: Too Many Small Files

**What happens:**
- Each file creates metadata on the NameNode
- NameNode runs out of memory
- Too many mappers create excessive overhead
- Job startup and completion take very long

**Solutions:**
- Use CombineFileInputFormat to group small files
- Merge files into larger files before processing
- Use columnar formats like Parquet or ORC
- Use SequenceFiles as containers for multiple small files

### Problem 2: NameNode Memory Issues

**What happens:**
- NameNode crashes with OutOfMemoryError
- Cluster becomes unavailable
- Cannot create new files

**Solutions:**
- Increase NameNode heap memory
- Reduce number of small files
- Enable High Availability with standby NameNode
- Consider HDFS Federation for very large clusters

### Problem 3: Poor Parallelism with Compressed Files

**What happens:**
- Large gzipped files cannot be split
- One mapper handles entire file
- Other mappers finish early and sit idle
- Job takes much longer than necessary

**Solutions:**
- Use splittable compression formats (bzip2)
- Compress only output files, not input files
- Split files before compressing
- Use compression codecs that support splitting

### Problem 4: Loss of Data Locality

**What happens:**
- Tasks run on machines far from the data
- Excessive network traffic during data reading
- Jobs run slower than expected

**Solutions:**
- Keep cluster load balanced
- Avoid oversubscribing resources
- Increase scheduler delay settings to wait for local resources
- Add more capacity if cluster is consistently overloaded

### Problem 5: Shuffle Bottleneck

**What happens:**
- Large amounts of intermediate data
- Network becomes saturated during shuffle
- Reduce tasks take very long to start

**Solutions:**
- Use combiners to reduce intermediate data
- Compress map output
- Optimize your map function to emit less data
- Consider whether you can avoid the reduce step entirely
- Increase network capacity if needed

---

## Summary

Hadoop provides a way to store and process large datasets across many machines. The key concepts are:

**Storage (HDFS):**
- Data is split into blocks
- Blocks are replicated for safety
- NameNode tracks metadata
- DataNodes store actual data

**Processing (YARN):**
- ResourceManager allocates resources
- ApplicationMaster manages each job
- NodeManagers run tasks in containers
- Tasks run where data is stored when possible

**MapReduce:**
- Input is divided into splits
- One mapper processes each split
- Map output is shuffled to reducers
- Reducers produce final output

