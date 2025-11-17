# Building Your First MapReduce Program: The Classic WordCount

## What is MapReduce?

MapReduce is a powerful programming paradigm for processing massive datasets across distributed computing clusters. Think of it as dividing a huge task among many workers, then combining their results. The "Hello World" of MapReduce is WordCount, a program that counts how many times each word appears in a text document.

In this tutorial, we'll build WordCount from scratch, test it on your local machine, and then scale it up to run on Hadoop.

## The MapReduce Philosophy

MapReduce works in two main stages:

**Map Stage:** Break down the problem
- Read input data
- Transform it into key-value pairs
- Emit intermediate results

**Reduce Stage:** Combine the results
- Receive grouped key-value pairs
- Aggregate or process them
- Produce final output

For WordCount:
- **Mapper:** Takes text → Outputs (word, 1) for each word
- **Reducer:** Takes (word, [1, 1, 1, ...]) → Outputs (word, total_count)

## Step 1: Writing the Mapper

The mapper is the first component we'll build. Its job is simple: read text, extract words, and output each word with a count of 1.

Create a file named `mapper.py`:

```python
#!/usr/bin/env python3
import sys

# Process each line from standard input
for line in sys.stdin:
    # Clean up whitespace
    line = line.strip()
    
    # Convert to lowercase for case-insensitive counting
    line = line.lower()
    
    # Split into individual words
    words = line.split()
    
    # Output each word with count 1
    for word in words:
        print(f"{word}\t1")
```

**What's happening here?**
- We read from `sys.stdin` line by line (standard Unix pattern)
- Convert everything to lowercase so "Hello" and "hello" count as the same word
- Split on whitespace to get individual words
- Print each word followed by a tab and the number 1

The output format `word\t1` is crucial—MapReduce expects tab-separated key-value pairs.

## Step 2: Writing the Reducer

The reducer receives all the intermediate results from the mapper(s), aggregates them, and produces the final word counts.

Create a file named `reducer.py`:

```python
#!/usr/bin/env python3
import sys

current_word = None
current_count = 0

# Process each line from standard input
for line in sys.stdin:
    line = line.strip()
    
    # Split the line into word and count
    word, count = line.split('\t')
    count = int(count)
    
    # Check if we're still processing the same word
    if current_word == word:
        current_count += count
    else:
        # New word encountered
        if current_word:
            # Output the result for the previous word
            print(f"{current_word}\t{current_count}")
        
        # Reset for the new word
        current_word = word
        current_count = count

# Output the last word
if current_word:
    print(f"{current_word}\t{current_count}")
```

**Key insight:** The reducer assumes its input is sorted by word. This is critical! When we see a different word, we know we've finished counting the previous word and can output its total.

**Why the last output?** After the loop ends, we still have one word's count in memory that needs to be printed.

## Step 3: Local Testing

Before running on Hadoop, always test locally. It's faster, easier to debug, and costs nothing.

### Create Test Data

Make a file called `input.txt`:

```
Hello World
Hello Hadoop
Goodbye World
Hello MapReduce
MapReduce is powerful
Hadoop is great
```

### Make Scripts Executable

```bash
chmod +x mapper.py reducer.py
```

### Run the Full Pipeline

Here's where Unix pipes shine:

```bash
cat input.txt | ./mapper.py | sort | ./reducer.py
```

**Breaking it down:**
1. `cat input.txt` — Reads your input file
2. `| ./mapper.py` — Pipes each line to the mapper
3. `| sort` — Sorts the output by word (critical for the reducer!)
4. `| ./reducer.py` — Pipes sorted data to the reducer

**You should see:**

```
goodbye	1
great	1
hadoop	2
hello	3
is	2
mapreduce	2
powerful	1
world	2
```

Perfect! "hello" appears 3 times, "hadoop" appears 2 times, and so on.

### Debug Individual Steps

If something's wrong, test each component:

```bash
# Test just the mapper
cat input.txt | ./mapper.py

# Test mapper + sort
cat input.txt | ./mapper.py | sort
```

This helps you pinpoint exactly where issues occur.

## Step 4: Running on Hadoop Streaming

Now for the real deal—running on a Hadoop cluster!

### Prepare Your HDFS Workspace

```bash
# Create input directory in HDFS
hdfs dfs -mkdir -p /user/yourname/wordcount/input

# Upload your input file
hdfs dfs -put input.txt /user/yourname/wordcount/input/

# Verify it's there
hdfs dfs -ls /user/yourname/wordcount/input
```

