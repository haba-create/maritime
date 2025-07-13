# Databricks Sunset Guide

## Overview
This guide provides comprehensive instructions for migrating away from Databricks, including workload assessment, migration strategies, and platform alternatives.

## Sunset Planning

### Business Drivers for Migration
Common reasons for Databricks sunset:
- **Cost Optimization**: Reducing cloud compute costs
- **Platform Consolidation**: Standardizing on fewer platforms
- **Skill Set Alignment**: Moving to more familiar technologies
- **Vendor Strategy**: Reducing vendor lock-in
- **Performance Issues**: Addressing latency or reliability concerns
- **Compliance Requirements**: Meeting new regulatory needs

### Impact Assessment

#### Current Usage Analysis
```python
# databricks_usage_analyzer.py
import requests
import pandas as pd
from datetime import datetime, timedelta

class DatabricksUsageAnalyzer:
    def __init__(self, instance_url, access_token):
        self.instance_url = instance_url
        self.headers = {"Authorization": f"Bearer {access_token}"}
    
    def get_cluster_usage(self, days=30):
        """Analyze cluster usage patterns"""
        
        endpoint = f"{self.instance_url}/api/2.0/clusters/list"
        response = requests.get(endpoint, headers=self.headers)
        clusters = response.json().get('clusters', [])
        
        usage_data = []
        for cluster in clusters:
            cluster_id = cluster['cluster_id']
            cluster_name = cluster['cluster_name']
            
            # Get cluster events
            events_endpoint = f"{self.instance_url}/api/2.0/clusters/events"
            events_response = requests.post(
                events_endpoint,
                headers=self.headers,
                json={
                    "cluster_id": cluster_id,
                    "start_time": int((datetime.now() - timedelta(days=days)).timestamp() * 1000),
                    "end_time": int(datetime.now().timestamp() * 1000),
                    "limit": 500
                }
            )
            
            events = events_response.json().get('events', [])
            
            # Calculate usage metrics
            running_time = 0
            start_events = [e for e in events if e['type'] == 'RUNNING']
            terminating_events = [e for e in events if e['type'] in ['TERMINATING', 'TERMINATED']]
            
            for start_event in start_events:
                start_time = start_event['timestamp']
                # Find corresponding termination
                corresponding_termination = None
                for term_event in terminating_events:
                    if term_event['timestamp'] > start_time:
                        corresponding_termination = term_event
                        break
                
                if corresponding_termination:
                    running_time += (corresponding_termination['timestamp'] - start_time) / 1000 / 3600  # hours
            
            usage_data.append({
                'cluster_id': cluster_id,
                'cluster_name': cluster_name,
                'node_type': cluster.get('node_type_id', 'unknown'),
                'driver_node_type': cluster.get('driver_node_type_id', 'unknown'),
                'num_workers': cluster.get('num_workers', 0),
                'running_hours': running_time,
                'estimated_cost': self.estimate_cluster_cost(cluster, running_time)
            })
        
        return pd.DataFrame(usage_data)
    
    def get_job_analysis(self):
        """Analyze job definitions and dependencies"""
        
        endpoint = f"{self.instance_url}/api/2.1/jobs/list"
        response = requests.get(endpoint, headers=self.headers)
        jobs = response.json().get('jobs', [])
        
        job_analysis = []
        for job in jobs:
            job_id = job['job_id']
            settings = job.get('settings', {})
            
            # Analyze job complexity
            notebook_task = settings.get('notebook_task', {})
            spark_submit_task = settings.get('spark_submit_task', {})
            python_wheel_task = settings.get('python_wheel_task', {})
            
            complexity_score = self.calculate_job_complexity(settings)
            
            job_analysis.append({
                'job_id': job_id,
                'job_name': settings.get('name', 'Unknown'),
                'task_type': self.get_primary_task_type(settings),
                'notebook_path': notebook_task.get('notebook_path', ''),
                'schedule': settings.get('schedule', {}).get('quartz_cron_expression', 'Manual'),
                'cluster_spec': settings.get('new_cluster', {}),
                'complexity_score': complexity_score,
                'migration_priority': self.assess_migration_priority(complexity_score, settings)
            })
        
        return pd.DataFrame(job_analysis)
    
    def calculate_job_complexity(self, job_settings):
        """Calculate complexity score for migration planning"""
        score = 0
        
        # Task type complexity
        if job_settings.get('notebook_task'):
            score += 2  # Notebooks are moderately complex
        elif job_settings.get('spark_submit_task'):
            score += 4  # Spark submit is more complex
        elif job_settings.get('python_wheel_task'):
            score += 3  # Python wheels are moderately complex
        
        # Cluster dependencies
        if job_settings.get('new_cluster'):
            score += 1  # Creates own cluster
        elif job_settings.get('existing_cluster_id'):
            score += 2  # Depends on existing cluster
        
        # Libraries and dependencies
        libraries = job_settings.get('libraries', [])
        score += min(len(libraries), 5)  # Cap at 5 points for libraries
        
        return score
```

