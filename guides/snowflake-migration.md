# Snowflake Migration Guide

## Overview
This guide provides comprehensive instructions for migrating data platforms to Snowflake, including planning, execution, and post-migration optimization.

## Migration Planning

### Pre-Migration Assessment

#### Current State Analysis
```sql
-- Assess current data volumes
SELECT 
    database_name,
    schema_name,
    table_name,
    row_count,
    size_gb,
    last_updated
FROM information_schema.table_storage_metrics
ORDER BY size_gb DESC;

-- Identify complex queries for performance testing
SELECT 
    query_text,
    execution_time_ms,
    rows_produced,
    compilation_time_ms
FROM query_history
WHERE execution_time_ms > 60000  -- Queries taking more than 1 minute
ORDER BY execution_time_ms DESC
LIMIT 100;
```

#### Migration Scope Definition
| Component | Current Platform | Migration Priority | Complexity | Dependencies |
|-----------|------------------|-------------------|------------|--------------|
| Customer Data | Oracle | High | Medium | CRM, Marketing |
| Transaction Data | SQL Server | High | High | Finance, Reporting |
| Product Catalog | MySQL | Medium | Low | E-commerce |
| Log Data | Hadoop | Low | High | Analytics |

### Migration Strategy Options

#### 1. Big Bang Migration
**Pros:**
- Clean cutover
- Minimal dual maintenance
- Faster overall timeline

**Cons:**
- Higher risk
- Longer downtime
- Limited rollback options

**Best For:** Smaller datasets, less complex systems

#### 2. Phased Migration
**Pros:**
- Lower risk
- Gradual learning curve
- Easier rollback

**Cons:**
- Longer overall timeline
- Dual maintenance overhead
- Data synchronization complexity

**Best For:** Large enterprises, complex systems

#### 3. Parallel Run
**Pros:**
- Zero downtime
- Gradual confidence building
- Easy comparison

**Cons:**
- Highest cost
- Complex synchronization
- Extended timeline

**Best For:** Mission-critical systems

### Resource Planning

#### Team Requirements
| Role | Responsibilities | Skills Required | Time Commitment |
|------|-----------------|-----------------|-----------------|
| Migration Lead | Overall coordination | Project management, Snowflake expertise | 100% |
| Data Engineers | ETL development | SQL, Python, data modeling | 80% |
| Database Administrators | Schema migration | Database administration, performance tuning | 60% |
| Business Analysts | Requirements validation | Business knowledge, testing | 40% |
| DevOps Engineers | Infrastructure setup | Cloud platforms, automation | 30% |

#### Timeline Template
```
Phase 1: Planning (Weeks 1-4)
├── Current state assessment
├── Architecture design
├── Tool selection
└── Environment setup

Phase 2: Schema Migration (Weeks 5-8)
├── Schema conversion
├── Data type mapping
├── Constraint recreation
└── Security setup

Phase 3: Data Migration (Weeks 9-12)
├── Initial data load
├── Incremental sync setup
├── Data validation
└── Performance testing

Phase 4: Application Migration (Weeks 13-16)
├── Connection string updates
├── Query optimization
├── Application testing
└── User training

Phase 5: Go-Live (Weeks 17-18)
├── Final sync
├── Cutover execution
├── Monitoring setup
└── Post-migration validation
```

## Schema Migration

### Data Type Mapping

#### Common Type Conversions
| Source Type | Snowflake Type | Notes |
|-------------|----------------|-------|
| INT, INTEGER | NUMBER(38,0) | Snowflake uses NUMBER for all numeric types |
| BIGINT | NUMBER(38,0) | Same as above |
| DECIMAL(p,s) | NUMBER(p,s) | Direct mapping |
| FLOAT | FLOAT | Direct mapping |
| VARCHAR(n) | VARCHAR(n) | Direct mapping |
| CHAR(n) | CHAR(n) | Direct mapping |
| TEXT | VARCHAR | Use appropriate length |
| DATE | DATE | Direct mapping |
| DATETIME | TIMESTAMP_NTZ | Timezone handling needed |
| TIMESTAMP | TIMESTAMP_TZ | Consider timezone requirements |
| BLOB | BINARY | For binary data |
| CLOB | VARCHAR | May need size adjustment |

