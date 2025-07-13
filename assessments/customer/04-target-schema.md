# Customer Data Target Schema Design

## Overview
This document defines the target schema for unified customer data in Snowflake.

## Schema Architecture

### Database Structure
```
CUSTOMER_DB
├── RAW
│   ├── SRC_SYSTEM_1
│   ├── SRC_SYSTEM_2
│   └── SRC_SYSTEM_3
├── STAGING
│   ├── STG_CUSTOMERS
│   └── STG_ADDRESSES
├── INTEGRATION
│   ├── INT_CUSTOMER_DEDUP
│   └── INT_ADDRESS_STANDARD
└── PRESENTATION
    ├── DIM_CUSTOMER
    ├── DIM_ADDRESS
    └── FACT_CUSTOMER_ACTIVITY
```

## Table Definitions

### DIM_CUSTOMER (Type 2 SCD)
```sql
CREATE TABLE PRESENTATION.DIM_CUSTOMER (
    -- Surrogate Keys
    customer_key NUMBER IDENTITY PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    
    -- Attributes
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    full_name VARCHAR(200),
    email VARCHAR(255),
    phone_primary VARCHAR(50),
    phone_secondary VARCHAR(50),
    
    -- Demographics
    date_of_birth DATE,
    gender VARCHAR(20),
    language_preference VARCHAR(10),
    
    -- Status
    customer_status VARCHAR(50),
    customer_segment VARCHAR(50),
    lifetime_value NUMBER(18,2),
    
    -- Metadata
    source_system VARCHAR(50),
    created_date TIMESTAMP_NTZ,
    updated_date TIMESTAMP_NTZ,
    
    -- SCD Type 2 Fields
    effective_from TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    effective_to TIMESTAMP_NTZ DEFAULT '9999-12-31',
    is_current BOOLEAN DEFAULT TRUE,
    version_number NUMBER DEFAULT 1,
    
    -- Audit
    etl_inserted_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    etl_updated_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### DIM_ADDRESS
```sql
CREATE TABLE PRESENTATION.DIM_ADDRESS (
    -- Keys
    address_key NUMBER IDENTITY PRIMARY KEY,
    customer_key NUMBER REFERENCES DIM_CUSTOMER(customer_key),
    
    -- Address Fields
    address_type VARCHAR(50), -- billing, shipping, etc.
    address_line_1 VARCHAR(255),
    address_line_2 VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code VARCHAR(2),
    country_name VARCHAR(100),
    
    -- Geocoding
    latitude NUMBER(10,8),
    longitude NUMBER(11,8),
    
    -- Status
    is_primary BOOLEAN,
    is_validated BOOLEAN,
    validation_date DATE,
    
    -- Metadata
    effective_from TIMESTAMP_NTZ,
    effective_to TIMESTAMP_NTZ,
    is_current BOOLEAN
);
```

### FACT_CUSTOMER_ACTIVITY
```sql
CREATE TABLE PRESENTATION.FACT_CUSTOMER_ACTIVITY (
    -- Keys
    activity_key NUMBER IDENTITY PRIMARY KEY,
    customer_key NUMBER REFERENCES DIM_CUSTOMER(customer_key),
    date_key NUMBER,
    
    -- Metrics
    login_count NUMBER,
    page_views NUMBER,
    transaction_count NUMBER,
    transaction_amount NUMBER(18,2),
    support_tickets NUMBER,
    
    -- Dimensions
    channel VARCHAR(50),
    device_type VARCHAR(50),
    
    -- Audit
    etl_batch_id VARCHAR(50),
    etl_inserted_timestamp TIMESTAMP_NTZ
);
```

## Data Types and Standards

### Naming Conventions
- Tables: UPPER_SNAKE_CASE
- Columns: lower_snake_case
- Primary Keys: [table]_key
- Foreign Keys: [referenced_table]_key

### Data Type Standards
| Logical Type | Snowflake Type | Example |
|--------------|----------------|---------|
| ID/Key | VARCHAR(50) | customer_id |
| Name | VARCHAR(100) | first_name |
| Description | VARCHAR(4000) | notes |
| Amount | NUMBER(18,2) | transaction_amount |
| Count | NUMBER | login_count |
| Date | DATE | birth_date |
| Timestamp | TIMESTAMP_NTZ | created_timestamp |
| Boolean | BOOLEAN | is_active |

## Indexing and Performance

### Clustering Keys
```sql
ALTER TABLE DIM_CUSTOMER CLUSTER BY (customer_id, is_current);
ALTER TABLE FACT_CUSTOMER_ACTIVITY CLUSTER BY (date_key, customer_key);
```

### Search Optimization
```sql
ALTER TABLE DIM_CUSTOMER ADD SEARCH OPTIMIZATION ON EQUALITY(email, phone_primary);
```

## Security and Governance

### Row Access Policies
```sql
CREATE ROW ACCESS POLICY customer_region_policy
AS (region VARCHAR) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('ADMIN', 'GLOBAL_ANALYST')
  OR region = CURRENT_REGION();
```

### Column Masking
```sql
CREATE MASKING POLICY pii_email_mask AS (val STRING) 
RETURNS STRING ->
  CASE 
    WHEN CURRENT_ROLE() IN ('ADMIN', 'ANALYST') THEN val
    ELSE REGEXP_REPLACE(val, '.+@', '***@')
  END;
```

## Migration Mapping
| Source System | Source Table | Target Table | Transformation |
|---------------|--------------|--------------|----------------|
| CRM | customers | DIM_CUSTOMER | Standardize names |
| ERP | cust_master | DIM_CUSTOMER | Merge duplicates |
| Legacy | addresses | DIM_ADDRESS | Geocode, validate |