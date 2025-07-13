# Data Platform Tools and Automation Guide

## Overview
This guide provides comprehensive information about tools, scripts, and automation used in our data platform operations.

## Tool Categories

### 1. Data Integration Tools

#### Fivetran
**Purpose**: Automated ELT connector for various data sources
**Use Cases**: 
- SaaS application data extraction
- Database replication
- Real-time data synchronization

**Configuration Example**:
```yaml
# fivetran_config.yaml
connectors:
  salesforce:
    connector_type: "salesforce"
    destination_schema: "RAW_SALESFORCE"
    sync_frequency: "15min"
    tables:
      - "Account"
      - "Contact"
      - "Opportunity"
      - "Lead"
    
  postgres_db:
    connector_type: "postgres"
    host: "${DB_HOST}"
    port: 5432
    database: "production"
    destination_schema: "RAW_POSTGRES"
    sync_frequency: "1hr"
```

**Management Scripts**:
```python
# scripts/fivetran_manager.py
import requests
import json

class FivetranManager:
    def __init__(self, api_key, api_secret):
        self.auth = (api_key, api_secret)
        self.base_url = "https://api.fivetran.com/v1"
    
    def get_connector_status(self, connector_id):
        """Get status of a specific connector"""
        response = requests.get(
            f"{self.base_url}/connectors/{connector_id}",
            auth=self.auth
        )
        return response.json()
    
    def trigger_sync(self, connector_id):
        """Manually trigger connector sync"""
        response = requests.post(
            f"{self.base_url}/connectors/{connector_id}/force",
            auth=self.auth
        )
        return response.json()
    
    def pause_connector(self, connector_id):
        """Pause a connector"""
        data = {"paused": True}
        response = requests.patch(
            f"{self.base_url}/connectors/{connector_id}",
            auth=self.auth,
            json=data
        )
        return response.json()
```

#### dbt (Data Build Tool)
**Purpose**: SQL-based data transformation and modeling
**Use Cases**:
- Data warehouse modeling
- Data quality testing
- Documentation generation

**Project Structure**:
```
dbt_project/
├── dbt_project.yml
├── models/
│   ├── staging/
│   │   ├── stg_customers.sql
│   │   └── stg_orders.sql
│   ├── intermediate/
│   │   └── int_customer_orders.sql
│   └── marts/
│       ├── dim_customers.sql
│       └── fact_orders.sql
├── tests/
├── macros/
└── docs/
```

**Model Example**:
```sql
-- models/staging/stg_customers.sql
{{ config(materialized='view') }}

SELECT
    customer_id,
    UPPER(TRIM(first_name)) as first_name,
    UPPER(TRIM(last_name)) as last_name,
    LOWER(TRIM(email)) as email,
    REGEXP_REPLACE(phone, '[^0-9]', '') as phone_cleaned,
    created_at,
    updated_at,
    _fivetran_synced
FROM {{ source('raw_salesforce', 'contact') }}
WHERE customer_id IS NOT NULL
```

**Automation Script**:
```python
# scripts/dbt_automation.py
import subprocess
import json
from datetime import datetime

class DBTAutomation:
    def __init__(self, project_dir):
        self.project_dir = project_dir
    
    def run_models(self, models=None, full_refresh=False):
        """Run dbt models"""
        cmd = ["dbt", "run"]
        
        if models:
            cmd.extend(["--models", models])
        
        if full_refresh:
            cmd.append("--full-refresh")
        
        result = subprocess.run(
            cmd,
            cwd=self.project_dir,
            capture_output=True,
            text=True
        )
        
        return {
            'success': result.returncode == 0,
            'stdout': result.stdout,
            'stderr': result.stderr
        }
    
    def run_tests(self):
        """Run dbt tests"""
        result = subprocess.run(
            ["dbt", "test"],
            cwd=self.project_dir,
            capture_output=True,
            text=True
        )
        
        return {
            'success': result.returncode == 0,
            'stdout': result.stdout,
            'stderr': result.stderr
        }
    
    def generate_docs(self):
        """Generate and serve dbt documentation"""
        # Generate docs
        subprocess.run(["dbt", "docs", "generate"], cwd=self.project_dir)
        
        # Serve docs (in background)
        subprocess.Popen(["dbt", "docs", "serve", "--port", "8080"], cwd=self.project_dir)
```