#### Advanced Type Mapping
```sql
-- Oracle to Snowflake mapping examples
-- Oracle: NUMBER without precision
NUMBER → NUMBER(38,0)

-- Oracle: TIMESTAMP WITH TIME ZONE
TIMESTAMP WITH TIME ZONE → TIMESTAMP_TZ

-- Oracle: INTERVAL data types
INTERVAL YEAR TO MONTH → VARCHAR (store as string, convert in queries)
INTERVAL DAY TO SECOND → VARCHAR (store as string, convert in queries)

-- SQL Server specific mappings
UNIQUEIDENTIFIER → VARCHAR(36)
MONEY → NUMBER(19,4)
SMALLMONEY → NUMBER(10,4)
HIERARCHYID → VARCHAR (store as string representation)
```

### Schema Conversion Process

#### 1. Automated Schema Extraction
```python
# schema_extractor.py
import pyodbc
import snowflake.connector

class SchemaExtractor:
    def __init__(self, source_conn_string, target_conn):
        self.source_conn = pyodbc.connect(source_conn_string)
        self.target_conn = target_conn
    
    def extract_table_definitions(self, schema_name):
        """Extract table definitions from source database"""
        
        query = """
        SELECT 
            table_name,
            column_name,
            data_type,
            character_maximum_length,
            numeric_precision,
            numeric_scale,
            is_nullable,
            column_default
        FROM information_schema.columns
        WHERE table_schema = ?
        ORDER BY table_name, ordinal_position
        """
        
        cursor = self.source_conn.cursor()
        cursor.execute(query, schema_name)
        
        tables = {}
        for row in cursor.fetchall():
            table_name = row.table_name
            if table_name not in tables:
                tables[table_name] = []
            
            tables[table_name].append({
                'column_name': row.column_name,
                'data_type': row.data_type,
                'length': row.character_maximum_length,
                'precision': row.numeric_precision,
                'scale': row.numeric_scale,
                'nullable': row.is_nullable == 'YES',
                'default': row.column_default
            })
        
        return tables
    
    def convert_to_snowflake_ddl(self, table_definitions):
        """Convert table definitions to Snowflake DDL"""
        
        ddl_statements = []
        
        for table_name, columns in table_definitions.items():
            ddl = f"CREATE TABLE {table_name} (\n"
            
            column_definitions = []
            for col in columns:
                sf_type = self.map_data_type(col)
                nullable = "" if col['nullable'] else " NOT NULL"
                default = f" DEFAULT {col['default']}" if col['default'] else ""
                
                col_def = f"    {col['column_name']} {sf_type}{nullable}{default}"
                column_definitions.append(col_def)
            
            ddl += ",\n".join(column_definitions)
            ddl += "\n);"
            
            ddl_statements.append(ddl)
        
        return ddl_statements
    
    def map_data_type(self, column_info):
        """Map source data types to Snowflake equivalents"""
        
        data_type = column_info['data_type'].upper()
        length = column_info['length']
        precision = column_info['precision']
        scale = column_info['scale']
        
        type_mapping = {
            'INT': 'NUMBER(38,0)',
            'INTEGER': 'NUMBER(38,0)',
            'BIGINT': 'NUMBER(38,0)',
            'SMALLINT': 'NUMBER(38,0)',
            'TINYINT': 'NUMBER(38,0)',
            'FLOAT': 'FLOAT',
            'REAL': 'FLOAT',
            'DOUBLE': 'FLOAT',
            'DATE': 'DATE',
            'TIME': 'TIME',
            'DATETIME': 'TIMESTAMP_NTZ',
            'DATETIME2': 'TIMESTAMP_NTZ',
            'SMALLDATETIME': 'TIMESTAMP_NTZ',
            'TEXT': 'VARCHAR',
            'NTEXT': 'VARCHAR',
            'IMAGE': 'BINARY',
            'BINARY': 'BINARY',
            'VARBINARY': 'BINARY'
        }
        
        if data_type in type_mapping:
            return type_mapping[data_type]
        elif data_type in ['VARCHAR', 'CHAR', 'NVARCHAR', 'NCHAR']:
            return f"{data_type}({length if length else 'MAX'})"
        elif data_type in ['DECIMAL', 'NUMERIC']:
            return f"NUMBER({precision},{scale})"
        else:
            return 'VARCHAR'  # Default fallback
```

