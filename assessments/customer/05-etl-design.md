# Customer Data ETL Design

## Overview
This document outlines the ETL architecture and processes for customer data integration.

## ETL Architecture

### High-Level Design
```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│   Sources   │ --> │   Extract    │ --> │  Transform  │ --> │     Load     │
│  Systems    │     │   (Raw)      │     │  (Staging)  │     │(Presentation)│
└─────────────┘     └──────────────┘     └─────────────┘     └──────────────┘
```

## Extract Processes

### Source System 1: CRM API
```python
# Extract Pattern
def extract_crm_customers():
    """Extract customer data from CRM API"""
    config = {
        'endpoint': 'https://api.crm.com/v2/customers',
        'auth_type': 'oauth2',
        'page_size': 1000,
        'incremental_field': 'last_modified'
    }
    
    # Get last successful run timestamp
    last_run = get_last_run_timestamp('crm_customers')
    
    # Extract incremental data
    params = {
        'modified_since': last_run,
        'page_size': config['page_size']
    }
    
    # Pagination logic
    all_records = []
    page = 1
    while True:
        response = api_call(config['endpoint'], params, page)
        all_records.extend(response['data'])
        if not response['has_next']:
            break
        page += 1
    
    return all_records
```

### Source System 2: Database
```sql
-- Extract Pattern for Database Sources
CREATE OR REPLACE TASK extract_erp_customers
  WAREHOUSE = ETL_WH
  SCHEDULE = 'USING CRON 0 2 * * * UTC'
AS
  COPY INTO RAW.SRC_ERP.CUSTOMERS
  FROM (
    SELECT 
      customer_id,
      customer_name,
      email,
      phone,
      address,
      created_date,
      modified_date,
      CURRENT_TIMESTAMP() as extracted_timestamp
    FROM ERP_DB.CUSTOMERS
    WHERE modified_date > (
      SELECT MAX(extracted_timestamp) 
      FROM RAW.SRC_ERP.CUSTOMERS
    )
  );
```

## Transform Processes

### Data Quality and Cleansing
```sql
-- Staging Layer Transformations
CREATE OR REPLACE VIEW STAGING.STG_CUSTOMERS AS
WITH cleaned_data AS (
  SELECT
    -- Standardize IDs
    UPPER(TRIM(customer_id)) as customer_id,
    
    -- Name standardization
    INITCAP(TRIM(first_name)) as first_name,
    INITCAP(TRIM(last_name)) as last_name,
    CONCAT_WS(' ', first_name, last_name) as full_name,
    
    -- Email validation and standardization
    CASE 
      WHEN REGEXP_LIKE(LOWER(TRIM(email)), '^[^@]+@[^@]+\.[^@]+$') 
      THEN LOWER(TRIM(email))
      ELSE NULL 
    END as email,
    
    -- Phone standardization
    REGEXP_REPLACE(phone, '[^0-9]', '') as phone_cleaned,
    
    -- Address parsing
    PARSE_JSON(address_json) as address_parsed,
    
    -- Metadata
    source_system,
    extracted_timestamp
  FROM RAW.ALL_SOURCES.CUSTOMERS_UNION
)
SELECT * FROM cleaned_data
WHERE customer_id IS NOT NULL;
```

### Deduplication Logic
```sql
-- Integration Layer - Customer Deduplication
CREATE OR REPLACE VIEW INTEGRATION.INT_CUSTOMER_DEDUP AS
WITH ranked_customers AS (
  SELECT 
    *,
    ROW_NUMBER() OVER (
      PARTITION BY 
        COALESCE(email, phone_cleaned, customer_id)
      ORDER BY 
        CASE source_system
          WHEN 'CRM' THEN 1
          WHEN 'ERP' THEN 2
          ELSE 3
        END,
        extracted_timestamp DESC
    ) as priority_rank
  FROM STAGING.STG_CUSTOMERS
)
SELECT * FROM ranked_customers
WHERE priority_rank = 1;
```