#### Workload Categorization
| Category | Description | Migration Complexity | Alternative Platforms |
|----------|-------------|---------------------|----------------------|
| **ETL Jobs** | Data transformation pipelines | Medium | dbt, Airflow + Spark, Snowflake |
| **ML Training** | Model training workflows | High | SageMaker, Vertex AI, MLflow |
| **Analytics Notebooks** | Interactive analysis | Low | Jupyter, Colab, Snowflake Notebooks |
| **Streaming** | Real-time data processing | High | Kafka Streams, Flink, Kinesis |
| **SQL Analytics** | Dashboard queries | Low | Snowflake, BigQuery, Redshift |

### Migration Timeline
```
Phase 1: Assessment and Planning (Weeks 1-4)
├── Workload inventory
├── Dependency mapping
├── Platform selection
└── Migration strategy

Phase 2: Environment Preparation (Weeks 5-8)
├── Target platform setup
├── Security configuration
├── Data migration planning
└── Tool integration

Phase 3: Workload Migration (Weeks 9-16)
├── Simple workloads (analytics, SQL)
├── ETL pipeline migration
├── ML workload migration
└── Complex streaming jobs

Phase 4: Validation and Cutover (Weeks 17-20)
├── End-to-end testing
├── Performance validation
├── User training
└── Production cutover

Phase 5: Decommissioning (Weeks 21-24)
├── Data archival
├── Resource cleanup
├── Cost validation
└── Documentation
```

## Migration Strategies

### 1. Migrate to Snowflake + dbt

#### For ETL Workloads
```sql
-- Databricks Spark SQL
CREATE TABLE customer_metrics
USING DELTA
LOCATION 's3://data-lake/customer_metrics'
AS
SELECT 
    customer_id,
    region,
    SUM(order_amount) as total_spent,
    COUNT(*) as order_count,
    AVG(order_amount) as avg_order_value
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY customer_id, region;
```

Becomes:

```sql
-- dbt model: models/marts/customer_metrics.sql
{{ config(materialized='table') }}

SELECT 
    customer_id,
    region,
    SUM(order_amount) as total_spent,
    COUNT(*) as order_count,
    AVG(order_amount) as avg_order_value
FROM {{ ref('orders') }} o
JOIN {{ ref('customers') }} c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY customer_id, region
```

#### Migration Automation Script
```python
# databricks_to_dbt_migrator.py
import re
import os
from pathlib import Path

class DatabricksToDBTMigrator:
    def __init__(self, databricks_notebooks_path, dbt_models_path):
        self.databricks_path = Path(databricks_notebooks_path)
        self.dbt_path = Path(dbt_models_path)
    
    def convert_notebook_to_dbt_model(self, notebook_path):
        """Convert Databricks notebook to dbt model"""
        
        with open(notebook_path, 'r') as f:
            notebook_content = f.read()
        
        # Extract SQL cells (simplified - actual implementation would parse JSON)
        sql_cells = self.extract_sql_cells(notebook_content)
        
        for i, sql_content in enumerate(sql_cells):
            if self.is_model_creation(sql_content):
                dbt_model = self.convert_sql_to_dbt(sql_content)
                model_name = self.extract_model_name(sql_content)
                
                # Write dbt model file
                model_file = self.dbt_path / f"{model_name}.sql"
                with open(model_file, 'w') as f:
                    f.write(dbt_model)
                
                print(f"Converted {notebook_path} -> {model_file}")
    
    def convert_sql_to_dbt(self, sql_content):
        """Convert Databricks SQL to dbt model SQL"""
        
        # Remove CREATE TABLE/VIEW statements
        sql_content = re.sub(r'CREATE (TABLE|VIEW) \w+', '', sql_content, flags=re.IGNORECASE)
        sql_content = re.sub(r'USING DELTA', '', sql_content, flags=re.IGNORECASE)
        sql_content = re.sub(r'LOCATION \'[^\']+\'', '', sql_content, flags=re.IGNORECASE)
        
        # Convert table references to dbt refs
        # This is a simplified conversion - real implementation would be more sophisticated
        sql_content = re.sub(r'\bFROM (\w+)', r"FROM {{ ref('\1') }}", sql_content)
        sql_content = re.sub(r'\bJOIN (\w+)', r"JOIN {{ ref('\1') }}", sql_content)
        
        # Add dbt config
        config = "{{ config(materialized='table') }}\n\n"
        
        return config + sql_content.strip()
    
    def extract_sql_cells(self, notebook_content):
        """Extract SQL cells from notebook (simplified)"""
        # In reality, you'd parse the JSON structure of the notebook
        sql_pattern = r'%sql\s+(.*?)(?=%\w+|\Z)'
        matches = re.findall(sql_pattern, notebook_content, re.DOTALL | re.IGNORECASE)
        return matches
    
    def is_model_creation(self, sql_content):
        """Check if SQL creates a table/view"""
        return re.search(r'CREATE (TABLE|VIEW)', sql_content, re.IGNORECASE) is not None
    
    def extract_model_name(self, sql_content):
        """Extract model name from CREATE statement"""
        match = re.search(r'CREATE (?:TABLE|VIEW) (\w+)', sql_content, re.IGNORECASE)
        return match.group(1).lower() if match else 'unknown_model'
```