#### 2. Index and Constraint Migration
```sql
-- Extract constraints from source system
-- Primary Keys
SELECT 
    table_name,
    column_name,
    constraint_name
FROM information_schema.key_column_usage
WHERE constraint_name IN (
    SELECT constraint_name 
    FROM information_schema.table_constraints 
    WHERE constraint_type = 'PRIMARY KEY'
);

-- Foreign Keys
SELECT 
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name,
    tc.constraint_name
FROM information_schema.table_constraints AS tc 
JOIN information_schema.key_column_usage AS kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
  ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';

-- Snowflake constraint creation (informational only)
-- Note: Snowflake constraints are for optimization, not enforcement
CREATE TABLE customer (
    customer_id NUMBER(38,0) PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    created_date TIMESTAMP_NTZ NOT NULL
);
```

## Data Migration

### Migration Tools Comparison

#### Snowflake Native Tools
| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| COPY INTO | Bulk loading from files | Fast, native, cost-effective | Requires staged files |
| Snowpipe | Real-time loading | Automated, event-driven | More complex setup |
| Replication | Database replication | Near real-time | Limited source support |

#### Third-Party Tools
| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| Fivetran | SaaS and database connectors | Managed, extensive connectors | Cost, less control |
| Talend | Complex transformations | Visual ETL, enterprise features | Licensing cost |
| AWS DMS | Database migration | AWS integration, change capture | AWS-specific |
| Azure Data Factory | Hybrid cloud scenarios | Microsoft integration | Azure-specific |

### Data Loading Strategies

#### 1. Bulk Loading with COPY INTO
```sql
-- Create file format
CREATE OR REPLACE FILE FORMAT csv_format
TYPE = 'CSV'
FIELD_DELIMITER = ','
SKIP_HEADER = 1
NULL_IF = ('NULL', 'null', '')
EMPTY_FIELD_AS_NULL = TRUE
FIELD_OPTIONALLY_ENCLOSED_BY = '"'
ESCAPE_CHARACTER = '\\'
DATE_FORMAT = 'YYYY-MM-DD'
TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS';

-- Create stage
CREATE OR REPLACE STAGE migration_stage
URL = 's3://migration-bucket/data/'
CREDENTIALS = (AWS_KEY_ID = 'your_key' AWS_SECRET_KEY = 'your_secret')
FILE_FORMAT = csv_format;

-- Load data
COPY INTO customers
FROM @migration_stage/customers/
PATTERN = '.*\.csv'
ON_ERROR = 'SKIP_FILE'
PURGE = FALSE;

-- Validate load
SELECT 
    file_name,
    status,
    rows_parsed,
    rows_loaded,
    error_count,
    first_error
FROM table(information_schema.copy_history(
    table_name => 'CUSTOMERS',
    start_time => dateadd(hours, -1, current_timestamp())
));
```

#### 2. Incremental Loading
```python
# incremental_loader.py
import snowflake.connector
import pandas as pd
from datetime import datetime, timedelta

class IncrementalLoader:
    def __init__(self, source_conn, target_conn):
        self.source_conn = source_conn
        self.target_conn = target_conn
    
    def get_max_timestamp(self, table_name, timestamp_column):
        """Get the maximum timestamp from target table"""
        
        query = f"SELECT MAX({timestamp_column}) FROM {table_name}"
        cursor = self.target_conn.cursor()
        cursor.execute(query)
        result = cursor.fetchone()
        
        return result[0] if result[0] else datetime(1900, 1, 1)
    
    def extract_incremental_data(self, table_name, timestamp_column, max_timestamp):
        """Extract incremental data from source"""
        
        query = f"""
        SELECT * FROM {table_name}
        WHERE {timestamp_column} > ?
        ORDER BY {timestamp_column}
        """
        
        return pd.read_sql(query, self.source_conn, params=[max_timestamp])
    
    def load_incremental_data(self, df, table_name):
        """Load incremental data to Snowflake"""
        
        if df.empty:
            print(f"No new data to load for {table_name}")
            return
        
        # Write to temporary staging table
        staging_table = f"{table_name}_staging"
        
        # Create staging table
        self.target_conn.cursor().execute(f"CREATE OR REPLACE TABLE {staging_table} LIKE {table_name}")
        
        # Use Snowflake Python connector to write data
        # This is a simplified example - use appropriate method for your setup
        success, nchunks, nrows, _ = write_pandas(
            self.target_conn,
            df,
            staging_table,
            auto_create_table=False
        )
        
        if success:
            # Merge staging data into main table
            merge_query = f"""
            MERGE INTO {table_name} AS target
            USING {staging_table} AS source
            ON target.id = source.id
            WHEN MATCHED THEN UPDATE SET *
            WHEN NOT MATCHED THEN INSERT *
            """
            
            self.target_conn.cursor().execute(merge_query)
            
            # Clean up staging table
            self.target_conn.cursor().execute(f"DROP TABLE {staging_table}")
            
            print(f"Loaded {nrows} rows into {table_name}")
        else:
            print(f"Failed to load data into {table_name}")
```

