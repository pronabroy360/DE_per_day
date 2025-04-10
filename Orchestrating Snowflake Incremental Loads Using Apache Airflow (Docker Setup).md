
---

# 🔧 **End-to-End Guide: Orchestrating Snowflake Incremental Loads Using Apache Airflow (Docker Setup)**

---

## **🧭 Objective**
Build an end-to-end **automated data pipeline** that:

- Loads data into a **Snowflake warehouse** incrementally using **Change Data Capture (CDC)**.
- Uses **Apache Airflow** to schedule and orchestrate tasks.
- Operates inside a **Dockerized environment** for portability and ease of deployment.

---

## 📦 **System Setup**

### **Tools & Technologies**
| Tool           | Purpose                                    |
|----------------|--------------------------------------------|
| Docker         | Containerized environment for Airflow      |
| Apache Airflow | Workflow orchestration engine              |
| Snowflake      | Cloud-based data warehouse                 |
| VS Code        | Code editor to create DAGs and SQL scripts |
| Python/Snowpark| Language for Snowflake stored procedures   |

---

## 🔧 **Step 1: Set Up Airflow in Docker**

### ✅ Prerequisites
- Docker Desktop installed and running.
- A code editor like Visual Studio Code.

### 📁 Folder Structure
```
/your-project/
│
├── dags/                  # DAG scripts
├── plugins/               # Optional custom plugins
├── logs/                  # Auto-generated logs
└── docker-compose.yaml    # Airflow setup
```

### ⚙️ docker-compose.yaml Configuration
Includes services:
- webserver (Airflow UI)
- scheduler
- postgres (metadata DB)

### 🚀 Start Airflow
```bash
docker-compose up
```

### 🌐 Airflow UI
- URL: [http://localhost:8080](http://localhost:8080)
- Username: `airflow`
- Password: `airflow`

---

## 🧊 **Step 2: Prepare Snowflake**

### 🧭 UI Navigation
- Navigate to Admin panel:
  - **Warehouses** → Ensure at least one exists.
  - **Users** → Confirm access and roles.
  - **Marketplace** → For future data integrations.

### 🗂️ Database Structure
- **Database**: `airflow`
- **Schemas**:
  - `bronze`: staging data
  - `silver`: cleaned/incremental data

### 🧪 Create Tables

#### 1. Bronze Table: `sales_data`
```sql
CREATE OR REPLACE TABLE bronze.sales_data (
  sale_id INT,
  product_name STRING,
  quantity INT,
  price FLOAT,
  last_modified_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 2. Silver Table: `sales_incremental`
```sql
CREATE OR REPLACE TABLE silver.sales_incremental (
  sale_id INT,
  product_name STRING,
  quantity INT,
  price FLOAT,
  last_modified_timestamp TIMESTAMP
);
```

> 📝 **Note:** `last_modified_timestamp` is crucial for incremental loading.

---

## 📥 **Step 3: Insert Sample Data**

### ➕ Load Initial Records
```sql
INSERT INTO bronze.sales_data (sale_id, product_name, quantity, price)
VALUES
  (1, 'Apple', 10, 2.5),
  (2, 'Banana', 5, 1.0),
  -- Add more rows up to 20
;
```

---

## 🧠 **Step 4: Create Stored Procedure for Incremental Loading**

### 🔍 Incremental Logic
- Compare `last_modified_timestamp` between `bronze.sales_data` and `silver.sales_incremental`.
- Load only rows with newer timestamps.

### 🐍 Python Stored Procedure (Snowpark)
```sql
CREATE OR REPLACE PROCEDURE silver.incremental_load()
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.8'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'run'

AS
$$
def run(session):
    max_ts = session.sql("""
        SELECT MAX(last_modified_timestamp) FROM silver.sales_incremental
    """).collect()[0][0]

    if max_ts is None:
        df_new = session.table("bronze.sales_data")
    else:
        df_new = session.sql(f"""
            SELECT * FROM bronze.sales_data
            WHERE last_modified_timestamp > '{max_ts}'
        """)

    df_new.write.mode("append").save_as_table("silver.sales_incremental")
    return "Incremental load complete"
$$;
```

### ✅ Test the Procedure
```sql
CALL silver.incremental_load();
```

---

## 🔗 **Step 5: Connect Airflow to Snowflake**

### 🔐 Airflow UI → Admin → Connections

| Field        | Value Example                                           |
|--------------|--------------------------------------------------------|
| Conn ID      | `snowflake_conn`                                       |
| Conn Type    | Snowflake                                               |
| Account      | `abc12345.region.aws`                                  |
| Warehouse    | `COMPUTE_WH`                                           |
| Database     | `airflow`                                              |
| Schema       | `silver`                                               |
| Role         | `ACCOUNTADMIN`                                         |
| Login/Password | Your Snowflake credentials                           |

### 🧩 Install Airflow Snowflake Provider (if missing)
```bash
pip install apache-airflow-providers-snowflake
```
Restart your Airflow environment.

---

## 🧮 **Step 6: Create DAG in Airflow to Trigger the Procedure**

### 📁 File: `dags/snowflake_query_dag.py`
```python
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2024, 1, 1),
    'retries': 1
}

