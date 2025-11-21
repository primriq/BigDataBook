# HDFS Part-1 Practice Lab

## Contents
1. Prerequisites & Health Check
2. Dataset Preparation (Local)
3. Create HDFS Directory Structure
4. Ingest Data into HDFS
5. Explore & Inspect in HDFS
6. Organize: Move & Copy
7. Export & Merge
8. Cleanup Options
9. Deliverables & Rubric
10. Command Reference
11. Troubleshooting

## 1. Prerequisites & Quick Health Check

```bash
start-dfs.sh  #run this to start the hdfs file system
start-yarn.sh #run this to start the resource manager
stop-dfs.sh   #run this to stop hdfs service. use it after you are done working with hadoop
stop-yarn.sh  #run this to stop resource manager. use it after you are done working with hadoop
jps
hdfs dfsadmin -report | sed -n '1,80p'
```

## 2. Dataset Preparation (Local Files)

```bash
mkdir -p ~/hdfs_lab_p1/local/{logs,notes,data}
printf "line1
line2
line3
" > ~/hdfs_lab_p1/local/notes/readme.txt
seq 1 500 > ~/hdfs_lab_p1/local/data/numbers_1_500.txt
split -l 100 ~/hdfs_lab_p1/local/data/numbers_1_500.txt ~/hdfs_lab_p1/local/data/part_
echo "ERROR something broke"  > ~/hdfs_lab_p1/local/logs/app.log
echo "WARN low memory"       >> ~/hdfs_lab_p1/local/logs/app.log
gzip -k ~/hdfs_lab_p1/local/logs/app.log
```

## 3. Create HDFS Directory Structure

```bash
hdfs dfs -mkdir -p /lab/p1/{raw,staging,archive}
hdfs dfs -ls -R /lab/p1
```

## 4. Ingest Data into HDFS

```bash
hdfs dfs -put ~/hdfs_lab_p1/local/notes/readme.txt /lab/p1/raw/
hdfs dfs -put ~/hdfs_lab_p1/local/logs/app.log.gz /lab/p1/raw/
hdfs dfs -put -f ~/hdfs_lab_p1/local/data /lab/p1/staging/
hdfs dfs -ls -R /lab/p1
hdfs dfs -du -h /lab/p1
```

## 5. Explore & Inspect in HDFS

```bash
hdfs dfs -cat /lab/p1/raw/readme.txt
hdfs dfs -text /lab/p1/raw/app.log.gz | head
hdfs dfs -stat "%n | size=%b | perm=%a | owner=%u | modified=%y" /lab/p1/raw/readme.txt
hdfs dfs -count -h /lab/p1
```

## 6. Organize: Move & Copy

```bash
hdfs dfs -mkdir -p /lab/p1/archive/notes
hdfs dfs -mv /lab/p1/raw/readme.txt /lab/p1/archive/notes/
hdfs dfs -cp /lab/p1/raw/app.log.gz /lab/p1/archive/
hdfs dfs -ls -R /lab/p1/archive
```

## 7. Export & Merge

```bash
mkdir -p ~/hdfs_lab_p1/out
hdfs dfs -get /lab/p1/staging/data/part_aa ~/hdfs_lab_p1/out/
hdfs dfs -getmerge /lab/p1/staging/data ~/hdfs_lab_p1/out/merged_numbers.txt
wc -l  ~/hdfs_lab_p1/out/merged_numbers.txt
head -n 5 ~/hdfs_lab_p1/out/merged_numbers.txt
tail -n 5 ~/hdfs_lab_p1/out/merged_numbers.txt
```

## 8. Cleanup Options

```bash
hdfs dfs -rm -r -skipTrash /lab/p1
rm -rf ~/hdfs_lab_p1
```