#### 3. Change Data Capture (CDC)
```python
# cdc_processor.py
import json
from datetime import datetime

class CDCProcessor:
    def __init__(self, snowflake_conn):
        self.conn = snowflake_conn
    
    def process_cdc_record(self, cdc_record):
        """Process a single CDC record"""
        
        operation = cdc_record.get('operation')  # INSERT, UPDATE, DELETE
        table_name = cdc_record.get('table_name')
        data = cdc_record.get('data')
        
        if operation == 'INSERT':
            self.handle_insert(table_name, data)
        elif operation == 'UPDATE':
            self.handle_update(table_name, data)
        elif operation == 'DELETE':
            self.handle_delete(table_name, data)
    
    def handle_insert(self, table_name, data):
        """Handle INSERT operation"""
        
        columns = ', '.join(data.keys())
        placeholders = ', '.join(['%s'] * len(data))
        values = list(data.values())
        
        query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        self.conn.cursor().execute(query, values)
    
    def handle_update(self, table_name, data):
        """Handle UPDATE operation"""
        
        # Assume 'id' is the primary key
        primary_key = data.pop('id')
        
        set_clause = ', '.join([f"{k} = %s" for k in data.keys()])
        values = list(data.values()) + [primary_key]
        
        query = f"UPDATE {table_name} SET {set_clause} WHERE id = %s"
        
        self.conn.cursor().execute(query, values)
    
    def handle_delete(self, table_name, data):
        """Handle DELETE operation"""
        
        # Use primary key for deletion
        primary_key = data.get('id')
        
        query = f"DELETE FROM {table_name} WHERE id = %s"
        
        self.conn.cursor().execute(query, [primary_key])
```

### Data Validation

#### Validation Strategies
```sql
-- Row count validation
SELECT 
    'source' AS system,
    COUNT(*) AS row_count
FROM source_system.customers
UNION ALL
SELECT 
    'target' AS system,
    COUNT(*) AS row_count
FROM snowflake.customers;

-- Data completeness validation
SELECT 
    'source' AS system,
    COUNT(*) AS total_rows,
    COUNT(customer_id) AS non_null_ids,
    COUNT(email) AS non_null_emails
FROM source_system.customers
UNION ALL
SELECT 
    'target' AS system,
    COUNT(*) AS total_rows,
    COUNT(customer_id) AS non_null_ids,
    COUNT(email) AS non_null_emails
FROM snowflake.customers;

-- Sample data comparison
WITH source_sample AS (
    SELECT * FROM source_system.customers 
    ORDER BY customer_id 
    LIMIT 1000
),
target_sample AS (
    SELECT * FROM snowflake.customers 
    ORDER BY customer_id 
    LIMIT 1000
)
SELECT 
    s.customer_id,
    s.email AS source_email,
    t.email AS target_email,
    CASE WHEN s.email = t.email THEN 'MATCH' ELSE 'MISMATCH' END AS status
FROM source_sample s
FULL OUTER JOIN target_sample t ON s.customer_id = t.customer_id;
```

## Performance Optimization

### Query Optimization Strategies

#### 1. Clustering Keys
```sql
-- Analyze query patterns
SELECT 
    query_text,
    execution_time,
    partitions_scanned,
    partitions_total
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%customers%'
ORDER BY execution_time DESC;

-- Set clustering key based on common filters
ALTER TABLE customers CLUSTER BY (region, created_date);

-- Monitor clustering effectiveness
SELECT 
    table_name,
    clustering_key,
    total_cluster_keys,
    average_depth,
    average_overlaps
FROM snowflake.information_schema.table_clustering_information
WHERE table_name = 'CUSTOMERS';
```

