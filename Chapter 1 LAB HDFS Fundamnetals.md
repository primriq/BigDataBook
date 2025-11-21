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

## Deliverables & Rubric

### Deliverables
- Recursive listing of `/lab/p1`
- Metadata of `readme.txt`
- Archive directory listing
- Line count of merged file (500 expected)
- Explanation of -put, -copyFromLocal, -cp

### Rubric
- Setup and ingestion: 3
- Exploration & metadata: 2
- Move/copy: 2
- Export & merge: 2
- Short answer: 1

## Command Reference
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