### 2. Migrate to Apache Airflow + Spark

#### For Complex ETL Pipelines
```python
# Original Databricks Job
# databricks_job.py (simplified representation)
from pyspark.sql import SparkSession

def process_customer_data():
    spark = SparkSession.builder.appName("CustomerProcessing").getOrCreate()
    
    # Read data
    customers = spark.read.table("raw.customers")
    orders = spark.read.table("raw.orders")
    
    # Transform
    customer_metrics = customers.join(orders, "customer_id") \
        .groupBy("customer_id", "region") \
        .agg(
            sum("order_amount").alias("total_spent"),
            count("*").alias("order_count")
        )
    
    # Write results
    customer_metrics.write.mode("overwrite").saveAsTable("analytics.customer_metrics")

# Becomes Airflow DAG
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'customer_processing_pipeline',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False
)

process_customers = SparkSubmitOperator(
    task_id='process_customer_data',
    application='/opt/spark/jobs/customer_processing.py',
    conf={
        'spark.sql.adaptive.enabled': 'true',
        'spark.sql.adaptive.coalescePartitions.enabled': 'true'
    },
    dag=dag
)
```

### 3. Migrate to Cloud-Native ML Platforms

#### For ML Workloads
```python
# Original Databricks MLflow code
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from databricks import feature_store

# Databricks Feature Store
fs = feature_store.FeatureStoreClient()
features = fs.read_table(name="customer_features")

# Becomes AWS SageMaker
import boto3
import sagemaker
from sagemaker.sklearn.estimator import SKLearn

class SageMakerMLMigrator:
    def __init__(self, role, bucket):
        self.sagemaker_session = sagemaker.Session()
        self.role = role
        self.bucket = bucket
    
    def migrate_training_job(self, databricks_notebook_path):
        """Migrate Databricks ML training to SageMaker"""
        
        # Create SageMaker estimator
        sklearn_estimator = SKLearn(
            entry_point='train.py',
            role=self.role,
            instance_type='ml.m5.large',
            framework_version='0.23-1',
            py_version='py3',
            script_mode=True,
            hyperparameters={
                'n_estimators': 100,
                'max_depth': 10
            }
        )
        
        # Convert training script
        training_script = self.convert_databricks_training_script(databricks_notebook_path)
        
        # Save training script
        with open('train.py', 'w') as f:
            f.write(training_script)
        
        # Start training
        sklearn_estimator.fit({'train': f's3://{self.bucket}/train'})
        
        return sklearn_estimator
    
    def convert_databricks_training_script(self, notebook_path):
        """Convert Databricks training notebook to SageMaker script"""
        
        template = '''
import argparse
import joblib
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
import boto3

def model_fn(model_dir):
    """Load model for inference"""
    model = joblib.load(f"{model_dir}/model.joblib")
    return model

def train():
    parser = argparse.ArgumentParser()
    parser.add_argument('--n_estimators', type=int, default=100)
    parser.add_argument('--max_depth', type=int, default=10)
    args = parser.parse_args()
    
    # Load training data
    train_data = pd.read_csv('/opt/ml/input/data/train/train.csv')
    
    X = train_data.drop('target', axis=1)
    y = train_data['target']
    
    # Train model
    model = RandomForestClassifier(
        n_estimators=args.n_estimators,
        max_depth=args.max_depth,
        random_state=42
    )
    model.fit(X, y)
    
    # Save model
    joblib.dump(model, '/opt/ml/model/model.joblib')
    
    # Log metrics
    accuracy = accuracy_score(y, model.predict(X))
    print(f"Training accuracy: {accuracy}")

if __name__ == '__main__':
    train()
        '''
        
        return template
```

