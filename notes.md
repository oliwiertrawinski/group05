# DAT535 - Assignment Report: CI/CD Pipeline & Infrastructure Setup
**Group:** group05  
**Environment:** OpenStack VM & GitHub Actions Self-Hosted Runner  

## 1. Overview and Purpose
The goal of this assignment was to transition from manual data pipeline execution to an automated, industry-standard **CI/CD (Continuous Integration/Continuous Deployment)** framework utilizing the **Medallion Architecture** (Bronze, Silver, and Gold data layers) inside Apache Spark.

By dividing our pipeline into **Dev** (Development) and **Prod** (Production) environments, we ensure that experimental code or unexpected pipeline failures never corrupt critical, business-ready production data.

---

## 2. Infrastructure Architecture & "Runs-On: Self-Hosted"
Our PySpark pipelines require a configured environment with Java 11/8, Python 3.11, and Apache Spark dependencies. Because GitHub's cloud-hosted servers do not have these tools pre-installed and cannot access our secure university network (`5gnuc1.ux.uis.no`), we utilized a **Self-Hosted Runner**.

By specifying `runs-on: self-hosted` inside our GitHub Actions workflows (`dat535-dev.yml` and `dat535-prod.yml`), we instruct GitHub to establish a secure outbound polling connection to our OpenStack VM. This enables:
* Native execution inside our isolated Python virtual environment (`~/spark-env/`).
* High-performance execution leveraging our VM's Spark Standalone cluster.
* Absolute network isolation behind the university firewall.

---

## 3. Step-by-Step Implementation

### Step 1: Git Repository Setup & Safeguards
We cloned the instructor's template and remapped the remote branches:
```bash
git clone https://github.com group05
cd group05
git remote rename origin upstream
git remote add origin https://github.com
```
* **Purpose:** Renaming the template to `upstream` prevents accidental pushes to the source project, ensuring full read/write decoupling into our team's `origin` repository.

### Step 2: Self-Hosted Runner Background Configuration
We downloaded and initialized the active runner interface inside the VM:
```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.337.0.tar.gz -L https://github.com...
tar xzf ./actions-runner-linux-x64-2.337.0.tar.gz
./config.sh --url https://github.com --token [REDACTED_TOKEN]
```
After validating its functionality interactively via `./run.sh`, we decoupled it from our active SSH session by daemonizing it as a persistent system background service:
```bash
sudo ./svc.sh install
sudo ./svc.sh start
```

### Step 3: Triggering the Dev Pipeline
We initialized our development branch, injected a verification logging statement into `lab2_pipeline.py`, and pushed the changes:
```bash
git checkout -b dev
git push -u origin dev
# Added print("CI/CD pipeline test trigger active!")
git add lab2_pipeline.py
git commit -m "feat(lab2): implement ingest and testing logging statement"
git push origin dev
```
This automated change seamlessly triggered the `dev` pipeline, processing scratch datasets inside `~/spark-lab-data/dev/`.

### Step 4: Production Release Management
1. Opened a **Pull Request (PR)** on GitHub targeting `main ← dev`.
2. Reviewed the code diff and executed the merge.
3. Triggered `dat535-prod.yml`, which verified our data constraints and completed the generation of our enterprise production tables under `~/spark-lab-data/prod/shared/` (`bronze`, `silver`, `gold`).

---

## 4. Challenges Encountered & Technical Resolutions

### Challenge 1: Hidden Environment Protection Rules
* **Problem:** When navigating to the `main` environment settings to add mandatory peer reviewers, the **Deployment Protection Rules** section was entirely invisible, and attempting to recreate it caused a *"Name has already been taken"* constraint error.
* **Root Cause:** The repository was initially configured as **Private**. GitHub standard tiers restrict environment gating and reviewers strictly to **Public** repositories.
* **Resolution:** Swapped repository visibility to **Public** in the GitHub Danger Zone options, instantly unlocking the `Required reviewers` toggle interface for the `main` deployment block.

### Challenge 2: Git Author Identity Profile Error
* **Problem:** When attempting our first pipeline configuration update commit, Git threw a fatal error: `Author identity unknown... fatal: unable to auto-detect email address`.
* **Root Cause:** The newly provisioned Ubuntu image on the OpenStack VM did not have localized global Git user markers bound to the terminal configuration profile.
* **Resolution:** Registered our explicit identity variables using target context commands before repushing:
  ```bash
  git config --global user.email "your-email@example.com"
  git config --global user.name "Your Name"
  ```

---