### 2. Workflow Orchestration

#### Apache Airflow
**Purpose**: Workflow orchestration and scheduling
**Use Cases**:
- ETL pipeline scheduling
- Data quality monitoring
- Cross-system coordination

**DAG Example**:
```python
# dags/customer_data_pipeline.py
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.operators.bash_operator import BashOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email_on_retry': False
}

dag = DAG(
    'customer_data_pipeline',
    default_args=default_args,
    description='Customer data processing pipeline',
    schedule_interval='@daily',
    catchup=False,
    tags=['customer', 'etl']
)

# Extract data
extract_salesforce = PythonOperator(
    task_id='extract_salesforce_data',
    python_callable=extract_salesforce_data,
    dag=dag
)

# Transform with dbt
transform_data = BashOperator(
    task_id='transform_customer_data',
    bash_command='cd /opt/dbt && dbt run --models customers',
    dag=dag
)

# Data quality checks
quality_check = SnowflakeOperator(
    task_id='quality_check_customers',
    sql='SELECT COUNT(*) FROM analytics.dim_customers WHERE customer_id IS NULL',
    snowflake_conn_id='snowflake_default',
    dag=dag
)

# Set dependencies
extract_salesforce >> transform_data >> quality_check
```

**Management Utilities**:
```python
# scripts/airflow_utils.py
from airflow.models import DagBag, TaskInstance
from airflow.utils.state import State

class AirflowManager:
    def __init__(self):
        self.dagbag = DagBag()
    
    def get_dag_status(self, dag_id):
        """Get current status of a DAG"""
        dag = self.dagbag.get_dag(dag_id)
        if not dag:
            return None
        
        latest_run = dag.get_last_dagrun()
        return {
            'dag_id': dag_id,
            'last_run_date': latest_run.execution_date if latest_run else None,
            'state': latest_run.state if latest_run else None,
            'is_paused': dag.is_paused
        }
    
    def trigger_dag(self, dag_id, conf=None):
        """Trigger a DAG run"""
        dag = self.dagbag.get_dag(dag_id)
        dag.create_dagrun(
            run_id=f"manual_{datetime.now().isoformat()}",
            execution_date=datetime.now(),
            state=State.RUNNING,
            conf=conf
        )
```

### 3. Data Quality Tools

#### Great Expectations
**Purpose**: Data validation and quality testing
**Use Cases**:
- Data profiling
- Automated testing
- Quality reporting

**Configuration**:
```yaml
# great_expectations/great_expectations.yml
config_version: 3.0

datasources:
  snowflake_db:
    class_name: Datasource
    execution_engine:
      class_name: SqlAlchemyExecutionEngine
      connection_string: snowflake://user:password@account/database/schema?warehouse=warehouse
    data_connectors:
      default_inferred_data_connector:
        class_name: InferredAssetSqlDataConnector
        include_schema_name: true
```

**Expectation Suite Example**:
```python
# expectations/customer_expectations.py
import great_expectations as ge

def create_customer_expectations():
    """Create expectation suite for customer data"""
    
    # Connect to data
    context = ge.get_context()
    datasource = context.get_datasource("snowflake_db")
    
    # Get data asset
    asset = datasource.get_asset("customers")
    batch_request = asset.build_batch_request()
    
    # Create validator
    validator = context.get_validator(
        batch_request=batch_request,
        expectation_suite_name="customer_expectations"
    )
    
    # Define expectations
    validator.expect_table_row_count_to_be_between(min_value=1000)
    validator.expect_column_values_to_not_be_null("customer_id")
    validator.expect_column_values_to_be_unique("customer_id")
    validator.expect_column_values_to_match_regex("email", r'^[^@]+@[^@]+\.[^@]+$')
    validator.expect_column_values_to_be_between("age", min_value=0, max_value=120)
    
    # Save expectation suite
    validator.save_expectation_suite()
    
    return validator.expectation_suite
```

**Automation Script**:
```python
# scripts/data_quality_runner.py
import great_expectations as ge
from datetime import datetime

class DataQualityRunner:
    def __init__(self):
        self.context = ge.get_context()
    
    def run_checkpoint(self, checkpoint_name):
        """Run a Great Expectations checkpoint"""
        
        checkpoint = self.context.get_checkpoint(checkpoint_name)
        results = checkpoint.run()
        
        return {
            'success': results.success,
            'run_id': results.run_id,
            'validation_results': results.list_validation_results(),
            'timestamp': datetime.now()
        }
    
    def generate_data_docs(self):
        """Generate and update data documentation"""
        self.context.build_data_docs()
```

