# Database Schema Template

## Schema Overview

### Basic Information
| Property | Value |
|----------|-------|
| Schema Name | [schema_name] |
| Database | [database_name] |
| Purpose | [Primary business purpose] |
| Data Domain | [Customer/Product/Financial/Operational] |
| Owner | [Team/Individual] |
| Created Date | [Date] |
| Last Modified | [Date] |
| Version | [Version number] |

### Business Context
- **Purpose**: [Why this schema exists]
- **Scope**: [What business processes it supports]
- **Stakeholders**: [Who uses this schema]
- **Data Sources**: [Where the data comes from]

## Schema Architecture

### Database Structure
```
DATABASE: [database_name]
├── SCHEMA: [schema_name]
│   ├── TABLES
│   │   ├── [table_1]
│   │   ├── [table_2]
│   │   └── [table_n]
│   ├── VIEWS
│   │   ├── [view_1]
│   │   └── [view_2]
│   ├── PROCEDURES
│   │   ├── [proc_1]
│   │   └── [proc_2]
│   └── FUNCTIONS
│       ├── [func_1]
│       └── [func_2]
```

### Entity Relationship Diagram
```
[Entity1] ||--o{ [Entity2] : relationship_name
[Entity2] ||--o{ [Entity3] : relationship_name
[Entity1] ||--|| [Entity4] : relationship_name
```

## Table Definitions

### Table 1: [table_name]
**Purpose**: [What this table stores]

```sql
CREATE TABLE [schema_name].[table_name] (
    -- Primary Key
    [table_name]_id NUMBER IDENTITY(1,1) PRIMARY KEY,
    
    -- Business Keys
    [business_key] VARCHAR(50) NOT NULL UNIQUE,
    
    -- Attributes
    [attribute_1] VARCHAR(100) NOT NULL,
    [attribute_2] VARCHAR(255),
    [attribute_3] NUMBER(18,2),
    [attribute_4] DATE,
    [attribute_5] BOOLEAN DEFAULT FALSE,
    
    -- Foreign Keys
    [parent_table]_id NUMBER REFERENCES [parent_table]([parent_table]_id),
    
    -- Metadata
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    created_by VARCHAR(100) DEFAULT USER,
    updated_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    updated_by VARCHAR(100) DEFAULT USER,
    
    -- Soft Delete
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_date TIMESTAMP_NTZ,
    deleted_by VARCHAR(100),
    
    -- Version Control
    version_number NUMBER DEFAULT 1,
    
    -- Audit Trail
    etl_batch_id VARCHAR(50),
    source_system VARCHAR(50),
    data_hash VARCHAR(64)
);
```

### Column Specifications
| Column Name | Data Type | Length | Nullable | Default | Description | Business Rules |
|-------------|-----------|--------|----------|---------|-------------|----------------|
| [column_1] | [type] | [length] | [Y/N] | [default] | [Description] | [Rules] |
| [column_2] | [type] | [length] | [Y/N] | [default] | [Description] | [Rules] |
| [column_3] | [type] | [length] | [Y/N] | [default] | [Description] | [Rules] |

### Indexes
```sql
-- Primary Index (Clustered)
CREATE CLUSTERED INDEX IX_[table_name]_PK 
ON [table_name] ([table_name]_id);

-- Business Key Index
CREATE UNIQUE NONCLUSTERED INDEX IX_[table_name]_business_key 
ON [table_name] ([business_key]);

-- Foreign Key Index
CREATE NONCLUSTERED INDEX IX_[table_name]_FK_parent 
ON [table_name] ([parent_table]_id);

-- Search/Filter Indexes
CREATE NONCLUSTERED INDEX IX_[table_name]_search 
ON [table_name] ([attribute_1], [attribute_2]) 
INCLUDE ([attribute_3], [attribute_4]);
```