#### 2. Materialized Views
```sql
-- Create materialized view for common aggregations
CREATE MATERIALIZED VIEW customer_summary AS
SELECT 
    region,
    customer_segment,
    COUNT(*) AS customer_count,
    SUM(lifetime_value) AS total_ltv,
    AVG(lifetime_value) AS avg_ltv
FROM customers
GROUP BY region, customer_segment;

-- Monitor materialized view usage
SELECT 
    table_name,
    last_refresh_time,
    last_refresh_duration_ms,
    bytes_accessed,
    query_count
FROM snowflake.account_usage.materialized_view_refresh_history
WHERE table_name = 'CUSTOMER_SUMMARY';
```

#### 3. Search Optimization
```sql
-- Add search optimization for point lookups
ALTER TABLE customers ADD SEARCH OPTIMIZATION ON EQUALITY(customer_id, email);

-- Monitor search optimization effectiveness
SELECT 
    table_name,
    search_optimization_access_ratio,
    search_optimization_bytes_accessed
FROM snowflake.account_usage.search_optimization_history
WHERE table_name = 'CUSTOMERS';
```

### Warehouse Sizing and Management
```sql
-- Create appropriately sized warehouses
CREATE WAREHOUSE ETL_WH WITH
    WAREHOUSE_SIZE = 'LARGE'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3
    SCALING_POLICY = 'STANDARD'
    COMMENT = 'Warehouse for ETL operations';

CREATE WAREHOUSE ANALYTICS_WH WITH
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 2
    SCALING_POLICY = 'ECONOMY'
    COMMENT = 'Warehouse for analytics queries';

-- Monitor warehouse usage
SELECT 
    warehouse_name,
    avg_running,
    avg_queued_load,
    avg_queued_provisioning,
    avg_blocked
FROM snowflake.account_usage.warehouse_load_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP());
```

## Security Configuration

### Authentication and Authorization
```sql
-- Create roles
CREATE ROLE data_engineer;
CREATE ROLE data_analyst;
CREATE ROLE data_scientist;

-- Grant privileges
GRANT USAGE ON WAREHOUSE ETL_WH TO ROLE data_engineer;
GRANT USAGE ON DATABASE analytics TO ROLE data_engineer;
GRANT ALL ON SCHEMA analytics.staging TO ROLE data_engineer;

GRANT USAGE ON WAREHOUSE ANALYTICS_WH TO ROLE data_analyst;
GRANT USAGE ON DATABASE analytics TO ROLE data_analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.marts TO ROLE data_analyst;

-- Create users and assign roles
CREATE USER john_doe 
    PASSWORD = 'strong_password'
    DEFAULT_ROLE = data_engineer
    DEFAULT_WAREHOUSE = ETL_WH;

GRANT ROLE data_engineer TO USER john_doe;
```

### Data Masking and Row-Level Security
```sql
-- Create masking policy for PII
CREATE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
  CASE 
    WHEN CURRENT_ROLE() IN ('DATA_PRIVACY_OFFICER', 'ADMIN') THEN val
    WHEN CURRENT_ROLE() IN ('DATA_ANALYST') THEN 
      REGEXP_REPLACE(val, '.(?=.{4})', '*')
    ELSE '***MASKED***'
  END;

-- Apply masking policy
ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY email_mask;

-- Create row access policy
CREATE ROW ACCESS POLICY region_policy AS (region VARCHAR) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('ADMIN', 'GLOBAL_ANALYST')
  OR region = CURRENT_USER_REGION();

-- Apply row access policy
ALTER TABLE customers ADD ROW ACCESS POLICY region_policy ON (region);
```

## Post-Migration Activities

