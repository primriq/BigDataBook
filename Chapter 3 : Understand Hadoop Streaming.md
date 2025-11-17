# Hadoop Streaming with Python — A Complete Beginner Guide

Hadoop Streaming allows you to write MapReduce programs in any language that can read from **stdin** (standard input) and write to **stdout** (standard output). Python is one of the most common choices because it is easy to read, easy to debug, and perfect for working with text-based data.

This guide explains how Hadoop Streaming works, how input flows through your Python mapper and reducer, and why this model is great for beginners.

---

## 1. What Hadoop Streaming Is

Hadoop Streaming is a utility that comes with the Hadoop distribution. It allows you to run MapReduce jobs using executable scripts or programs written in:

* Python
* Bash
* Perl
* Ruby
* Any language that supports standard input and output

You do not need to write Java code or understand complex Hadoop APIs. Hadoop handles all the distributed computing tasks, such as:

* Splitting input files into manageable chunks
* Distributing work across multiple nodes
* Shuffling data between mappers and reducers
* Sorting keys automatically
* Managing reducer processes
* Writing final output to HDFS

Your only responsibility is to write two simple scripts: a mapper and a reducer.

---

## 2. How Hadoop Sends Input to the Mapper (via stdin)

When you start a Hadoop Streaming job, Hadoop automatically performs the following steps:

1. **Splits the input file** into input splits (typically 128 MB or 256 MB blocks)
2. **Starts multiple mapper tasks** across the cluster, one for each input split
3. **Sends each mapper its portion of the data** line by line through standard input

Your mapper reads this data using Python's standard input stream. Here's how it works:

```python
import sys

for line in sys.stdin:
    # Each line is a string from the input file
    # Process each line here
    pass
```

### Important Points:

* Each line comes from HDFS, but you never directly access HDFS in your code
* Hadoop pipes the data into your script automatically
* The mapper receives one line at a time from its assigned input split
* Different mappers process different parts of the file in parallel
* You don't need to open files or manage file handles

This streaming approach makes Python MapReduce programs incredibly simple. You're just reading from standard input like you would read user input from a terminal.

---

## 3. Mapper Prints "key TAB value" to stdout

The mapper must output data in a specific format that Hadoop can understand:

```
key<TAB>value
```

The key and value must be separated by a **tab character** (`\t`), not spaces. This is crucial because Hadoop uses the tab to split the key from the value.

### Example Output:

```
apple	1
banana	1
apple	1
orange	2
banana	1
```

Your Python mapper prints these lines using `print()` or `sys.stdout.write()`. Each printed line becomes mapper output.

### Example Mapper (Word Count):

```python
import sys

for line in sys.stdin:
    # Remove leading/trailing whitespace
    line = line.strip()
    
    # Split the line into words
    words = line.split()
    
    # Emit each word with count of 1
    for word in words:
        print(f"{word}\t1")
```

### What Happens Next:

* Each line you print goes to Hadoop's shuffle mechanism
* Hadoop collects all these key–value pairs from all mappers
* The pairs are prepared for the reduce phase
* You don't write any code to send data to reducers—Hadoop handles it automatically

### Key Points:

* Always use `\t` (tab) as the separator, never spaces or commas
* The key can be any string (a word, number, or composite key)
* The value can be any string (often a number or structured data)
* Both key and value should be on a single line
* Use `strip()` to remove any unwanted newline characters from input

---

## 4. Hadoop Automatically Sorts by Key

After all mappers finish their work, Hadoop performs the **shuffle and sort** phase automatically. This is one of the most powerful features of Hadoop, and you don't write any code for it.

### What Hadoop Does:

1. **Collects all key–value pairs** from every mapper across the cluster
2. **Sorts them by key** in ascending order (alphabetically for strings, numerically for numbers)
3. **Groups identical keys together** so all values for the same key are adjacent
4. **Partitions keys** and sends each partition to a specific reducer
5. **Ensures each reducer receives keys in sorted order**

### Example:

If your mappers produce this output:

```
apple	1
banana	1
apple	1
cherry	1
banana	1
apple	1
```

After shuffle and sort, the data sent to reducers looks like:

```
apple	1
apple	1
apple	1
banana	1
banana	1
cherry	1
```

### Important Guarantees:

* All values for a given key go to the **same reducer**
* Keys are delivered in **sorted order**
* If 10 mappers output pairs for the key "apple", Hadoop merges them into one sorted stream
* The reducer receives one key group at a time
* You can rely on this behavior—it's guaranteed by the framework

This automatic sorting and grouping is what makes MapReduce powerful for aggregation tasks. You never need to manually sort data or worry about which reducer gets which key.

---

## 5. Reducer Receives Grouped Keys from stdin

The reducer receives sorted, pre-grouped key–value pairs through standard input, just like the mapper received its input. Each line has the same format:

```
key<TAB>value
```

The reducer reads this stream line by line:

```python
import sys

for line in sys.stdin:
    # Remove whitespace and split by tab
    key, value = line.strip().split('\t')
    # Process the key and value
    pass
```

### Key Characteristics:

* Keys appear in **sorted order**
* All values for the same key appear **consecutively**
* You can detect when a key changes by comparing the current key with the previous key
* Each reducer processes a subset of all keys (determined by the partitioner)

### Processing Strategy:

Because keys are sorted and grouped, you can use a simple pattern to aggregate values:

1. Keep track of the current key
2. Accumulate values while the key stays the same
3. When the key changes, output the result for the previous key
4. Start accumulating for the new key