### 4. Monitoring and Alerting

#### DataDog Integration
**Purpose**: Infrastructure and application monitoring
**Use Cases**:
- Performance monitoring
- Error tracking
- Custom metrics

**Configuration**:
```python
# monitoring/datadog_integration.py
from datadog import initialize, statsd
import os

# Initialize DataDog
options = {
    'api_key': os.getenv('DATADOG_API_KEY'),
    'app_key': os.getenv('DATADOG_APP_KEY')
}
initialize(**options)

class DataPlatformMetrics:
    def __init__(self):
        self.statsd = statsd
    
    def record_pipeline_duration(self, pipeline_name, duration_seconds):
        """Record pipeline execution time"""
        self.statsd.histogram(
            'data_platform.pipeline.duration',
            duration_seconds,
            tags=[f'pipeline:{pipeline_name}']
        )
    
    def record_data_quality_score(self, table_name, score):
        """Record data quality score"""
        self.statsd.gauge(
            'data_platform.data_quality.score',
            score,
            tags=[f'table:{table_name}']
        )
    
    def increment_error_count(self, component, error_type):
        """Increment error counter"""
        self.statsd.increment(
            'data_platform.errors',
            tags=[f'component:{component}', f'error_type:{error_type}']
        )
```

#### Custom Monitoring Dashboard
```python
# monitoring/dashboard_generator.py
import plotly.graph_objects as go
import plotly.express as px
from plotly.subplots import make_subplots

class MonitoringDashboard:
    def __init__(self, metrics_data):
        self.data = metrics_data
    
    def create_pipeline_health_chart(self):
        """Create pipeline health status chart"""
        
        fig = go.Figure()
        
        # Add pipeline success rates
        fig.add_trace(go.Bar(
            x=self.data['pipeline_names'],
            y=self.data['success_rates'],
            name='Success Rate',
            marker_color='green'
        ))
        
        fig.update_layout(
            title="Pipeline Health Dashboard",
            xaxis_title="Pipeline",
            yaxis_title="Success Rate (%)",
            yaxis=dict(range=[0, 100])
        )
        
        return fig
    
    def create_data_freshness_chart(self):
        """Create data freshness monitoring chart"""
        
        fig = px.line(
            self.data['freshness_data'],
            x='timestamp',
            y='lag_minutes',
            color='table_name',
            title='Data Freshness Monitoring'
        )
        
        # Add SLA line
        fig.add_hline(y=60, line_dash="dash", line_color="red", 
                     annotation_text="SLA Threshold (60 min)")
        
        return fig
```

### 5. CI/CD and Deployment Tools

#### GitHub Actions Workflows
**Purpose**: Continuous integration and deployment
**Use Cases**:
- Automated testing
- Code deployment
- Documentation updates

**Workflow Example**:
```yaml
# .github/workflows/data-platform-ci.yml
name: Data Platform CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run dbt tests
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
        run: |
          cd dbt_project
          dbt deps
          dbt run --models tag:test
          dbt test
      
      - name: Run Great Expectations
        run: |
          python scripts/run_data_quality_checks.py
      
      - name: Security scan
        uses: github/super-linter@v4
        env:
          DEFAULT_BRANCH: main
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to production
        env:
          AIRFLOW_HOST: ${{ secrets.AIRFLOW_HOST }}
          AIRFLOW_USERNAME: ${{ secrets.AIRFLOW_USERNAME }}
          AIRFLOW_PASSWORD: ${{ secrets.AIRFLOW_PASSWORD }}
        run: |
          python scripts/deploy_dags.py
```

#### Terraform Infrastructure
**Purpose**: Infrastructure as code
**Use Cases**:
- Cloud resource provisioning
- Environment management
- Configuration standardization