## Data Migration

### Delta Lake to Alternative Formats

#### Option 1: Migrate to Iceberg (for Spark-based platforms)
```python
# delta_to_iceberg_migrator.py
from pyspark.sql import SparkSession
import os

class DeltaToIcebergMigrator:
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("DeltaToIceberg") \
            .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
            .config("spark.sql.catalog.spark_catalog", "org.apache.iceberg.spark.SparkSessionCatalog") \
            .config("spark.sql.catalog.spark_catalog.type", "hive") \
            .getOrCreate()
    
    def migrate_table(self, delta_table_path, iceberg_table_name):
        """Migrate Delta table to Iceberg format"""
        
        # Read Delta table
        delta_df = self.spark.read.format("delta").load(delta_table_path)
        
        # Write as Iceberg table
        delta_df.write \
            .format("iceberg") \
            .mode("overwrite") \
            .saveAsTable(iceberg_table_name)
        
        print(f"Migrated {delta_table_path} to {iceberg_table_name}")
        
        # Verify migration
        iceberg_count = self.spark.table(iceberg_table_name).count()
        delta_count = delta_df.count()
        
        if iceberg_count == delta_count:
            print(f"✅ Migration successful: {iceberg_count} rows")
        else:
            print(f"❌ Row count mismatch: Delta={delta_count}, Iceberg={iceberg_count}")
    
    def migrate_with_history(self, delta_table_path, iceberg_table_name):
        """Migrate Delta table with version history"""
        
        # Get Delta table history
        delta_table = DeltaTable.forPath(self.spark, delta_table_path)
        history = delta_table.history().collect()
        
        # Migrate each version
        for version_info in reversed(history):  # Start from oldest
            version = version_info['version']
            timestamp = version_info['timestamp']
            
            # Read specific version
            version_df = self.spark.read \
                .format("delta") \
                .option("versionAsOf", version) \
                .load(delta_table_path)
            
            # Write with timestamp
            version_df.write \
                .format("iceberg") \
                .mode("append") \
                .option("write.metadata.create-table-transaction", "true") \
                .saveAsTable(iceberg_table_name)
```

#### Option 2: Migrate to Snowflake
```python
# delta_to_snowflake_migrator.py
import snowflake.connector
from pyspark.sql import SparkSession

class DeltaToSnowflakeMigrator:
    def __init__(self, snowflake_conn_params):
        self.spark = SparkSession.builder.appName("DeltaToSnowflake").getOrCreate()
        self.sf_conn = snowflake.connector.connect(**snowflake_conn_params)
    
    def migrate_table(self, delta_table_path, snowflake_table, batch_size=100000):
        """Migrate Delta table to Snowflake in batches"""
        
        # Read Delta table
        delta_df = self.spark.read.format("delta").load(delta_table_path)
        
        # Get schema for Snowflake table creation
        schema_ddl = self.generate_snowflake_schema(delta_df.schema)
        
        # Create Snowflake table
        cursor = self.sf_conn.cursor()
        cursor.execute(f"CREATE OR REPLACE TABLE {snowflake_table} ({schema_ddl})")
        
        # Migrate data in batches
        total_rows = delta_df.count()
        num_partitions = max(1, total_rows // batch_size)
        
        delta_df.repartition(num_partitions).write \
            .format("snowflake") \
            .option("sfUrl", snowflake_conn_params['account']) \
            .option("sfUser", snowflake_conn_params['user']) \
            .option("sfPassword", snowflake_conn_params['password']) \
            .option("sfDatabase", snowflake_conn_params['database']) \
            .option("sfSchema", snowflake_conn_params['schema']) \
            .option("sfWarehouse", snowflake_conn_params['warehouse']) \
            .option("dbtable", snowflake_table) \
            .mode("append") \
            .save()
        
        # Verify migration
        cursor.execute(f"SELECT COUNT(*) FROM {snowflake_table}")
        sf_count = cursor.fetchone()[0]
        
        if sf_count == total_rows:
            print(f"✅ Migration successful: {sf_count} rows")
        else:
            print(f"❌ Row count mismatch: Delta={total_rows}, Snowflake={sf_count}")
    
    def generate_snowflake_schema(self, spark_schema):
        """Convert Spark schema to Snowflake DDL"""
        
        type_mapping = {
            'StringType': 'VARCHAR',
            'IntegerType': 'NUMBER(38,0)',
            'LongType': 'NUMBER(38,0)',
            'DoubleType': 'FLOAT',
            'BooleanType': 'BOOLEAN',
            'DateType': 'DATE',
            'TimestampType': 'TIMESTAMP_NTZ'
        }
        
        columns = []
        for field in spark_schema.fields:
            sf_type = type_mapping.get(type(field.dataType).__name__, 'VARCHAR')
            nullable = "" if field.nullable else " NOT NULL"
            columns.append(f"{field.name} {sf_type}{nullable}")
        
        return ",\n    ".join(columns)
```