### Launch the MapReduce Job

Hadoop Streaming lets you use any executable (like our Python scripts) as mappers and reducers:

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar \
  -input /user/yourname/wordcount/input \
  -output /user/yourname/wordcount/output \
  -mapper mapper.py \
  -reducer reducer.py \
  -file mapper.py \
  -file reducer.py
```

**What each parameter means:**
- `-input`: Where to find input data in HDFS
- `-output`: Where to write results (must not already exist!)
- `-mapper`: The mapper script name
- `-reducer`: The reducer script name
- `-file`: Copies your local scripts to the cluster nodes

### Watch It Run

You'll see output like:

```
24/11/17 10:30:15 INFO mapreduce.Job: Running job: job_1234567890_0001
24/11/17 10:30:22 INFO mapreduce.Job:  map 0% reduce 0%
24/11/17 10:30:35 INFO mapreduce.Job:  map 100% reduce 0%
24/11/17 10:30:45 INFO mapreduce.Job:  map 100% reduce 100%
24/11/17 10:30:46 INFO mapreduce.Job: Job job_1234567890_0001 completed successfully
```

You can also monitor jobs in the Hadoop web UI (usually at `http://localhost:8088` or your cluster's ResourceManager address).

## Step 5: Reading HDFS Output

Success! Now let's see the results.

### View Output Directly

```bash
# List what's in the output directory
hdfs dfs -ls /user/yourname/wordcount/output

# Display the results
hdfs dfs -cat /user/yourname/wordcount/output/part-*

# Use less for large outputs
hdfs dfs -cat /user/yourname/wordcount/output/part-* | less

# Search for specific words
hdfs dfs -cat /user/yourname/wordcount/output/part-* | grep "hadoop"
```

### Download to Local Machine

```bash
# Download the entire output directory
hdfs dfs -get /user/yourname/wordcount/output ./wordcount_results

# View it locally
cat ./wordcount_results/part-*
```

### Understanding the Output Structure

Your output directory contains:
- `part-00000`, `part-00001`, etc. — Result files (one per reducer task)
- `_SUCCESS` — Empty marker file indicating job completion

Each part file has the format:
```
word1	count1
word2	count2
word3	count3
```

## Cleaning Up

Always clean up to avoid clutter and "directory exists" errors on reruns:

```bash
# Remove just the output directory
hdfs dfs -rm -r /user/yourname/wordcount/output

# Remove everything
hdfs dfs -rm -r /user/yourname/wordcount
```

## Common Issues and Solutions

**"Output directory already exists"**
- Delete it first: `hdfs dfs -rm -r /user/yourname/wordcount/output`

**Scripts not executing**
- Make them executable: `chmod +x mapper.py reducer.py`
- Check the shebang line: `#!/usr/bin/env python3`

**Different results locally vs. Hadoop**
- Verify your scripts don't have Windows line endings (use `dos2unix`)
- Ensure you're handling empty lines correctly

**ImportError or Python version issues**
- Hadoop nodes might have different Python versions
- Use `#!/usr/bin/env python3` explicitly
- Or specify Python: `-mapper "python3 mapper.py"`

## Taking It Further

Now that you have WordCount working, try these enhancements:

**1. Filter Short Words**
```python
# In mapper.py
for word in words:
    if len(word) >= 3:  # Only count words with 3+ characters
        print(f"{word}\t1")
```

**2. Remove Punctuation**
```python
import re

# In mapper.py, before splitting
line = re.sub(r'[^\w\s]', '', line)
```

**3. Top 10 Most Frequent Words**
```bash
# After getting output
hdfs dfs -cat /user/yourname/wordcount/output/part-* | \
  sort -t$'\t' -k2 -nr | head -10
```

**4. Process Multiple Files**
```bash
# Just add more files to input directory
hdfs dfs -put file1.txt file2.txt file3.txt /user/yourname/wordcount/input/
```



## What You've Learned

Congratulations! You've successfully:
- Written a MapReduce program from scratch
- Tested it locally using Unix pipes
- Deployed it on a Hadoop cluster
- Retrieved and analyzed distributed output

These fundamentals apply to far more complex MapReduce jobs. The same patterns you learned here scale to processing terabytes of log files, analyzing social media data, or building recommendation systems.

