# **Chapter: HDFS Internals — Blocks, Replication, Safemode & Cluster Health**

---

## **Introduction**
This chapter provides a clear and hands-on exploration of key internal components of the Hadoop Distributed File System (HDFS). Students will execute commands to observe system behavior.

You will learn:
- How HDFS stores data as blocks
- How replication ensures fault tolerance
- How to inspect cluster health
- What Safe Mode is and why it matters
- How to validate file integrity with `fsck`
- How quotas protect the filesystem

Every concept is paired with a demonstration.

---

## **1. Verifying HDFS Setup**
Before interacting with HDFS, ensure that your Hadoop daemons are running correctly.

### **1.1 Check Running Daemons**
```bash
jps
```
**Explanation:**  
Lists Java processes. You should see NameNode, DataNode, ResourceManager, NodeManager, and JobHistoryServer.

### **1.2 Check Cluster Health**
```bash
hdfs dfsadmin -report
```
**Explanation:**  
Displays total storage, remaining storage, number of DataNodes, and the health of each node.

### **1.3 Create Working Directory in HDFS**
```bash
hdfs dfs -mkdir -p /lab/p2
```
**Explanation:**  
Creates a folder in HDFS for lab work. The `-p` flag ensures all parent directories are created.

---

## **2. Understanding Blocks & Replication**
HDFS stores files as fixed-size blocks (e.g., 128 MB). Replication ensures that each block is stored on multiple nodes.

### **2.1 View Block Size**
```bash
hdfs getconf -confKey dfs.blocksize
```
**Explanation:**  
Displays the block size configured for HDFS.

### **2.2 Create a File That Spans Multiple Blocks**
```bash
dd if=/dev/zero of=~/bigfile.bin bs=50M count=7 status=progress
```
**Explanation:**  
Creates a 350 MB file locally to ensure it spans multiple blocks in HDFS.

Upload to HDFS:
```bash
hdfs dfs -put -f ~/bigfile.bin /lab/p2/
```

Inspect block distribution:
```bash
hdfs fsck /lab/p2/bigfile.bin -files -blocks -locations
```
**Explanation:**  
Shows block IDs, sizes, and which DataNodes each block resides on.

### **2.3 Adjust Replication Factor**
```bash
hdfs dfs -setrep -w 3 /lab/p2/bigfile.bin
```
**Explanation:**  
Sets replication factor to 3 and waits for replication.

Verify replication:
```bash
hdfs dfs -stat %r /lab/p2/bigfile.bin
```

---

## **3. Cluster Health & Safemode**
Safe Mode is a read-only mode during which the NameNode awaits block reports from DataNodes.

### **3.1 Check Safemode Thresholds**
```bash
hdfs getconf -confKey dfs.namenode.safemode.threshold-pct
hdfs getconf -confKey dfs.namenode.replication.min
```
**Explanation:**  
These values determine when the NameNode will exit Safe Mode.

### **3.2 Demonstrate Safe Mode Behavior**
Check status:
```bash
hdfs dfsadmin -safemode get
```
Enter Safe Mode:
```bash
hdfs dfsadmin -safemode enter
```
Try writing a file (should fail):
```bash
hdfs dfs -touchz /lab/p2/test_safemode.txt
```
Exit Safe Mode:
```bash
hdfs dfsadmin -safemode leave
```

---

## **4. File System Integrity — fsck Deep Dive**
The `fsck` tool verifies file health, block metadata, and replication.

### **4.1 Inspect Block Health**
```bash
hdfs fsck /lab/p2/bigfile.bin -files -blocks -locations -racks
```
**Explanation:**  
Adds rack information to show how replicas are distributed.

### **4.2 Inspect Entire Namespace (limited)**
```bash
hdfs fsck / -blocks -locations | head -n 60
```
**Explanation:**  
Shows block distribution for the entire HDFS. `head` limits output.

---

## **5. Quotas: Namespace & Space Constraints**
Quotas protect the filesystem against accidental overuse.

### **5.1 Namespace Quotas**
```bash
hdfs dfs -mkdir -p /lab/p2/project
hdfs dfs -setQuota 10 /lab/p2/project
hdfs dfs -count -q /lab/p2/project
```
**Explanation:**  
Limits a directory to a maximum number of files and folders.

### **5.2 Space Quotas**
```bash
hdfs dfs -setSpaceQuota 5m /lab/p2/project
dd if=/dev/urandom of=~/tiny.bin bs=1M count=6 status=none
hdfs dfs -put -f ~/tiny.bin /lab/p2/project/
```
**Explanation:**  
Limits the maximum space a folder can occupy.

Clear quotas:
```bash
hdfs dfs -clrQuota /lab/p2/project
hdfs dfs -clrSpaceQuota /lab/p2/project
```

---

## **6. Cleanup Tasks**
Clean up HDFS and local workspace.

```bash
hdfs dfs -rm -r -skipTrash /lab/p2
rm -f ~/bigfile.bin ~/tiny.bin
```
**Explanation:**  
Removes lab directories and temporary files.

---

## **7. Appendix: Command Reference (With Explanations)**
### **HDFS File Commands**
- `hdfs dfs -mkdir -p` — Create directories in HDFS.
- `hdfs dfs -ls` — List directory contents.
- `hdfs dfs -put` — Upload file into HDFS.
- `hdfs dfs -cat` — Print HDFS file contents.
- `hdfs dfs -text` — Read compressed logs.
- `hdfs dfs -du -h` — Show disk usage.
- `hdfs dfs -mv` — Move files in HDFS.
- `hdfs dfs -cp` — Copy files in HDFS.
- `hdfs dfs -get` — Download from HDFS.
- `hdfs dfs -getmerge` — Merge reducer output to a single local file.
- `hdfs dfs -rm -r` — Delete recursively.

### **Administration Commands**
- `hdfs dfsadmin -report` — View cluster health.
- `hdfs dfsadmin -safemode` — Control Safe Mode.
- `hdfs fsck` — Check file/block integrity.
- `hdfs dfs -setrep` — Set replication factor.
- `hdfs dfs -setQuota` — Set namespace quota.
- `hdfs dfs -setSpaceQuota` — Set space quota.
- `hdfs dfs -count -q` — View quota usage.

---

© PrimrIQ AI Services LLP

