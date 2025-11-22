# Chapter: Hadoop Job Chaining with Python Streaming (PrimrIQ Edition)

This chapter walks through a complete **two‑stage MapReduce job chaining pipeline** using Hadoop Streaming and Python.  
It uses the **ecommerce_sale.csv** dataset and demonstrates how to build:

1. **Job 1:** Aggregate total quantity and revenue per `(category, product)`  
2. **Job 2:** Determine the **hero product** (highest revenue item) in each category  

All code below is copied **exactly as provided** — no modifications have been made.

---

# 1. Introduction

Job chaining in Hadoop allows the output of one MapReduce job to become the input of another.  
This is useful for multi-step analytical tasks such as:

- Aggregating data  
- Finding top‑K items  
- Multi‑level grouping  
- Incremental transformations  

In this chapter, we implement a two‑stage chain to compute **hero products per category**.

---

# 2. Dataset Description

The input dataset is:

```
ecommerce_sale.csv
```

Stored in HDFS at:

```
/user/<username>/input_jobchaining_sales/ecommerce_sale.csv
```

Each row contains:

| Column | Description |
|--------|-------------|
| order_id | Unique order identifier |
| order_date | Date of purchase |
| customer_id | Customer identifier |
| region | Region of purchase |
| category | Product category |
| product | Product name |
| unit_price | Price per item |
| quantity | Quantity sold |
| payment_type | Payment method |

The goal is to compute revenue per row:

```
revenue = unit_price × quantity
```

---

# 3. Job Chaining Architecture

**Job 1 → Job 2**

- **Job 1** groups by **category + product** and computes:  
  - total quantity  
  - total revenue  

- **Job 2** groups by **category** and selects:  
  - product with maximum revenue  

This forms a standard MapReduce pipeline.

---

# 4. Python MapReduce Scripts (Exact Copies)

Below are the **exact scripts** you provided, included without modification.

---

## 4.1 mapper1.py

```python
#!/usr/bin/env python3
import sys
import csv

reader = csv.reader(sys.stdin)
header = True

for row in reader:
    if header:
        header = False
        continue

    if len(row) != 9:
        continue

    order_id, order_date, customer_id, region, category, product, unit_price, quantity, payment_type = row

    try:
        unit_price = int(unit_price)
        quantity = int(quantity)
    except ValueError:
        continue

    revenue = unit_price * quantity

    key = f"{category}|{product}"
    print(f"{key}\t{quantity}\t{revenue}")
```

---

## 4.2 reducer1.py

```python
#!/usr/bin/env python3
import sys

current_key = None
total_qty = 0
total_revenue = 0

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue

    parts = line.split("\t")
    if len(parts) != 3:
        continue

    key, qty_str, rev_str = parts

    try:
        qty = int(qty_str)
        revenue = int(rev_str)
    except ValueError:
        continue

    if key == current_key:
        total_qty += qty
        total_revenue += revenue
    else:
        if current_key is not None:
            # flush previous key
            category, product = current_key.split("|", 1)
            print(f"{category},{product},{total_revenue},{total_qty}")

        current_key = key
        total_qty = qty
        total_revenue = revenue

# flush last key
if current_key is not None:
    category, product = current_key.split("|", 1)
    print(f"{category},{product},{total_revenue},{total_qty}")
```

---

## 4.3 mapper2.py

```python
#!/usr/bin/env python3
import sys

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue

    parts = line.split(",")
    if len(parts) != 4:
        continue

    category, product, total_revenue_str, total_qty_str = parts

    print(f"{category}\t{product}\t{total_revenue_str}\t{total_qty_str}")
```

---

## 4.4 reducer2.py

```python
#!/usr/bin/env python3
import sys

current_category = None
best_product = None
best_revenue = 0
best_qty = 0

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue

    parts = line.split("\t")
    if len(parts) != 4:
        continue

    category, product, revenue_str, qty_str = parts

    try:
        revenue = int(revenue_str)
        qty = int(qty_str)
    except ValueError:
        continue

    if category == current_category:
        if revenue > best_revenue:
            best_product = product
            best_revenue = revenue
            best_qty = qty
    else:
        if current_category is not None:
            print(f"{current_category},{best_product},{best_revenue},{best_qty}")

        current_category = category
        best_product = product
        best_revenue = revenue
        best_qty = qty

# flush last category
if current_category is not None:
    print(f"{current_category},{best_product},{best_revenue},{best_qty}")
```

---

# 5. Running the Job Chain

Below is the **exact job-chain.sh** script you provided.

---

## 5.1 job-chain.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

USER_NAME=$(whoami)

HDFS_INPUT_DIR="/user/${USER_NAME}/input_jobchaining_sales"
HDFS_INPUT_FILE="${HDFS_INPUT_DIR}/ecommerce_sale.csv"

HDFS_OUT_JOB1="/user/${USER_NAME}/output_job1_category_product"
HDFS_OUT_JOB2="/user/${USER_NAME}/output_job2_category_hero"

if [ -z "${STREAMING_JAR:-}" ]; then
  echo "ERROR: STREAMING_JAR is not set."
  echo "Please add this to your ~/.bashrc and reload:"
  echo '  export STREAMING_JAR=$(ls $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar)'
  exit 1
fi

echo "=== Hadoop Job Chaining Demo (Hero Product per Category) ==="
echo "HDFS input file: $HDFS_INPUT_FILE"
echo "Using Streaming JAR: $STREAMING_JAR"
echo

if ! hdfs dfs -test -e "$HDFS_INPUT_FILE"; then
  echo "ERROR: HDFS input file not found: $HDFS_INPUT_FILE"
  exit 1
fi

echo "HDFS input file found:"
hdfs dfs -ls "$HDFS_INPUT_FILE"
echo

echo "=== Running Job 1: Aggregate (category, product) totals ==="

hdfs dfs -rm -r -skipTrash "$HDFS_OUT_JOB1" 2>/dev/null || true

hadoop jar "$STREAMING_JAR"   -files mapper1.py,reducer1.py   -mapper mapper1.py   -reducer reducer1.py   -input "$HDFS_INPUT_FILE"   -output "$HDFS_OUT_JOB1"

echo
echo "Job 1 completed. Sample output:"

FIRST_PART=$(hdfs dfs -ls "$HDFS_OUT_JOB1" | awk '/part-/{print $8; exit}')
hdfs dfs -cat "$FIRST_PART" | head

echo
echo "=== Running Job 2: Find hero product per category ==="

hdfs dfs -rm -r -skipTrash "$HDFS_OUT_JOB2" 2>/dev/null || true

hadoop jar "$STREAMING_JAR"   -files mapper2.py,reducer2.py   -mapper mapper2.py   -reducer reducer2.py   -input "$HDFS_OUT_JOB1"   -output "$HDFS_OUT_JOB2"

echo
echo "Job 2 completed. Final hero product per category:"
hdfs dfs -cat "$HDFS_OUT_JOB2/part-*"
echo

echo "=== DONE: Job chaining flow finished successfully ==="
echo "Job 1 output: $HDFS_OUT_JOB1"
echo "Job 2 output: $HDFS_OUT_JOB2"
```

---

# 6. Summary

In this chapter, we:

- Explained Hadoop job chaining  
- Described the dataset  
- Walked through each stage of the pipeline  
- Included **exact, unmodified code** for all mappers, reducers, and the job chain script  

This forms a fully working pipeline to compute **hero products per category** in Hadoop.

---

# End of Chapter
Prepared by **PrimrIQ**