with DAG(
    dag_id='snowflake_incremental_load_dag',
    schedule_interval='@daily',
    catchup=False,
    default_args=default_args,
    tags=['snowflake', 'incremental']
) as dag:

    run_incremental_load = SnowflakeOperator(
        task_id='incremental_load',
        sql='CALL airflow.silver.incremental_load()',
        snowflake_conn_id='snowflake_conn'
    )

    run_incremental_load
```

### ✅ Deploy DAG
- Save file → Airflow auto-detects it.
- Unpause the DAG.
- Manually trigger or wait for the schedule.

---

## 🔍 **Step 7: Verify Data Loads**

### 🧪 Initial Run
- Since `silver.sales_incremental` is empty:
  - Loads all 20 rows from `bronze.sales_data`.

### 🔄 Add More Data
```sql
INSERT INTO bronze.sales_data (sale_id, product_name, quantity, price)
VALUES
  (21, 'Orange', 7, 1.8),
  (22, 'Mango', 3, 2.2),
  -- 8 more rows...
;
```

- Total rows in bronze: 30
- Re-run DAG → new 10 rows added to silver (now 30 total)

### 🔎 Validate
```sql
SELECT COUNT(*) FROM silver.sales_incremental;
```

---

## 📈 **Step 8: Optimization & Best Practices**

### 🔔 Notifications
- Add email alerts on task failure.

### 🔁 Retry Logic
- Increase retries for network-sensitive tasks.

### 🕓 Cron Scheduling
- Customize `schedule_interval`:
  - Every 15 mins: `'*/15 * * * *'`
  - Hourly: `'@hourly'`
  - Daily: `'@daily'`

---

## 🧠 **Key Takeaways**

| Component         | Role                                                         |
|------------------|--------------------------------------------------------------|
| Bronze Table      | Staging layer receiving raw data                            |
| Silver Table      | Incremental layer for filtered new/changed data             |
| Stored Procedure  | Executes filtering and insertion logic                      |
| Airflow DAG       | Triggers the stored procedure on schedule                   |
| Docker            | Hosts the orchestration layer locally or in production      |
| Snowflake         | Centralized data warehouse for historical and live data     |

---

## 🗺️ **Pipeline Architecture Overview**

```text
        ┌──────────────┐
        │  External    │
        │  Data Source │
        └──────┬───────┘
               │
               ▼
       ┌─────────────────┐
       │ bronze.sales_data│ ←── Insert Raw Data (with timestamps)
       └────────┬────────┘
                │
                ▼ (via stored procedure)
  ┌─────────────────────────────┐
  │ silver.sales_incremental    │ ←── New/Changed Records Only
  └─────────────────────────────┘
                ▲
                │
       ┌─────────────────┐
       │ Airflow DAG     │ ←── Triggers `CALL silver.incremental_load()`
       └─────────────────┘
```

---