**Configuration Example**:
```hcl
# infrastructure/snowflake.tf
terraform {
  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.68"
    }
  }
}

provider "snowflake" {
  account  = var.snowflake_account
  username = var.snowflake_username
  password = var.snowflake_password
  role     = "SYSADMIN"
}

# Create databases
resource "snowflake_database" "analytics" {
  name    = "ANALYTICS"
  comment = "Analytics database for processed data"
}

resource "snowflake_database" "raw" {
  name    = "RAW"
  comment = "Raw data from source systems"
}

# Create schemas
resource "snowflake_schema" "customer" {
  database = snowflake_database.analytics.name
  name     = "CUSTOMER"
  comment  = "Customer data marts"
}

# Create warehouses
resource "snowflake_warehouse" "etl_warehouse" {
  name           = "ETL_WH"
  warehouse_size = "MEDIUM"
  auto_suspend   = 60
  auto_resume    = true
  comment        = "Warehouse for ETL processes"
}
```

### 6. Utility Scripts and Tools

#### Data Lineage Tracker
```python
# scripts/lineage_tracker.py
import networkx as nx
import json
from sqlparse import parse, sql

class DataLineageTracker:
    def __init__(self):
        self.graph = nx.DiGraph()
    
    def parse_sql_lineage(self, sql_query):
        """Parse SQL to extract data lineage"""
        parsed = parse(sql_query)[0]
        
        # Extract table references
        tables_referenced = []
        tables_created = []
        
        # Simple parsing logic (can be enhanced)
        sql_upper = sql_query.upper()
        
        if 'CREATE TABLE' in sql_upper or 'CREATE VIEW' in sql_upper:
            # Extract target table
            target = self.extract_target_table(sql_query)
            tables_created.append(target)
        
        if 'FROM' in sql_upper:
            # Extract source tables
            sources = self.extract_source_tables(sql_query)
            tables_referenced.extend(sources)
        
        return {
            'sources': tables_referenced,
            'targets': tables_created
        }
    
    def add_lineage_relationship(self, source_table, target_table, transformation_type):
        """Add lineage relationship to graph"""
        self.graph.add_edge(
            source_table, 
            target_table, 
            transformation=transformation_type
        )
    
    def get_upstream_dependencies(self, table_name):
        """Get all upstream dependencies for a table"""
        return list(nx.ancestors(self.graph, table_name))
    
    def get_downstream_dependencies(self, table_name):
        """Get all downstream dependencies for a table"""
        return list(nx.descendants(self.graph, table_name))
    
    def export_lineage_graph(self, format='json'):
        """Export lineage graph"""
        if format == 'json':
            return nx.node_link_data(self.graph)
        elif format == 'dot':
            return nx.drawing.nx_pydot.to_pydot(self.graph).to_string()
```

#### Environment Manager
```python
# scripts/environment_manager.py
import os
import yaml
import subprocess

class EnvironmentManager:
    def __init__(self, config_file='environments.yml'):
        with open(config_file, 'r') as f:
            self.config = yaml.safe_load(f)
    
    def switch_environment(self, env_name):
        """Switch to a specific environment"""
        if env_name not in self.config['environments']:
            raise ValueError(f"Environment {env_name} not found")
        
        env_config = self.config['environments'][env_name]
        
        # Set environment variables
        for key, value in env_config.get('variables', {}).items():
            os.environ[key] = str(value)
        
        # Update dbt profiles
        self.update_dbt_profile(env_config.get('dbt', {}))
        
        # Update Airflow connections
        self.update_airflow_connections(env_config.get('airflow', {}))
        
        print(f"Switched to environment: {env_name}")
    
    def update_dbt_profile(self, dbt_config):
        """Update dbt profiles.yml"""
        profiles_path = os.path.expanduser('~/.dbt/profiles.yml')
        
        with open(profiles_path, 'r') as f:
            profiles = yaml.safe_load(f) or {}
        
        if 'maritime' not in profiles:
            profiles['maritime'] = {}
        
        profiles['maritime']['target'] = dbt_config.get('target', 'dev')
        profiles['maritime']['outputs'] = dbt_config.get('outputs', {})
        
        with open(profiles_path, 'w') as f:
            yaml.dump(profiles, f, default_flow_style=False)
    
    def validate_environment(self, env_name):
        """Validate environment configuration"""
        env_config = self.config['environments'][env_name]
        
        validation_results = {
            'database_connection': False,
            'api_access': False,
            'required_tools': False
        }
        
        # Test database connection
        try:
            # Implementation depends on your database
            validation_results['database_connection'] = True
        except:
            pass
        
        # Test API access
        try:
            # Test API endpoints
            validation_results['api_access'] = True
        except:
            pass
        
        # Check required tools
        required_tools = env_config.get('required_tools', [])
        tools_available = all(
            subprocess.run(['which', tool], capture_output=True).returncode == 0
            for tool in required_tools
        )
        validation_results['required_tools'] = tools_available
        
        return validation_results
```