This pattern is efficient because you only need to keep one key's data in memory at a time.

---

## 6. Reducer Prints Final Results to stdout

Just like the mapper, the reducer writes output to standard output using the same tab-separated format:

```
key<TAB>final_value
```

This output becomes the final MapReduce result and is automatically written to HDFS by Hadoop.

### Example Reducer (Word Count):

```python
import sys

current_word = None
current_count = 0

for line in sys.stdin:
    # Parse input
    word, count = line.strip().split('\t')
    count = int(count)
    
    # Check if we're still processing the same word
    if word == current_word:
        # Same word, add to count
        current_count += count
    else:
        # New word encountered
        if current_word:
            # Output the result for the previous word
            print(f"{current_word}\t{current_count}")
        
        # Reset for new word
        current_word = word
        current_count = count

# Don't forget to output the last word
if current_word:
    print(f"{current_word}\t{current_count}")
```

### How This Works:

1. Read the first line: `word="apple"`, `count=1`
2. Set `current_word="apple"`, `current_count=1`
3. Read the next line: `word="apple"`, `count=1`
4. Since `word == current_word`, add to count: `current_count=2`
5. Read the next line: `word="banana"`, `count=1`
6. Since `word != current_word`, print `"apple\t2"` and reset
7. Continue until all lines are processed
8. Print the final accumulated result

### Important Notes:

* Always handle the **last key** after the loop ends
* Convert values to appropriate types (e.g., `int(count)`)
* The output format must still be `key<TAB>value`
* This output is written to HDFS automatically by Hadoop
* You can have multiple reducers, each writing to separate output files

---

## 7. Why Python is Excellent for Beginners (stdin/stdout model)

Python combined with Hadoop Streaming is ideal for new learners and quick prototyping. Here's why:

### Simplicity:

* **No Java required** – Avoid the complexity of Java syntax and compilation
* **No Hadoop API** – You don't need to learn Hadoop's Java API or class hierarchies
* **Just two scripts** – A mapper and a reducer, each typically under 20 lines
* **Standard I/O only** – Use familiar `print()` and `sys.stdin` that you already know

### Easy Debugging:

* **Run locally first** – Test your mapper and reducer on your laptop before using Hadoop
* **Use sample data** – Create small test files and pipe them through your scripts
* **Standard Python debugging** – Use `print()` statements, debuggers, and IDE tools
* **No cluster required for development** – Develop and test without Hadoop installed

### Testing Example:

```bash
# Test mapper locally
cat input.txt | python mapper.py

# Test full pipeline locally
cat input.txt | python mapper.py | sort | python reducer.py
```

The `sort` command simulates Hadoop's shuffle and sort phase. If your output looks correct locally, it will work on Hadoop.

### Flexibility:

* **Rich Python libraries** – Use regex, JSON parsing, CSV handling, and more
* **Text processing strength** – Python excels at string manipulation
* **Quick iterations** – Edit code, save, and rerun in seconds
* **Easy to read** – Code is self-documenting and easy to share with teammates

### Perfect for Learning:

* Most beginners can write a working word count MapReduce program in **less than 20 lines**
* The logic is straightforward: read, process, print
* Focus on the MapReduce concept, not the implementation details
* Easy to experiment with different algorithms

### Example: Complete Word Count (Mapper + Reducer)

**mapper.py** (7 lines):
```python
import sys
for line in sys.stdin:
    words = line.strip().split()
    for word in words:
        print(f"{word}\t1")
```

**reducer.py** (12 lines):
```python
import sys
current_word = None
current_count = 0

for line in sys.stdin:
    word, count = line.strip().split('\t')
    count = int(count)
    if word == current_word:
        current_count += count
    else:
        if current_word:
            print(f"{current_word}\t{current_count}")
        current_word = word
        current_count = count

if current_word:
    print(f"{current_word}\t{current_count}")
```

That's it! A complete, production-ready MapReduce program in 19 lines of Python.

---

## 8. Running a Hadoop Streaming Job

To execute your Python MapReduce program on a Hadoop cluster, use the following command:

```bash
hadoop jar /path/to/hadoop-streaming.jar \
  -input /input/path/in/hdfs \
  -output /output/path/in/hdfs \
  -mapper mapper.py \
  -reducer reducer.py \
  -file mapper.py \
  -file reducer.py
```

### Command Breakdown:

* `hadoop jar` – Runs a JAR file (Hadoop Streaming is packaged as a JAR)
* `-input` – Path to input data in HDFS
* `-output` – Path where results will be written in HDFS (must not exist)
* `-mapper` – Name of your mapper script
* `-reducer` – Name of your reducer script
* `-file` – Copies the script to each node so it can be executed

### Optional Parameters:

* `-numReduceTasks N` – Set number of reducers (default is 1)
* `-combiner reducer.py` – Use reducer as a combiner for optimization
* `-D mapreduce.job.reduces=N` – Alternative way to set reducer count

---

## Summary

Hadoop Streaming with Python provides a simple, powerful way to write MapReduce programs:

1. **Mapper reads from stdin** – Processes input line by line
2. **Mapper writes to stdout** – Emits key–value pairs separated by tabs
3. **Hadoop sorts and groups** – Automatically handles shuffle and sort
4. **Reducer reads from stdin** – Receives sorted, grouped key–value pairs
5. **Reducer writes to stdout** – Produces final results

This stdin/stdout model makes MapReduce accessible to beginners while still providing the full power of distributed computing. You can focus on the logic of your problem instead of the complexities of distributed systems.
