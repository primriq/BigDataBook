# **Chapter 5: Combiner in Action**

---

##  **Before You Start — System Checklist**

Before running the hands-on exercises in this chapter, students must verify that their system, environment, and directory setup match the requirements. Use this checklist:

###  **1. WSL / Linux environment setup**
- Using **Ubuntu on WSL** or any Linux distro
- Python 3 installed (`python3 --version`)
- Java installed (`java -version`)
- Hadoop installed and configured for **pseudo-distributed mode**

###  **2. Hadoop Services Running**
Run:
```bash
jps
```
You should see at least:
- NameNode
- DataNode
- ResourceManager
- NodeManager
- JobHistoryServer

If any are missing, restart Hadoop using:
```bash
stop-all.sh
start-all.sh
mr-jobhistory-daemon.sh start historyserver
```

###  **3. Set STREAMING_JAR Variable**
Verify:
```bash
echo $STREAMING_JAR
```
If empty, add this to your `~/.bashrc`:
```bash
export STREAMING_JAR=$(ls $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar)
```
Reload:
```bash
source ~/.bashrc
```

###  **4. Directory Structure Must Be Clear**
Students must confirm where their files live.
Recommended structure:
```
/home/<username>/mapper.py
/home/<username>/reducer.py
/home/<username>/mapreduce-combiner-csv-demo/sales.csv
```
If your paths differ, **adapt all commands accordingly**.

### Examples:
- If mapper is in `/home/alex/hadoop/scripts/mapper.py` → change all commands to match that path.
- If CSV is in `/data/sales/sales.csv` → update both local and HDFS commands.

###  **5. HDFS Working Properly**
Test HDFS:
```bash
hdfs dfs -ls /
```
If you get an error, re-check NameNode/DataNode services.

###  **6. Internet Not Required**
Everything is local; no external downloads required.

---

: Combiner in Action — A Practical Walkthrough**

This chapter demonstrates how Hadoop Streaming processes data **with and without a Combiner**, using a CSV dataset and Python-based mapper and reducer. The goal is to understand how Combiners reduce shuffle data, improve performance, and optimize MapReduce jobs.

---

## ##  **1. Project Structure**

Your files are organized like this:

```
/home/rishu/mapper.py
/home/rishu/reducer.py
/home/rishu/mapreduce-combiner-csv-demo/sales.csv
```

Mapper & reducer sit in the home directory. Only the CSV lives inside the demo folder.

---

## ##  **2. Copy CSV File Into Working Directory**

```bash
cp "/mnt/c/Users/rishu/OneDrive - PrimrIQ AI Services LLP/Desktop/Hadoop Demo Files/sales.csv" \
   ~/mapreduce-combiner-csv-demo/sales.csv
```

Verify:

```bash
ls ~/mapreduce-combiner-csv-demo
```

---

## ##  **3. Upload CSV to HDFS**

```bash
hdfs dfs -mkdir -p /user/$(whoami)/csv_combiner_demo
hdfs dfs -put -f ~/mapreduce-combiner-csv-demo/sales.csv \
  /user/$(whoami)/csv_combiner_demo/sales.csv
```

Check:

```bash
hdfs dfs -ls /user/$(whoami)/csv_combiner_demo
```

---

## ##  **4. Test Mapper and Reducer Locally**

### Test Mapper
```bash
cat ~/mapreduce-combiner-csv-demo/sales.csv | /home/rishu/mapper.py | head
```

### Test Full Local Pipeline
```bash
cat ~/mapreduce-combiner-csv-demo/sales.csv \
  | /home/rishu/mapper.py \
  | sort -t $'\t' -k1,1 \
  | /home/rishu/reducer.py
```

This simulates Map → Shuffle → Reduce without Hadoop.

---

## ##  **5. Run Hadoop Job WITHOUT Combiner**

### Clean previous output
```bash
hdfs dfs -rm -r -f /user/$(whoami)/output_no_combiner
```

### Run job
```bash
hadoop jar "$STREAMING_JAR" \
  -files /home/rishu/mapper.py,/home/rishu/reducer.py \
  -mapper /home/rishu/mapper.py \
  -reducer /home/rishu/reducer.py \
  -input /user/$(whoami)/csv_combiner_demo/sales.csv \
  -output /user/$(whoami)/output_no_combiner
```

### View reducer output
```bash
hdfs dfs -cat /user/$(whoami)/output_no_combiner/part-* | sort
```

---

## ##  **6. Show Mapper Output (No Combiner)**

This shows exactly what Hadoop sends into the shuffle phase.

### Clean directory
```bash
hdfs dfs -rm -r -f /user/$(whoami)/output_mapper_only
```

### Run mapper-only job
```bash
hadoop jar "$STREAMING_JAR" \
  -files /home/rishu/mapper.py \
  -mapper /home/rishu/mapper.py \
  -reducer /bin/cat \
  -input /user/$(whoami)/csv_combiner_demo/sales.csv \
  -output /user/$(whoami)/output_mapper_only
```

### Check output
```bash
hdfs dfs -cat /user/$(whoami)/output_mapper_only/part-* | head
```

---

## ##  **7. Run Hadoop Job WITH Combiner**

### Clean old output
```bash
hdfs dfs -rm -r -f /user/$(whoami)/output_with_combiner
```

### Run job with combiner
```bash
hadoop jar "$STREAMING_JAR" \
  -files /home/rishu/mapper.py,/home/rishu/reducer.py \
  -mapper /home/rishu/mapper.py \
  -reducer /home/rishu/reducer.py \
  -combiner /home/rishu/reducer.py \
  -input /user/$(whoami)/csv_combiner_demo/sales.csv \
  -output /user/$(whoami)/output_with_combiner
```

### View reducer output
```bash
hdfs dfs -cat /user/$(whoami)/output_with_combiner/part-* | sort
```

---

## ##  **8. Comparing Both Runs (Key Learnings & How to Adapt Commands)**

>  *Important for students:*
>
> The commands shown in this chapter assume:
> - Your `mapper.py` and `reducer.py` are in your **home directory** (`/home/<username>`)
> - Your CSV dataset lives in a workspace folder like `~/mapreduce-combiner-csv-demo`
> - Your system uses **WSL + Hadoop pseudo-distributed mode**
>
> If your directory structure is different, **modify the paths accordingly**.
> For example:
> - If your scripts are in `~/mapreduce/scripts`, change `-mapper /home/rishu/mapper.py` → `-mapper ~/mapreduce/scripts/mapper.py`
> - If your CSV is in another folder, adjust the `-input` path accordingly.

---

## Updated Comparison Table**

| Metric | No Combiner | With Combiner | Notes |
|--------|-------------|---------------|--------|
| Map Input Records | 100k | 100k | Same (CSV rows) |
| Map Output Records | 100k | 100k | Same (one per row) |
| Combine Input Records | 0 | ~100k | Only in combiner job |
| Combine Output Records | 0 | **≈10** | One per product (HUGE drop) |
| Reduce Input Records | 100k | **≈10** | Network traffic reduced massively |
| Result Correctness | ✔️ | ✔️ | Combiners never change final output |


---

## ##  **9. Summary**

By running MapReduce with and without a combiner, we saw:
- How mapper outputs blow up shuffle data
- How combiners aggregate early
- Why Hadoop recommends using combiners whenever possible

This chapter is a complete, repeatable hands-on demonstration of **Combiner Performance Optimization in Hadoop MapReduce**.

---

---

© Copyright **PrimrIQ AI Services LLP**