## Tool Integration Patterns

### Cross-Tool Communication
```python
# utils/tool_integration.py
class ToolIntegrator:
    def __init__(self):
        self.tools = {
            'airflow': AirflowManager(),
            'dbt': DBTAutomation('/opt/dbt'),
            'fivetran': FivetranManager(api_key, api_secret),
            'great_expectations': DataQualityRunner(),
            'datadog': DataPlatformMetrics()
        }
    
    def orchestrate_data_pipeline(self, pipeline_config):
        """Orchestrate cross-tool data pipeline"""
        
        results = {}
        
        # 1. Trigger data extraction
        if 'fivetran' in pipeline_config:
            connector_results = []
            for connector_id in pipeline_config['fivetran']['connectors']:
                result = self.tools['fivetran'].trigger_sync(connector_id)
                connector_results.append(result)
            results['extraction'] = connector_results
        
        # 2. Run transformations
        if 'dbt' in pipeline_config:
            dbt_result = self.tools['dbt'].run_models(
                models=pipeline_config['dbt'].get('models')
            )
            results['transformation'] = dbt_result
        
        # 3. Run data quality checks
        if 'great_expectations' in pipeline_config:
            for checkpoint in pipeline_config['great_expectations']['checkpoints']:
                gx_result = self.tools['great_expectations'].run_checkpoint(checkpoint)
                results['data_quality'] = gx_result
        
        # 4. Send metrics to monitoring
        if 'datadog' in pipeline_config:
            self.tools['datadog'].record_pipeline_duration(
                pipeline_config['name'],
                sum([r.get('duration', 0) for r in results.values()])
            )
        
        return results
```

### Configuration Management
```yaml
# config/tool_configurations.yml
tools:
  dbt:
    project_dir: "/opt/dbt"
    profiles_dir: "~/.dbt"
    target: "prod"
    
  airflow:
    dag_dir: "/opt/airflow/dags"
    conn_id_prefix: "maritime_"
    default_pool: "default_pool"
    
  fivetran:
    base_url: "https://api.fivetran.com/v1"
    rate_limit: 1000
    timeout: 30
    
  great_expectations:
    context_root_dir: "/opt/great_expectations"
    data_docs_sites:
      local_site:
        class_name: "SiteBuilder"
        base_directory: "/var/www/html/data_docs"
    
  snowflake:
    account: "${SNOWFLAKE_ACCOUNT}"
    warehouse: "COMPUTE_WH"
    role: "TRANSFORMER"
    
pipelines:
  customer_pipeline:
    schedule: "0 6 * * *"
    tools:
      fivetran:
        connectors: ["salesforce_prod", "postgres_crm"]
      dbt:
        models: "models.customer"
      great_expectations:
        checkpoints: ["customer_data_checkpoint"]
    notifications:
      slack: "#data-alerts"
      email: ["team@company.com"]
```

## Best Practices

### Tool Selection Criteria
1. **Integration Capability**: How well does it integrate with existing stack?
2. **Scalability**: Can it handle growing data volumes?
3. **Maintainability**: How easy is it to maintain and support?
4. **Cost**: What are the total ownership costs?
5. **Community**: Is there good community support and documentation?

### Automation Guidelines
1. **Start Simple**: Begin with basic automation, add complexity gradually
2. **Monitor Everything**: Instrument all automated processes
3. **Fail Fast**: Design for quick failure detection and recovery
4. **Version Control**: Keep all configurations in version control
5. **Test Thoroughly**: Implement comprehensive testing for automation

### Security Considerations
1. **Credential Management**: Use secure credential storage (e.g., HashiCorp Vault)
2. **Access Controls**: Implement principle of least privilege
3. **Audit Logging**: Log all automated actions
4. **Encryption**: Encrypt data in transit and at rest
5. **Regular Updates**: Keep all tools updated with security patches

---
**Document Control**
- Version: 1.0
- Created: [Date]
- Last Updated: [Date]
- Owner: Data Platform Team
- Next Review: [Date]