## Command Reference
- start-dfs.sh
- start-yarn.sh
- jps
- hdfs dfsadmin -report
- hdfs dfs -mkdir -p
- hdfs dfs -ls
- hdfs dfs -put
- hdfs dfs -du -h
- hdfs dfs -cat
- hdfs dfs -text
- hdfs dfs -stat
- hdfs dfs -count -h
- hdfs dfs -mv
- hdfs dfs -cp
- hdfs dfs -get
- hdfs dfs -getmerge
- hdfs dfs -rm

## Troubleshooting
- PATH issues
- Permission errors
- File exists errors
- Large output handling
- Missing daemons
- Merge order issues

### 🔍 **HDFS COMMANDS EXPLAINED**


###  **1. `jps`**
Shows which Java-based Hadoop daemons are currently running.

Typical output includes:
- `NameNode`
- `DataNode`
- `ResourceManager`
- `NodeManager`
- `JobHistoryServer`

If any are missing, Hadoop may not work properly.

---

###  **2. `hdfs dfsadmin -report`**
Displays detailed HDFS cluster information including:
- Total capacity
- Used / remaining space
- Number of DataNodes
- Health status of each node

Great for validating HDFS health.

---

###  **3. `hdfs dfs -mkdir -p <path>`**
Creates directories inside HDFS.

`-p` ensures parent directories are created if they don’t exist.

Example:
```bash
hdfs dfs -mkdir -p /user/student/input
```

---

###  **4. `hdfs dfs -ls <path>`**
Lists files and directories inside HDFS.

Example:
```bash
hdfs dfs -ls /user/$(whoami)
```

---

###  **5. `hdfs dfs -put <local> <hdfs>`**
Uploads a file from the local filesystem to HDFS.

Example:
```bash
hdfs dfs -put sales.csv /user/rishu/data/
```

---

###  **6. `hdfs dfs -du -h <path>`**
Shows disk usage for files/directories in HDFS in human-readable format.

Great for checking dataset sizes.

---

###  **7. `hdfs dfs -cat <path>`**
Prints the contents of a file stored in HDFS to the terminal.

Example:
```bash
hdfs dfs -cat /user/rishu/output/part-00000
```

---

### **8. `hdfs dfs -text <path>`**
Displays file contents in text format.
Useful when files are compressed with codecs.

Example:
```bash
hdfs dfs -text /user/rishu/logs.gz
```

---

###  **9. `hdfs dfs -stat <format> <path>`**
Prints metadata about a file.
Common formats:
- `%n` → name
- `%b` → size in bytes
- `%y` → modification time

Example:
```bash
hdfs dfs -stat %b /user/rishu/data/sales.csv
```

---

### **10. `hdfs dfs -count -h <path>`**
Displays:
- number of directories
- number of files
- disk space consumed

`-h` makes it human-readable.

---

###  **11. `hdfs dfs -mv <source> <dest>`**
Moves files inside HDFS.

Example:
```bash
hdfs dfs -mv /user/rishu/temp/file.txt /user/rishu/final/
```

---

###  **12. `hdfs dfs -cp <source> <dest>`**
Copies files inside HDFS.

Example:
```bash
hdfs dfs -cp /data/input/file.txt /backup/file.txt
```

---

###  **13. `hdfs dfs -get <hdfs> <local>`**
Downloads files from HDFS to the local filesystem.

Example:
```bash
hdfs dfs -get /user/rishu/output/part-00000 ~/downloads/
```

---

###  **14. `hdfs dfs -getmerge <hdfs-dir> <local-file>`**
Merges multiple HDFS files (usually reducer outputs) into a single local file.

Example:
```bash
hdfs dfs -getmerge /user/rishu/output/ result.txt
```

---

###  **15. `hdfs dfs -rm <path>`**
Deletes a file from HDFS.
To delete folders:
```bash
hdfs dfs -rm -r <dir>
```
Add `-f` to avoid errors when the file doesn’t exist.

Example:
```bash
hdfs dfs -rm -r -f /user/rishu/old_output
```

---

© Copyright **PrimrIQ AI Services LLP**
 **PrimrIQ AI Services LLP**