## Code Migration Patterns

### Notebook to Script Conversion
```python
# notebook_to_script_converter.py
import nbformat
import re
from pathlib import Path

class NotebookConverter:
    def __init__(self):
        self.magic_commands = {
            '%sql': self.convert_sql_magic,
            '%python': self.convert_python_magic,
            '%scala': self.convert_scala_magic,
            '%r': self.convert_r_magic
        }
    
    def convert_databricks_notebook(self, notebook_path, output_dir):
        """Convert Databricks notebook to standalone scripts"""
        
        with open(notebook_path, 'r') as f:
            notebook = nbformat.read(f, as_version=4)
        
        # Group cells by language
        scripts = {
            'python': [],
            'sql': [],
            'scala': [],
            'r': []
        }
        
        for cell in notebook.cells:
            if cell.cell_type == 'code':
                source = cell.source
                
                # Detect magic commands
                for magic, converter in self.magic_commands.items():
                    if source.startswith(magic):
                        language = magic[1:]  # Remove %
                        converted_code = converter(source)
                        scripts[language].append(converted_code)
                        break
                else:
                    # Default to Python if no magic command
                    scripts['python'].append(source)
        
        # Write separate files for each language
        output_path = Path(output_dir)
        notebook_name = Path(notebook_path).stem
        
        for language, code_blocks in scripts.items():
            if code_blocks:
                script_content = self.generate_script_header(language) + '\n\n'
                script_content += '\n\n'.join(code_blocks)
                
                script_file = output_path / f"{notebook_name}.{self.get_file_extension(language)}"
                with open(script_file, 'w') as f:
                    f.write(script_content)
                
                print(f"Created {script_file}")
    
    def convert_sql_magic(self, cell_source):
        """Convert %sql magic to standalone SQL"""
        return cell_source.replace('%sql\n', '').strip()
    
    def convert_python_magic(self, cell_source):
        """Convert %python magic to Python code"""
        return cell_source.replace('%python\n', '').strip()
    
    def generate_script_header(self, language):
        """Generate appropriate header for each language"""
        
        headers = {
            'python': '''#!/usr/bin/env python3
"""
Converted from Databricks notebook
Generated automatically - review before using in production
"""

from pyspark.sql import SparkSession
import pyspark.sql.functions as F

# Initialize Spark session
spark = SparkSession.builder.appName("MigratedScript").getOrCreate()''',
            
            'sql': '''-- Converted from Databricks notebook
-- Generated automatically - review before using in production
-- Note: You may need to adjust table references and syntax for your target platform''',
            
            'scala': '''// Converted from Databricks notebook
// Generated automatically - review before using in production

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object MigratedScript {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder.appName("MigratedScript").getOrCreate()
    import spark.implicits._'''
        }
        
        return headers.get(language, '')
    
    def get_file_extension(self, language):
        """Get file extension for each language"""
        extensions = {
            'python': 'py',
            'sql': 'sql',
            'scala': 'scala',
            'r': 'R'
        }
        return extensions.get(language, 'txt')
```

