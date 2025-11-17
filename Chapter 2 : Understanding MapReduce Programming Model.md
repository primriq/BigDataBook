# MapReduce Programming Model — A Clear Beginner-Friendly Guide

The MapReduce model is the core programming framework in Hadoop. It provides a simple way to process large datasets using two main functions: **Map** and **Reduce**. Hadoop executes these functions in a distributed manner so that data can be processed faster and more efficiently.

This article explains all important concepts you need to understand before writing or teaching MapReduce programs.

---

## 1. MapReduce Programming Model

MapReduce works in three main stages:

1. **Map phase** – reads input and produces intermediate key–value pairs
2. **Shuffle and Sort phase** – moves and groups intermediate data
3. **Reduce phase** – processes grouped keys and produces final output

Every MapReduce job follows this pipeline, no matter how simple or complex the logic is.

---

## 2. What a Mapper Does

The **Mapper** is the first function that runs in a MapReduce program.

* It takes an **input split** (a portion of a file).
* It processes the data **record by record**.
* For each record, it emits **intermediate key–value pairs**.
* The output does not need to be sorted.
* The number of mappers depends on the number of input splits.

Typical mapper responsibilities:

* Parsing input
* Extracting fields
* Filtering or transforming data
* Emitting key–value pairs for the reducer

A mapper does not communicate with other mappers. Each mapper works independently.

---

## 3. What a Reducer Does

The **Reducer** receives grouped keys and all values for each key.

* It processes one key at a time.
* It iterates over the list of values for that key.
* It produces the **final output key–value pair**.
* Reducer outputs are stored in HDFS.

Typical reducer responsibilities:

* Aggregation (sum, count, max, min)
* Merging
* Formatting final output

The number of reducers can be controlled by the user depending on how much parallelism is needed.

---

## 4. What Shuffle and Sort Means

Shuffle and sort happens **between map and reduce** phases.

### Shuffle

* Transfers mapper output to the reducers.
* Ensures that all values for the same key reach the same reducer.
* Moves data across the cluster when needed.

### Sort

* Sorts the intermediate keys.
* Groups values for identical keys together.
* Reducers always receive data in sorted order.

This phase is managed entirely by Hadoop. Users do not write code for it.

---

## 5. What a Combiner Is

A **Combiner** is an optional mini-reducer that runs on the mapper's output **before shuffle**.

* It reduces the amount of data sent over the network.
* It performs local aggregation (e.g., sum, min, max).
* It must be **idempotent** and not change correctness even if run multiple times.

Use combiners for operations like:

* Sum
* Count
* Min/Max

Do not use combiners if your reduce logic depends on the full, original list of values.

---

## 6. What a Partitioner Is

A **Partitioner** decides which reducer will process a particular key.

* It receives a key from the mapper.
* It calculates a partition number (usually by hashing the key).
* All keys assigned to the same partition go to the same reducer.

Default partitioner:

```
partition = hash(key) % number_of_reducers
```

Custom partitioners are useful when:

* You want grouped data to be distributed in a specific pattern.
* You want to avoid data skew (too much data going to one reducer).

---

## 7. How Hadoop Guarantees Sorting and Grouping

Hadoop guarantees that:

* All values for a given key go to the **same reducer** (via the partitioner).
* Keys are **sorted** before reaching the reducer (via sort phase).
* Each reducer receives a **sorted list of keys**, one group at a time.

This is achieved automatically through:

* The partitioner
* The shuffle phase
* The sort phase
* The grouping comparator

Users do not need to manually sort or group intermediate data.

---

## 8. Fault Tolerance Basics

Hadoop provides strong fault tolerance for both map and reduce tasks.

**If a mapper fails:**

* Hadoop reruns the mapper on another node using the original input split.
* No data is lost because mapper outputs are temporary.

**If a reducer fails:**

* Hadoop reruns the reducer from the beginning.
* It fetches mapper outputs again from each map task.

**If a node fails:**

* Tasks running on that node are rescheduled on other nodes.
* Replication in HDFS ensures data is still available.

**Speculative execution:**

* Hadoop can run duplicate copies of slow tasks.
* The fastest successful copy is used.

These mechanisms ensure that MapReduce jobs continue running even with hardware failures.