### Business Logic Transformations
```sql
-- Customer Segmentation
CREATE OR REPLACE VIEW INTEGRATION.INT_CUSTOMER_SEGMENT AS
SELECT
  customer_id,
  CASE
    WHEN lifetime_value > 10000 THEN 'Platinum'
    WHEN lifetime_value > 5000 THEN 'Gold'
    WHEN lifetime_value > 1000 THEN 'Silver'
    ELSE 'Bronze'
  END as customer_segment,
  
  CASE
    WHEN last_activity_date > DATEADD(day, -30, CURRENT_DATE()) THEN 'Active'
    WHEN last_activity_date > DATEADD(day, -90, CURRENT_DATE()) THEN 'At Risk'
    ELSE 'Churned'
  END as customer_status
FROM INTEGRATION.INT_CUSTOMER_METRICS;
```

## Load Processes

### Slowly Changing Dimension (Type 2)
```sql
-- Load DIM_CUSTOMER with SCD Type 2
MERGE INTO PRESENTATION.DIM_CUSTOMER AS target
USING (
  SELECT * FROM INTEGRATION.INT_CUSTOMER_FINAL
) AS source
ON target.customer_id = source.customer_id
   AND target.is_current = TRUE

-- Update existing record
WHEN MATCHED 
  AND (target.email != source.email
       OR target.phone_primary != source.phone_primary
       OR target.customer_segment != source.customer_segment)
THEN UPDATE SET
  effective_to = CURRENT_TIMESTAMP(),
  is_current = FALSE,
  etl_updated_timestamp = CURRENT_TIMESTAMP()

-- Insert new version
WHEN NOT MATCHED THEN
INSERT (
  customer_id, first_name, last_name, email,
  phone_primary, customer_segment, customer_status,
  source_system, effective_from, effective_to,
  is_current, version_number
) VALUES (
  source.customer_id, source.first_name, source.last_name, source.email,
  source.phone_primary, source.customer_segment, source.customer_status,
  source.source_system, CURRENT_TIMESTAMP(), '9999-12-31'::TIMESTAMP,
  TRUE, 1
);
```

## Orchestration

### Airflow DAG Example
```python
from airflow import DAG
from airflow.operators.snowflake_operator import SnowflakeOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'customer_data_pipeline',
    default_args=default_args,
    schedule_interval='0 3 * * *',
    catchup=False
)

# Extract tasks
extract_crm = SnowflakeOperator(
    task_id='extract_crm_data',
    sql='CALL extract_crm_customers();',
    dag=dag
)

extract_erp = SnowflakeOperator(
    task_id='extract_erp_data',
    sql='CALL extract_erp_customers();',
    dag=dag
)

# Transform tasks
transform_staging = SnowflakeOperator(
    task_id='transform_to_staging',
    sql='CALL transform_customer_staging();',
    dag=dag
)

# Load tasks
load_dimensions = SnowflakeOperator(
    task_id='load_customer_dimensions',
    sql='CALL load_dim_customer();',
    dag=dag
)

# Dependencies
[extract_crm, extract_erp] >> transform_staging >> load_dimensions
```

## Error Handling and Recovery

### Error Logging
```sql
CREATE TABLE ETL_CONTROL.ERROR_LOG (
  error_id NUMBER IDENTITY,
  pipeline_name VARCHAR(100),
  task_name VARCHAR(100),
  error_timestamp TIMESTAMP_NTZ,
  error_message VARCHAR(4000),
  record_count NUMBER,
  severity VARCHAR(20)
);
```

### Recovery Procedures
1. **Automated Retry**: Built into orchestration
2. **Manual Intervention**: Documented procedures
3. **Rollback Strategy**: Point-in-time recovery
4. **Data Validation**: Post-load checks

## Performance Optimization

### Batch Processing
- Optimal batch sizes: 10,000 - 100,000 records
- Parallel processing for independent sources
- Incremental loading where possible

### Resource Management
```sql
-- Warehouse sizing
ALTER WAREHOUSE ETL_WH SET 
  WAREHOUSE_SIZE = 'LARGE'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3
  SCALING_POLICY = 'STANDARD';
```