### Spark Configuration Migration
```python
# spark_config_migrator.py
import yaml
import json

class SparkConfigMigrator:
    def __init__(self):
        self.databricks_to_standard_mapping = {
            'spark.databricks.delta.preview.enabled': 'spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension',
            'spark.databricks.cluster.profile': None,  # Databricks-specific, remove
            'spark.databricks.passthrough.enabled': None,  # Databricks-specific, remove
            'spark.databricks.pyspark.enableProcessIsolation': 'spark.python.worker.reuse=false',
        }
    
    def convert_cluster_config(self, databricks_cluster_config):
        """Convert Databricks cluster config to standard Spark config"""
        
        # Extract Spark configuration
        spark_conf = databricks_cluster_config.get('spark_conf', {})
        
        # Convert Databricks-specific settings
        converted_conf = {}
        for key, value in spark_conf.items():
            if key in self.databricks_to_standard_mapping:
                standard_key = self.databricks_to_standard_mapping[key]
                if standard_key:  # Skip None values (removed configs)
                    converted_conf[standard_key] = value
            else:
                converted_conf[key] = value
        
        # Add recommended settings for standalone Spark
        recommended_settings = {
            'spark.sql.adaptive.enabled': 'true',
            'spark.sql.adaptive.coalescePartitions.enabled': 'true',
            'spark.sql.adaptive.skewJoin.enabled': 'true',
            'spark.serializer': 'org.apache.spark.serializer.KryoSerializer'
        }
        
        converted_conf.update(recommended_settings)
        
        return {
            'spark_configuration': converted_conf,
            'node_type_recommendation': self.recommend_instance_type(databricks_cluster_config),
            'scaling_recommendation': self.recommend_scaling_policy(databricks_cluster_config)
        }
    
    def recommend_instance_type(self, cluster_config):
        """Recommend instance types for different platforms"""
        
        node_type = cluster_config.get('node_type_id', '')
        num_workers = cluster_config.get('num_workers', 1)
        
        # Map Databricks instance types to cloud equivalents
        instance_mapping = {
            'i3.xlarge': {
                'aws': 'i3.xlarge',
                'azure': 'Standard_L8s_v2',
                'gcp': 'n1-standard-4'
            },
            'i3.2xlarge': {
                'aws': 'i3.2xlarge',
                'azure': 'Standard_L16s_v2',
                'gcp': 'n1-standard-8'
            }
        }
        
        return instance_mapping.get(node_type, {
            'aws': 'm5.xlarge',
            'azure': 'Standard_D4s_v3',
            'gcp': 'n1-standard-4'
        })
```

## Testing and Validation

### Automated Migration Testing
```python
# migration_validator.py
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count, sum as spark_sum

class MigrationValidator:
    def __init__(self, source_session, target_session):
        self.source_spark = source_session
        self.target_spark = target_session
    
    def validate_table_migration(self, source_table, target_table):
        """Validate that table migration was successful"""
        
        # Read tables
        source_df = self.source_spark.table(source_table)
        target_df = self.target_spark.table(target_table)
        
        # Row count validation
        source_count = source_df.count()
        target_count = target_df.count()
        
        assert source_count == target_count, f"Row count mismatch: source={source_count}, target={target_count}"
        
        # Schema validation
        source_schema = set(source_df.columns)
        target_schema = set(target_df.columns)
        
        assert source_schema == target_schema, f"Schema mismatch: source={source_schema}, target={target_schema}"
        
        # Data sampling validation
        source_sample = source_df.sample(0.1, seed=42).collect()
        target_sample = target_df.sample(0.1, seed=42).collect()
        
        # Compare sample data (simplified comparison)
        assert len(source_sample) == len(target_sample), "Sample size mismatch"
        
        print(f"✅ Table {source_table} -> {target_table} validation passed")
    
    def validate_job_output(self, databricks_output, migrated_output):
        """Validate that migrated job produces same output"""
        
        # Compare row counts
        db_count = databricks_output.count()
        migrated_count = migrated_output.count()
        
        assert db_count == migrated_count, f"Output row count mismatch: databricks={db_count}, migrated={migrated_count}"
        
        # Compare aggregations
        for col_name in databricks_output.columns:
            if databricks_output.schema[col_name].dataType.typeName() in ['integer', 'long', 'double', 'float']:
                db_sum = databricks_output.agg(spark_sum(col(col_name))).collect()[0][0]
                migrated_sum = migrated_output.agg(spark_sum(col(col_name))).collect()[0][0]
                
                assert abs(db_sum - migrated_sum) < 0.01, f"Sum mismatch for {col_name}: databricks={db_sum}, migrated={migrated_sum}"
        
        print("✅ Job output validation passed")
    
    def validate_performance(self, databricks_metrics, migrated_metrics):
        """Validate that performance is acceptable after migration"""
        
        # Compare execution times
        db_time = databricks_metrics.get('execution_time_seconds', 0)
        migrated_time = migrated_metrics.get('execution_time_seconds', 0)
        
        # Allow 50% performance degradation
        assert migrated_time <= db_time * 1.5, f"Performance degradation too high: {migrated_time}s vs {db_time}s"
        
        # Compare resource usage
        db_cores = databricks_metrics.get('cores_used', 1)
        migrated_cores = migrated_metrics.get('cores_used', 1)
        
        # Resource usage should be comparable
        assert migrated_cores <= db_cores * 2, f"Resource usage too high: {migrated_cores} cores vs {db_cores} cores"
        
        print("✅ Performance validation passed")
```