## 5. Verification Checklist Achievement
* [x] Created `dev` GitHub Environment (Open configuration mode)
* [x] Created `main` Production Environment (Equipped with manual reviewer gates)
* [x] Successfully pushed to `dev` branch -> Automated Dev Action triggered and passed
* [x] Created, approved, and merged PR into `main` -> Production workflow completed upon manual approval
* [x] Data directories validated locally on the VM (`~/spark-lab-data/prod/shared/`)


# Lab 3 Cheat Sheet: Advanced Spark & Production Patterns

## 📌 Core Topics & Key Takeaways

* **Window Functions:** Compute analytics across related rows without slow self-joins.
    * `row_number()`: Unique sequential numbers (no ties).
    * `rank()`: Leaves gaps after ties (e.g., 1, 2, 2, 4).
    * `dense_rank()`: No gaps after ties (e.g., 1, 2, 2, 3).
    * `LAG` / `LEAD`: Access data from the previous or next row.
    * `rowsBetween()`: Defines moving frames for running totals.

* **Partitioning Strategies:** How data folders are grouped on disk.
    * **Partition Pruning:** Spark skips folders that do not match `WHERE` filters, saving read time.
    * **Small File Problem:** Over-partitioning by columns with high unique values (high cardinality) creates thousands of tiny files, destroying write/read performance.

* **Caching (`.cache()` / `.persist()`):** 
    * Stores a DataFrame in memory so Spark does not recompute it for multiple downstream actions.
    * `unpersist()`: Frees up memory when done.

* **Joins & Broadcast Optimization:**
    * `left_semi`: Keeps left rows *with* a match in the right table (filter-only).
    * `left_anti`: Keeps left rows *without* a match in the right table (finds orphans).
    * **Broadcast Join:** Copies a small table to all workers, skipping the slow network shuffle of the big table.

* **Optimization Rules:**
    * **Filter Early:** Drop rows before joins to minimize network traffic.
    * **Column Pruning:** Use `.select()` early to reduce memory and serialization size.
    * **`.explain(True)`:** Shows the execution plans generated by the Catalyst Optimizer.

* **UDFs vs. Native Functions:**
    * **Native (`when`, `col`):** Fastest. Fully optimized in the JVM.
    * **Pandas UDF:** Medium. Vectorized batches using Apache Arrow (JVM ↔ Python).
    * **Python UDF:** Slowest. Processes row-by-row; heavy serialization overhead.

* **Production Patterns:**
    * **Incremental Processing:** Processes only data newer than the `last_processed` timestamp.
    * **SCD Type 2:** Keeps full history of changing data using `valid_from`, `valid_to`, and `is_current` flags.

---

## 💬 Assignment Defense Q&A (Short & Direct)

### Q1: Why did multi-level partitioning (`event_date`, `country`) slow down the write time?
**Answer:** It triggered the **Small File Problem**. The dataset is small, so splitting it into combinations of dates and countries created hundreds of tiny files. Spark spent more time creating filesystem directories than writing actual data.

### Q2: What is the difference between a `left_semi` and a `left_anti` join?
**Answer:** Neither appends new columns. `left_semi` acts as an `IN` filter (keeps rows that match the right table). `left_anti` acts as a `NOT IN` filter (keeps rows that do not match the right table).

### Q3: Why is a global `Window.orderBy()` without a `partitionBy()` dangerous?
**Answer:** It forces Spark to **shuffle all data into a single partition on one node** to sort it. In production, this causes a major bottleneck and crashes the cluster with an `OutOfMemoryError`.

### Q4: If Spark automatically optimizes queries, why should we manually filter early?
**Answer:** The Catalyst optimizer cannot always push filters past complex boundaries like outer joins without altering the logic. Filtering early guarantees smaller data sizes before expensive operations.

### Q5: When do you use a Python UDF vs. a Pandas UDF?
**Answer:** Use **Pandas UDF** when you need custom Python libraries (like Scikit-Learn) because it processes data in fast vectorized batches via Arrow. Avoid **Python UDF** completely because its row-by-row processing kills performance.

### Q6: What does the `.config("spark.sql.shuffle.partitions", "8")` line do in your setup? *(New)*
**Answer:** It reduces the number of partitions created during data shuffles from the default of 200 down to 8. This matches our small lab dataset size and prevents creating empty tasks.

### Q7: What is the purpose of `.withWatermark()` in the streaming section? *(New)*
**Answer:** It handles late data. It tells Spark how long (e.g., 10 seconds) to wait for delayed streaming events before closing the time window and deleting old states from memory.

### Q8: In your Data Quality framework pattern, how do you check for duplicate rows? *(New)*
**Answer:** By subtracting the distinct row count of a primary key (like `event_id`) from the total row count (`df.count()`). If the result is greater than 0, duplicates exist.
