# Hadoop Combiner + Job Chaining Optimization

## Prepared by PrimrIQ

---

## 1. Scenario

You have recently joined PrimrCart, an e-commerce analytics startup that provides real-time insights to partner retail brands.
PrimrCart collects millions of sales transactions daily from marketplaces such as:

-- Amazon
-- Flipkart
-- Ajio
-- BigBasket

The analytics engine is built on Hadoop (pseudo-distributed) for batch processing.
Your team is responsible for identifying the hero product in each product category — the item that generates the maximum revenue.

Recently, the engineering team noticed that the daily aggregation job is running slower due to a large number of mapper output records.
Your manager assigns you the following task:.

---

## 2. Business Requirement

Business Requirement

Improve the performance of the “Hero Product Finder” Hadoop pipeline by adding a Combiner to reduce mapper output size and verifying that the chained pipeline continues to produce correct results.

PrimrCart leadership wants to reduce:

  -- Network I/O between mappers and reducers

  -- Shuffle size

   -- Total job runtime

However, the final output must remain consistent.

---

## 3. Dataset Information

The dataset file:

```
ecommerce_sale.csv
```

stored in HDFS at:

```
/user/<username>/input_jobchaining_sales/ecommerce_sale.csv
```

The fields include:

category
product
unit_price
quantity
… and other metadata

You must not modify the dataset. 

---

## 4. Current Pipeline Overview (Without Combiner)

### **Job 1 — Category × Product Aggregation**

Mapper output:
```
category|product    quantity    revenue
```

Reducer output:
```
category,product,total_revenue,total_qty
```

### **Job 2 — Hero Product Selection**

Mapper output:
```
category    product    total_revenue    total_qty
```

Reducer output:
```
category,hero_product,total_revenue,total_qty
```

---

## 5. Your Assignment

PrimrCart’s CTO asks you to optimize the current Hadoop Streaming pipeline.

You must complete the following tasks:



### **Task 1 — Run Baseline Pipeline (No Combiner)**

1. Execute `run_job_chain.sh`.
2. Record:
   - Job 1 runtime  
   - Map output records  
   - Reduce input records  
   - Shuffle size  
3. Save final hero product output (baseline).

---

### **Task 2 — Implement the Combiner**

Create a file named:

```
combiner1.py
```

It must:

- Aggregate quantity and revenue per `category|product`
- Produce reducer-compatible output  
- Mirror reducer logic safely  

Write a brief explanation:
- Why can reducers sometimes serve as combiners?
- When is this safe?

---

### **Task 3 — Integrate the Combiner into Job 1**

Modify Job 1 configuration in:

```
run_job_chain.sh
```

Add the line:

```
-combiner combiner1.py
```

Re-run the chained pipeline.

---

### **Task 4 — Performance Comparison**

Compare the pipeline before and after using the combiner.

Answer:

1. Does final output match the baseline? Why or why not?  
2. How did the combiner change shuffle size?  
3. What changed in:
   - Map output records  
   - Reduce input records  
   - Job runtime  
4. Why do combiners reduce network traffic?  

---

### **Task 5 — Bonus Challenges (Choose One)**

#### **Option A — Region-Level Hero Products**

Modify keys to:

```
region|category|product
```

Compute hero products per (region, category).

#### **Option B — Add Job 3**

Create a third MapReduce job to compute:

```
Total revenue of all hero products combined.
```

#### **Option C — Engineering Report**

Write a 1–2 page report explaining:

- Original bottleneck  
- Impact of combiner  
- Why job chaining increases modularity  
- Proposed future improvements  

---

## 6. Deliverables

Submit:

1. `combiner1.py`  
2. Updated `run_job_chain.sh`  
3. Logs (before and after using combiner)  
4. Final hero product outputs  
5. Performance comparison write-up  
6. Bonus challenge solution (if done)  

---


Prepared by **PrimrIQ**