### Performance Monitoring
```python
# performance_monitor.py
import snowflake.connector
import pandas as pd
from datetime import datetime, timedelta

class PerformanceMonitor:
    def __init__(self, connection):
        self.conn = connection
    
    def get_warehouse_usage(self, days=7):
        """Get warehouse usage statistics"""
        
        query = f"""
        SELECT 
            warehouse_name,
            AVG(avg_running) as avg_running_clusters,
            SUM(credits_used) as total_credits,
            COUNT(*) as query_count
        FROM snowflake.account_usage.warehouse_metering_history
        WHERE start_time >= DATEADD(day, -{days}, CURRENT_TIMESTAMP())
        GROUP BY warehouse_name
        ORDER BY total_credits DESC
        """
        
        return pd.read_sql(query, self.conn)
    
    def get_expensive_queries(self, limit=10):
        """Get most expensive queries by credits consumed"""
        
        query = f"""
        SELECT 
            query_id,
            query_text,
            warehouse_name,
            total_elapsed_time,
            credits_used_cloud_services,
            rows_produced
        FROM snowflake.account_usage.query_history
        WHERE start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP())
          AND credits_used_cloud_services > 0
        ORDER BY credits_used_cloud_services DESC
        LIMIT {limit}
        """
        
        return pd.read_sql(query, self.conn)
    
    def get_table_storage_usage(self):
        """Get table storage usage statistics"""
        
        query = """
        SELECT 
            table_catalog,
            table_schema,
            table_name,
            active_bytes / (1024*1024*1024) as active_gb,
            time_travel_bytes / (1024*1024*1024) as time_travel_gb,
            failsafe_bytes / (1024*1024*1024) as failsafe_gb
        FROM snowflake.account_usage.table_storage_metrics
        ORDER BY active_bytes DESC
        """
        
        return pd.read_sql(query, self.conn)
```

### Cost Optimization
```sql
-- Identify unused tables
SELECT 
    table_name,
    last_altered,
    row_count,
    bytes / (1024*1024*1024) as size_gb
FROM snowflake.account_usage.tables
WHERE last_altered < DATEADD(month, -6, CURRENT_TIMESTAMP())
  AND row_count > 0
ORDER BY bytes DESC;

-- Find tables with high time travel usage
SELECT 
    table_name,
    active_bytes / (1024*1024*1024) as active_gb,
    time_travel_bytes / (1024*1024*1024) as time_travel_gb,
    (time_travel_bytes / active_bytes) * 100 as time_travel_percentage
FROM snowflake.account_usage.table_storage_metrics
WHERE time_travel_bytes > active_bytes
ORDER BY time_travel_percentage DESC;

-- Optimize time travel settings for large tables
ALTER TABLE large_historical_table SET DATA_RETENTION_TIME_IN_DAYS = 1;

-- Set up automatic clustering
ALTER TABLE frequently_queried_table RESUME RECLUSTER;
```

### Backup and Disaster Recovery
```sql
-- Set up database replication for disaster recovery
-- Primary account setup
CREATE DATABASE production_replica_db;

-- Enable replication
ALTER DATABASE production_db ENABLE REPLICATION TO ACCOUNTS ('target_account');

-- Secondary account setup (run on target account)
CREATE DATABASE production_db AS REPLICA OF source_account.production_db;

-- Monitor replication lag
SELECT 
    database_name,
    replication_group_id,
    phase,
    start_time,
    end_time,
    credits_used
FROM snowflake.account_usage.replication_usage_history
WHERE start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP());
```

## Troubleshooting Common Issues

### Performance Issues
```sql
-- Identify poorly performing queries
SELECT 
    query_id,
    query_text,
    user_name,
    warehouse_name,
    database_name,
    schema_name,
    execution_status,
    error_code,
    error_message,
    total_elapsed_time / 1000 as duration_seconds,
    queued_provisioning_time,
    queued_repair_time,
    queued_overload_time,
    compilation_time,
    execution_time,
    bytes_scanned,
    rows_produced
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND total_elapsed_time > 300000  -- More than 5 minutes
ORDER BY total_elapsed_time DESC;

-- Check for clustering effectiveness
SELECT 
    table_name,
    clustering_key,
    total_cluster_keys,
    average_depth,
    average_overlaps,
    overlap_percent
FROM snowflake.information_schema.table_clustering_information
WHERE average_depth > 5  -- Indicates poor clustering
ORDER BY average_depth DESC;
```

### Data Loading Issues
```sql
-- Check COPY command history
SELECT 
    table_name,
    file_name,
    status,
    rows_parsed,
    rows_loaded,
    error_count,
    first_error,
    first_error_line_number
FROM table(information_schema.copy_history(
    table_name => 'YOUR_TABLE',
    start_time => dateadd(hours, -24, current_timestamp())
))
WHERE status != 'LOADED'
ORDER BY last_load_time DESC;

-- Validate data after loading
SELECT 
    COUNT(*) as total_rows,
    COUNT(DISTINCT customer_id) as unique_customers,
    COUNT(*) - COUNT(customer_id) as null_customer_ids,
    MIN(created_date) as oldest_record,
    MAX(created_date) as newest_record
FROM customers;
```

