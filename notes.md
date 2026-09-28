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