## Cost Analysis

### Cost Comparison Framework
```python
# cost_analyzer.py
import pandas as pd
from datetime import datetime, timedelta

class CostAnalyzer:
    def __init__(self):
        self.databricks_pricing = {
            'standard': 0.15,  # per DBU
            'premium': 0.30,   # per DBU
            'compute_unit_factor': {
                'i3.xlarge': 1.0,
                'i3.2xlarge': 2.0,
                'i3.4xlarge': 4.0
            }
        }
    
    def calculate_databricks_cost(self, usage_data):
        """Calculate current Databricks costs"""
        
        total_cost = 0
        cost_breakdown = []
        
        for _, row in usage_data.iterrows():
            # Calculate DBUs consumed
            node_factor = self.databricks_pricing['compute_unit_factor'].get(row['node_type'], 1.0)
            dbus_consumed = row['running_hours'] * row['num_workers'] * node_factor
            
            # Calculate cost (assuming premium tier)
            cost = dbus_consumed * self.databricks_pricing['premium']
            total_cost += cost
            
            cost_breakdown.append({
                'cluster_name': row['cluster_name'],
                'running_hours': row['running_hours'],
                'dbus_consumed': dbus_consumed,
                'cost': cost
            })
        
        return {
            'total_cost': total_cost,
            'breakdown': cost_breakdown
        }
    
    def estimate_alternative_platform_cost(self, workload_analysis, target_platform):
        """Estimate costs for alternative platforms"""
        
        if target_platform == 'snowflake':
            return self.estimate_snowflake_cost(workload_analysis)
        elif target_platform == 'aws_emr':
            return self.estimate_emr_cost(workload_analysis)
        elif target_platform == 'gcp_dataproc':
            return self.estimate_dataproc_cost(workload_analysis)
        else:
            raise ValueError(f"Unknown platform: {target_platform}")
    
    def estimate_snowflake_cost(self, workload_analysis):
        """Estimate Snowflake costs for migrated workloads"""
        
        # Snowflake pricing (simplified)
        warehouse_pricing = {
            'X-Small': 1.0,   # credits per hour
            'Small': 2.0,
            'Medium': 4.0,
            'Large': 8.0,
            'X-Large': 16.0
        }
        
        credit_cost = 2.50  # USD per credit (varies by region and commitment)
        
        # Estimate warehouse size needed based on current cluster usage
        total_compute_hours = workload_analysis['running_hours'].sum()
        
        # Assume Medium warehouse for most workloads
        estimated_credits = total_compute_hours * warehouse_pricing['Medium']
        storage_cost = workload_analysis['data_size_gb'].sum() * 0.0236  # $0.0236 per GB per month
        
        total_cost = (estimated_credits * credit_cost) + storage_cost
        
        return {
            'compute_cost': estimated_credits * credit_cost,
            'storage_cost': storage_cost,
            'total_cost': total_cost,
            'credits_consumed': estimated_credits
        }
    
    def generate_cost_comparison_report(self, current_cost, alternative_costs):
        """Generate cost comparison report"""
        
        report = f"""
# Cost Analysis Report

## Current Databricks Costs
- Total Monthly Cost: ${current_cost['total_cost']:,.2f}
- Cost per Hour: ${current_cost['total_cost'] / 730:.2f}

## Alternative Platform Costs
"""
        
        for platform, costs in alternative_costs.items():
            savings = current_cost['total_cost'] - costs['total_cost']
            savings_percent = (savings / current_cost['total_cost']) * 100
            
            report += f"""
### {platform.title()}
- Total Monthly Cost: ${costs['total_cost']:,.2f}
- Monthly Savings: ${savings:,.2f} ({savings_percent:.1f}%)
- Payback Period: {self.calculate_payback_period(savings, platform)}
"""
        
        return report
    
    def calculate_payback_period(self, monthly_savings, platform):
        """Calculate payback period for migration"""
        
        # Estimated migration costs (simplified)
        migration_costs = {
            'snowflake': 50000,  # Professional services, training, etc.
            'aws_emr': 30000,
            'gcp_dataproc': 30000
        }
        
        migration_cost = migration_costs.get(platform, 40000)
        
        if monthly_savings <= 0:
            return "No payback (costs increase)"
        
        payback_months = migration_cost / monthly_savings
        return f"{payback_months:.1f} months"
```