### Connection and Authentication Issues
```python
# connection_troubleshooter.py
import snowflake.connector
from snowflake.connector import DictCursor

def test_connection(account, user, password, warehouse=None, database=None):
    """Test Snowflake connection with detailed error reporting"""
    
    try:
        conn = snowflake.connector.connect(
            account=account,
            user=user,
            password=password,
            warehouse=warehouse,
            database=database
        )
        
        cursor = conn.cursor(DictCursor)
        cursor.execute("SELECT CURRENT_VERSION()")
        result = cursor.fetchone()
        
        print(f"✅ Connection successful!")
        print(f"Snowflake version: {result['CURRENT_VERSION()']}")
        
        # Test warehouse access
        if warehouse:
            cursor.execute(f"USE WAREHOUSE {warehouse}")
            print(f"✅ Warehouse {warehouse} accessible")
        
        # Test database access
        if database:
            cursor.execute(f"USE DATABASE {database}")
            print(f"✅ Database {database} accessible")
        
        conn.close()
        return True
        
    except snowflake.connector.errors.ProgrammingError as e:
        print(f"❌ Programming error: {e}")
        return False
    except snowflake.connector.errors.DatabaseError as e:
        print(f"❌ Database error: {e}")
        return False
    except Exception as e:
        print(f"❌ Unexpected error: {e}")
        return False

def diagnose_role_permissions(connection, role_name):
    """Diagnose role permissions"""
    
    cursor = connection.cursor(DictCursor)
    
    # Check role existence
    cursor.execute(f"SHOW ROLES LIKE '{role_name}'")
    roles = cursor.fetchall()
    
    if not roles:
        print(f"❌ Role {role_name} does not exist")
        return
    
    print(f"✅ Role {role_name} exists")
    
    # Check role grants
    cursor.execute(f"SHOW GRANTS TO ROLE {role_name}")
    grants = cursor.fetchall()
    
    print("Role permissions:")
    for grant in grants:
        print(f"  - {grant['privilege']} on {grant['granted_on']} {grant['name']}")
```

## Migration Checklist

### Pre-Migration
- [ ] Current state assessment completed
- [ ] Migration strategy selected and approved
- [ ] Target architecture designed
- [ ] Team roles and responsibilities defined
- [ ] Timeline and milestones established
- [ ] Budget approved
- [ ] Snowflake account provisioned
- [ ] Security requirements documented
- [ ] Data mapping completed
- [ ] Migration tools selected
- [ ] Test environment set up

### Schema Migration
- [ ] Source schema documented
- [ ] Data type mapping validated
- [ ] DDL scripts generated
- [ ] Constraints identified and converted
- [ ] Indexes analyzed for clustering keys
- [ ] Views converted
- [ ] Stored procedures assessed
- [ ] Security policies designed
- [ ] Target schema created in Snowflake
- [ ] Schema validation completed

### Data Migration
- [ ] Migration approach finalized
- [ ] Data extraction process tested
- [ ] File staging configured
- [ ] COPY commands prepared
- [ ] Incremental loading strategy implemented
- [ ] Data validation queries prepared
- [ ] Error handling procedures defined
- [ ] Initial data load completed
- [ ] Data validation passed
- [ ] Performance baseline established

### Application Migration
- [ ] Connection strings updated
- [ ] Queries analyzed and optimized
- [ ] Application code updated
- [ ] Integration testing completed
- [ ] Performance testing passed
- [ ] User acceptance testing completed
- [ ] Documentation updated
- [ ] Training materials prepared
- [ ] Rollback procedures tested
- [ ] Go-live procedures documented

### Post-Migration
- [ ] Production cutover completed
- [ ] Data synchronization verified
- [ ] Performance monitoring active
- [ ] Cost monitoring configured
- [ ] Security validation completed
- [ ] User training conducted
- [ ] Documentation finalized
- [ ] Lessons learned documented
- [ ] Project retrospective completed
- [ ] Ongoing support plan activated

---
**Document Control**
- Version: 1.0
- Created: [Date]
- Last Updated: [Date]
- Author: Data Platform Team
- Reviewers: [Names]
- Next Review: [Date]