### Constraints
```sql
-- Check Constraints
ALTER TABLE [table_name] 
ADD CONSTRAINT CHK_[table_name]_attribute_3 
CHECK ([attribute_3] >= 0);

ALTER TABLE [table_name] 
ADD CONSTRAINT CHK_[table_name]_dates 
CHECK (updated_date >= created_date);

-- Foreign Key Constraints
ALTER TABLE [table_name] 
ADD CONSTRAINT FK_[table_name]_parent 
FOREIGN KEY ([parent_table]_id) 
REFERENCES [parent_table]([parent_table]_id);
```

## Views and Abstractions

### Business View: [view_name]
**Purpose**: [Business purpose of this view]

```sql
CREATE OR REPLACE VIEW [schema_name].[view_name] AS
SELECT 
    t1.[business_key],
    t1.[attribute_1] as [business_friendly_name_1],
    t1.[attribute_2] as [business_friendly_name_2],
    t2.[related_attribute],
    CASE 
        WHEN t1.[attribute_5] = TRUE THEN 'Active'
        ELSE 'Inactive'
    END as status_description,
    t1.created_date,
    t1.updated_date
FROM [table_name] t1
LEFT JOIN [related_table] t2 ON t1.[parent_table]_id = t2.[parent_table]_id
WHERE t1.is_deleted = FALSE;
```

### Materialized View: [materialized_view_name]
**Purpose**: [Performance optimization purpose]

```sql
CREATE MATERIALIZED VIEW [schema_name].[materialized_view_name] AS
SELECT 
    [dimension_column],
    COUNT(*) as record_count,
    SUM([measure_column]) as total_amount,
    AVG([measure_column]) as average_amount,
    MAX(updated_date) as last_updated
FROM [table_name]
WHERE is_deleted = FALSE
GROUP BY [dimension_column];

-- Refresh schedule
CREATE OR REPLACE TASK refresh_[materialized_view_name]
WAREHOUSE = COMPUTE_WH
SCHEDULE = 'USING CRON 0 6 * * *'
AS
ALTER MATERIALIZED VIEW [materialized_view_name] REFRESH;
```

## Stored Procedures and Functions

### Data Maintenance Procedure
```sql
CREATE OR REPLACE PROCEDURE [schema_name].sp_maintain_[table_name]()
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    processed_count INTEGER;
BEGIN
    -- Soft delete old records
    UPDATE [table_name]
    SET is_deleted = TRUE,
        deleted_date = CURRENT_TIMESTAMP(),
        deleted_by = 'SYSTEM_MAINTENANCE'
    WHERE [date_column] < DATEADD(year, -7, CURRENT_DATE())
      AND is_deleted = FALSE;
    
    processed_count := ROW_COUNT;
    
    -- Update statistics
    CALL SYSTEM$ANALYZE_TABLE('[schema_name].[table_name]');
    
    RETURN 'Processed ' || processed_count || ' records';
END;
$$;
```

### Business Logic Function
```sql
CREATE OR REPLACE FUNCTION [schema_name].fn_calculate_[business_metric](
    input_value NUMBER,
    factor NUMBER DEFAULT 1.0
)
RETURNS NUMBER
LANGUAGE SQL
AS
$$
    SELECT 
        CASE 
            WHEN input_value IS NULL THEN 0
            WHEN input_value < 0 THEN 0
            ELSE input_value * factor
        END
$$;
```

## Data Standards and Conventions

### Naming Conventions
| Object Type | Convention | Example |
|-------------|------------|---------|
| Tables | [domain]_[entity] | customer_profiles |
| Views | v_[descriptive_name] | v_active_customers |
| Indexes | IX_[table]_[columns] | IX_customers_email |
| Constraints | [type]_[table]_[column] | CHK_customers_age |
| Procedures | sp_[action]_[entity] | sp_update_customer |
| Functions | fn_[purpose] | fn_calculate_discount |

