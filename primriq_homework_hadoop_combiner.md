# PrimrIQ Homework: Hadoop Combiner + Job Chaining Optimization

## Prepared by PrimrIQ

---

## 1. Scenario

You have joined **PrimrCart**, an e-commerce analytics startup processing millions of sales records daily from major platforms. Your responsibility is to optimize the Hadoop batch analytics pipeline that identifies the **hero product** (highest revenue product) within each category.

The current pipeline works but is becoming slow due to high shuffle volume between mappers and reducers.  
Your task is to introduce a **Combiner** into the existing Hadoop Streaming workflow without changing the final output of the system.

---

## 2. Business Requirement

You must enhance the performance of the “Hero Product Finder” Hadoop pipeline by:

- Reducing mapper output records  
- Reducing shuffle size  
- Maintaining identical final results  
- Keeping compatibility with the chained job workflow  

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

Fields include:

- category  
- product  
- unit_price  
- quantity  
- and other metadata  

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

## 5. Your Tasks

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

### End of Assignment  
Prepared by **PrimrIQ**