## Decommissioning Process

### Data Archival Strategy
```python
# databricks_archival.py
import boto3
from datetime import datetime, timedelta

class DatabricksArchiver:
    def __init__(self, aws_session):
        self.s3 = aws_session.client('s3')
        self.archive_bucket = 'databricks-archive'
    
    def archive_delta_tables(self, table_list, retention_days=2555):  # 7 years
        """Archive Delta tables to S3 with lifecycle policy"""
        
        for table_info in table_list:
            table_path = table_info['path']
            table_name = table_info['name']
            
            # Create archive path
            archive_date = datetime.now().strftime('%Y-%m-%d')
            archive_path = f"databricks-archive/{archive_date}/{table_name}/"
            
            # Copy table data to archive bucket
            self.copy_delta_table_to_archive(table_path, archive_path)
            
            # Set lifecycle policy for cost optimization
            self.set_archive_lifecycle_policy(archive_path, retention_days)
            
            print(f"Archived {table_name} to {archive_path}")
    
    def archive_notebooks(self, workspace_export_path):
        """Archive Databricks notebooks"""
        
        archive_date = datetime.now().strftime('%Y-%m-%d')
        archive_key = f"databricks-archive/{archive_date}/notebooks/workspace_export.dbc"
        
        # Upload workspace export to S3
        self.s3.upload_file(workspace_export_path, self.archive_bucket, archive_key)
        
        # Create documentation
        metadata = {
            'export_date': archive_date,
            'source': 'databricks_workspace',
            'format': 'dbc',
            'retention_years': 7
        }
        
        # Store metadata
        metadata_key = f"databricks-archive/{archive_date}/notebooks/metadata.json"
        self.s3.put_object(
            Bucket=self.archive_bucket,
            Key=metadata_key,
            Body=json.dumps(metadata, indent=2)
        )
    
    def create_decommission_report(self, decommissioned_resources):
        """Create final decommissioning report"""
        
        report = {
            'decommission_date': datetime.now().isoformat(),
            'resources_decommissioned': decommissioned_resources,
            'data_archived': True,
            'cost_savings_achieved': True,
            'migration_completed': True
        }
        
        # Store report
        report_key = f"databricks-archive/decommission-report-{datetime.now().strftime('%Y-%m-%d')}.json"
        self.s3.put_object(
            Bucket=self.archive_bucket,
            Key=report_key,
            Body=json.dumps(report, indent=2)
        )
        
        return report
```

### Resource Cleanup Checklist
- [ ] Export all notebooks and code
- [ ] Archive all Delta tables and data
- [ ] Document migration decisions and lessons learned
- [ ] Transfer ownership of migrated workloads
- [ ] Update documentation and runbooks
- [ ] Cancel Databricks subscription
- [ ] Delete Databricks workspaces
- [ ] Clean up associated cloud resources
- [ ] Update cost center allocations
- [ ] Notify stakeholders of completion

## Success Metrics

### Migration Success Criteria
| Metric | Target | Measurement Method |
|--------|--------|--------------------|
| **Workload Migration** | 100% of critical workloads | Job execution validation |
| **Data Accuracy** | 99.99% data consistency | Automated validation scripts |
| **Performance** | ≤20% performance degradation | Benchmark comparisons |
| **Cost Reduction** | ≥30% cost savings | Monthly cost analysis |
| **Downtime** | <4 hours for critical systems | Cutover time tracking |
| **User Training** | 100% of users trained | Training completion tracking |

### Post-Migration Monitoring
```python
# post_migration_monitor.py
class PostMigrationMonitor:
    def __init__(self, monitoring_config):
        self.config = monitoring_config
    
    def monitor_job_health(self):
        """Monitor migrated job health"""
        
        metrics = {
            'success_rate': self.calculate_job_success_rate(),
            'performance_vs_baseline': self.compare_performance_to_baseline(),
            'error_frequency': self.count_recent_errors(),
            'user_satisfaction': self.collect_user_feedback()
        }
        
        return metrics
    
    def generate_weekly_report(self):
        """Generate weekly migration health report"""
        
        report = {
            'migration_health_score': self.calculate_health_score(),
            'issues_identified': self.identify_issues(),
            'recommendations': self.generate_recommendations(),
            'cost_tracking': self.track_cost_savings()
        }
        
        return report
```

---
**Document Control**
- Version: 1.0
- Created: [Date]
- Last Updated: [Date]
- Author: Data Platform Team
- Reviewers: [Names]
- Next Review: [Date]