### Data Types Standards
| Logical Type | Physical Type | Notes |
|--------------|---------------|-------|
| Identifier | NUMBER IDENTITY | Auto-incrementing primary keys |
| Business Key | VARCHAR(50) | Natural business identifiers |
| Name/Title | VARCHAR(100) | Person/entity names |
| Description | VARCHAR(4000) | Long text descriptions |
| Code | VARCHAR(20) | Status codes, categories |
| Amount | NUMBER(18,2) | Monetary values |
| Quantity | NUMBER(10,3) | Quantities with decimals |
| Percentage | NUMBER(5,4) | Percentages (0.0000 to 1.0000) |
| Date | DATE | Date only |
| Timestamp | TIMESTAMP_NTZ | Date and time |
| Flag | BOOLEAN | True/false values |

### Column Standards
```sql
-- Standard audit columns for every table
created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
created_by VARCHAR(100) DEFAULT USER,
updated_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
updated_by VARCHAR(100) DEFAULT USER,

-- Standard soft delete columns
is_deleted BOOLEAN DEFAULT FALSE,
deleted_date TIMESTAMP_NTZ,
deleted_by VARCHAR(100),

-- Standard versioning
version_number NUMBER DEFAULT 1,

-- Standard lineage tracking
etl_batch_id VARCHAR(50),
source_system VARCHAR(50)
```

## Security and Access Control

### Row Level Security
```sql
-- Create row access policy
CREATE ROW ACCESS POLICY [schema_name].[table_name]_policy
AS (region VARCHAR) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('ADMIN', 'GLOBAL_VIEWER')
  OR region = CURRENT_USER_REGION();

-- Apply policy to table
ALTER TABLE [schema_name].[table_name] 
ADD ROW ACCESS POLICY [schema_name].[table_name]_policy ON (region_column);
```

### Column Level Security
```sql
-- Create masking policy for PII
CREATE MASKING POLICY [schema_name].mask_pii AS (val STRING) 
RETURNS STRING ->
  CASE 
    WHEN CURRENT_ROLE() IN ('ADMIN', 'PII_VIEWER') THEN val
    WHEN CURRENT_ROLE() IN ('ANALYST') THEN 
      REGEXP_REPLACE(val, '.(?=.{4})', '*')
    ELSE '***MASKED***'
  END;

-- Apply masking policy
ALTER TABLE [schema_name].[table_name] 
MODIFY COLUMN [sensitive_column] 
SET MASKING POLICY [schema_name].mask_pii;
```

### Access Permissions
| Role | Permissions | Tables/Views |
|------|-------------|--------------|
| [role_1] | SELECT | All views |
| [role_2] | SELECT, INSERT, UPDATE | Operational tables |
| [role_3] | ALL | All objects |

```sql
-- Grant permissions
GRANT SELECT ON ALL VIEWS IN SCHEMA [schema_name] TO ROLE [read_role];
GRANT SELECT, INSERT, UPDATE ON [schema_name].[table_name] TO ROLE [write_role];
GRANT ALL ON SCHEMA [schema_name] TO ROLE [admin_role];
```

## Data Quality and Validation

### Data Quality Rules
```sql
-- Completeness checks
SELECT 
    'Completeness' as check_type,
    COUNT(*) as total_records,
    COUNT([required_column]) as complete_records,
    (COUNT([required_column]) / COUNT(*)) * 100 as completeness_pct
FROM [table_name];

-- Validity checks
SELECT 
    'Validity' as check_type,
    COUNT(*) as total_records,
    COUNT(CASE WHEN [email_column] LIKE '%@%.%' THEN 1 END) as valid_emails,
    (COUNT(CASE WHEN [email_column] LIKE '%@%.%' THEN 1 END) / COUNT(*)) * 100 as validity_pct
FROM [table_name]
WHERE [email_column] IS NOT NULL;

-- Uniqueness checks
SELECT 
    'Uniqueness' as check_type,
    COUNT(*) as total_records,
    COUNT(DISTINCT [business_key]) as unique_keys,
    CASE WHEN COUNT(*) = COUNT(DISTINCT [business_key]) 
         THEN 'PASS' ELSE 'FAIL' END as uniqueness_check
FROM [table_name];
```

### Validation Procedures
```sql
CREATE OR REPLACE PROCEDURE [schema_name].sp_validate_data_quality()
RETURNS TABLE (
    table_name STRING,
    check_type STRING,
    status STRING,
    message STRING,
    check_timestamp TIMESTAMP_NTZ
)
LANGUAGE SQL
AS
$$
DECLARE
    result_cursor CURSOR FOR
        -- Data quality validation queries
        SELECT * FROM data_quality_checks;
BEGIN
    FOR record IN result_cursor DO
        -- Process each validation result
        INSERT INTO data_quality_log VALUES (record.*);
    END FOR;
    
    RETURN TABLE(SELECT * FROM data_quality_log WHERE check_date = CURRENT_DATE());
END;
$$;
```

## Performance Optimization

### Partitioning Strategy
```sql
-- Time-based partitioning
CREATE TABLE [schema_name].[large_table] (
    [columns...]
) PARTITION BY (DATE_TRUNC('MONTH', created_date));

-- Key-based partitioning
CREATE TABLE [schema_name].[distributed_table] (
    [columns...]
) PARTITION BY ([partition_key]);
```

### Clustering Keys
```sql
-- Set clustering key for performance
ALTER TABLE [schema_name].[table_name] 
CLUSTER BY ([frequently_filtered_column], [join_column]);
```

### Query Optimization
```sql
-- Optimized query patterns
-- Use clustering key in WHERE clause
SELECT * FROM [table_name] 
WHERE [clustering_column] = 'value'
  AND created_date BETWEEN '2024-01-01' AND '2024-12-31';

-- Use projection to reduce data transfer
SELECT [needed_columns] FROM [table_name]
WHERE [filter_conditions];

-- Use appropriate JOIN types
SELECT t1.[columns], t2.[columns]
FROM [table_1] t1
INNER JOIN [table_2] t2 ON t1.[key] = t2.[key]
WHERE [filter_conditions];
```

## Maintenance and Operations

### Regular Maintenance Tasks
```sql
-- Daily: Update table statistics
CALL SYSTEM$ANALYZE_TABLE('[schema_name].[table_name]');

-- Weekly: Rebuild clustered indexes
ALTER TABLE [schema_name].[table_name] RECLUSTER;

-- Monthly: Archive old data
CALL [schema_name].sp_archive_old_data();
```

### Monitoring Queries
```sql
-- Table size monitoring
SELECT 
    table_name,
    row_count,
    bytes / (1024*1024*1024) as size_gb,
    last_altered
FROM information_schema.tables
WHERE table_schema = '[SCHEMA_NAME]'
ORDER BY bytes DESC;

-- Query performance monitoring
SELECT 
    query_text,
    total_elapsed_time / 1000 as duration_seconds,
    rows_produced,
    bytes_scanned / (1024*1024*1024) as gb_scanned
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%[table_name]%'
  AND start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC;
```

## Documentation and Metadata

### Data Dictionary
| Table | Column | Type | Description | Source | Business Rules |
|-------|--------|------|-------------|--------|----------------|
| [table] | [column] | [type] | [description] | [source] | [rules] |

### Change Log
| Version | Date | Changes | Author | Impact |
|---------|------|---------|--------|--------|
| 1.0 | [Date] | Initial schema creation | [Name] | New |
| 1.1 | [Date] | Added [table/column] | [Name] | Low |
| 2.0 | [Date] | Breaking change: [description] | [Name] | High |

### Dependencies
- **Upstream Dependencies**: [Systems/schemas that this depends on]
- **Downstream Dependencies**: [Systems/schemas that depend on this]
- **External Dependencies**: [Third-party systems/APIs]

---
**Document Control**
- Version: 1.0
- Created: [Date]
- Last Updated: [Date]
- Owner: [Name/Team]
- Next Review: [Date]
- Classification: [Internal/Confidential]