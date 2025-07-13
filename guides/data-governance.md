# Data Governance Guide

## Overview
This guide establishes comprehensive data governance practices for our data platform, ensuring data quality, security, compliance, and proper stewardship across the organization.

## Data Governance Framework

### Governance Structure

#### Data Governance Council
| Role | Responsibilities | Authority Level |
|------|-----------------|-----------------|
| **Chief Data Officer (CDO)** | Strategic direction, policy approval | Executive |
| **Data Governance Manager** | Program management, policy enforcement | Strategic |
| **Data Stewards** | Domain expertise, data quality | Tactical |
| **Data Custodians** | Technical implementation, maintenance | Operational |
| **Business Data Owners** | Business requirements, access decisions | Domain |

#### Governance Committees
```
Data Governance Council (Executive)
├── Data Quality Committee
├── Data Security Committee
├── Data Privacy Committee
└── Data Architecture Committee
```

### Roles and Responsibilities

#### Data Owner
**Responsibilities:**
- Define data usage policies and access controls
- Approve data sharing agreements
- Resolve data quality issues
- Make data-related business decisions

**Accountability:**
- Business value and outcomes from data
- Compliance with regulatory requirements
- Data-related risk management

#### Data Steward
**Responsibilities:**
- Monitor data quality metrics
- Implement data quality rules
- Coordinate with IT for technical solutions
- Train users on data standards

**Skills Required:**
- Deep domain knowledge
- Understanding of data flows
- Communication and problem-solving
- Basic technical knowledge

#### Data Custodian
**Responsibilities:**
- Implement technical data controls
- Maintain data infrastructure
- Execute data quality procedures
- Provide technical support

**Skills Required:**
- Technical expertise in data platforms
- Database administration
- Security implementation
- Automation and scripting

## Data Classification and Inventory

### Data Classification Schema
| Classification | Description | Examples | Access Controls |
|----------------|-------------|----------|-----------------|
| **Public** | Information that can be shared publicly | Marketing materials, public reports | No restrictions |
| **Internal** | Information for internal use only | Employee directories, internal metrics | Employee access only |
| **Confidential** | Sensitive business information | Financial data, strategic plans | Role-based access |
| **Restricted** | Highly sensitive, regulated data | PII, PHI, payment data | Strict access controls |

### Data Inventory Management
```python
# data_inventory_manager.py
import pandas as pd
from datetime import datetime
import json

class DataInventoryManager:
    def __init__(self, snowflake_connection):
        self.conn = snowflake_connection
        self.inventory = {}
    
    def discover_data_assets(self):
        """Automatically discover data assets in Snowflake"""
        
        discovery_query = """
        SELECT 
            table_catalog as database_name,
            table_schema as schema_name,
            table_name,
            table_type,
            row_count,
            bytes,
            created,
            last_altered,
            comment
        FROM information_schema.tables
        WHERE table_schema NOT IN ('INFORMATION_SCHEMA', 'ACCOUNT_USAGE')
        ORDER BY bytes DESC
        """
        
        cursor = self.conn.cursor()
        cursor.execute(discovery_query)
        results = cursor.fetchall()
        
        # Process results into inventory
        for row in results:
            asset_id = f"{row[0]}.{row[1]}.{row[2]}"
            
            self.inventory[asset_id] = {
                'database': row[0],
                'schema': row[1],
                'table': row[2],
                'type': row[3],
                'row_count': row[4],
                'size_bytes': row[5],
                'created_date': row[6],
                'last_modified': row[7],
                'description': row[8] or '',
                'classification': self.classify_data_asset(asset_id),
                'owner': self.identify_data_owner(asset_id),
                'steward': self.identify_data_steward(asset_id),
                'tags': self.extract_data_tags(asset_id),
                'lineage': self.discover_lineage(asset_id),
                'quality_score': self.calculate_quality_score(asset_id)
            }
    
    def classify_data_asset(self, asset_id):
        """Automatically classify data based on content analysis"""
        
        # Sample data for classification
        sample_query = f"""
        SELECT * FROM {asset_id} 
        SAMPLE (100 ROWS)
        """
        
        try:
            cursor = self.conn.cursor()
            cursor.execute(sample_query)
            sample_data = cursor.fetchall()
            columns = [desc[0] for desc in cursor.description]
            
            # Classification logic
            classification = 'Internal'  # Default
            
            # Check for PII indicators
            pii_indicators = ['ssn', 'social_security', 'email', 'phone', 'address', 'credit_card']
            if any(indicator in ' '.join(columns).lower() for indicator in pii_indicators):
                classification = 'Restricted'
            
            # Check for financial data
            financial_indicators = ['salary', 'revenue', 'profit', 'cost', 'price', 'amount']
            if any(indicator in ' '.join(columns).lower() for indicator in financial_indicators):
                classification = 'Confidential'
            
            return classification
            
        except Exception as e:
            print(f"Error classifying {asset_id}: {e}")
            return 'Internal'  # Safe default
    
    def generate_inventory_report(self):
        """Generate comprehensive data inventory report"""
        
        report = {
            'summary': {
                'total_assets': len(self.inventory),
                'classification_breakdown': self.get_classification_breakdown(),
                'size_breakdown': self.get_size_breakdown(),
                'quality_summary': self.get_quality_summary()
            },
            'assets': self.inventory,
            'generated_date': datetime.now().isoformat()
        }
        
        return report
    
    def export_to_data_catalog(self, catalog_system='snowflake'):
        """Export inventory to data catalog system"""
        
        if catalog_system == 'snowflake':
            self.update_snowflake_tags()
        elif catalog_system == 'aws_glue':
            self.update_glue_catalog()
        elif catalog_system == 'azure_purview':
            self.update_purview_catalog()
```

### Data Lineage Tracking
```sql
-- Create data lineage tracking tables
CREATE SCHEMA IF NOT EXISTS DATA_GOVERNANCE;

CREATE TABLE DATA_GOVERNANCE.DATA_LINEAGE (
    lineage_id VARCHAR(50) PRIMARY KEY,
    source_system VARCHAR(100),
    source_table VARCHAR(100),
    target_system VARCHAR(100),
    target_table VARCHAR(100),
    transformation_type VARCHAR(50),
    transformation_logic TEXT,
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    created_by VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE DATA_GOVERNANCE.DATA_TRANSFORMATIONS (
    transformation_id VARCHAR(50) PRIMARY KEY,
    lineage_id VARCHAR(50) REFERENCES DATA_LINEAGE(lineage_id),
    transformation_step INTEGER,
    transformation_description TEXT,
    sql_logic TEXT,
    business_rule TEXT
);

-- Automated lineage capture from query history
INSERT INTO DATA_GOVERNANCE.DATA_LINEAGE
SELECT 
    MD5(query_text) as lineage_id,
    'SNOWFLAKE' as source_system,
    REGEXP_SUBSTR(query_text, 'FROM\\s+(\\w+\\.\\w+\\.\\w+)', 1, 1, 'i', 1) as source_table,
    'SNOWFLAKE' as target_system,
    REGEXP_SUBSTR(query_text, 'INTO\\s+(\\w+\\.\\w+\\.\\w+)', 1, 1, 'i', 1) as target_table,
    'SQL_TRANSFORMATION' as transformation_type,
    query_text as transformation_logic,
    start_time as created_date,
    user_name as created_by,
    TRUE as is_active
FROM snowflake.account_usage.query_history
WHERE query_type = 'INSERT'
  AND query_text ILIKE '%INSERT INTO%'
  AND start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS';
```

## Data Quality Management

### Data Quality Framework

#### Quality Dimensions
| Dimension | Definition | Measurement | Target |
|-----------|------------|-------------|--------|
| **Completeness** | Presence of required data | % of non-null values | >95% |
| **Accuracy** | Correctness of data values | % of valid values | >98% |
| **Consistency** | Uniformity across systems | % of consistent records | >99% |
| **Timeliness** | Data freshness and currency | Hours since last update | <24 hours |
| **Validity** | Conformance to business rules | % of rule-compliant records | >97% |
| **Uniqueness** | Absence of duplicates | % of unique records | >99% |

#### Quality Rules Engine
```python
# data_quality_engine.py
import pandas as pd
from typing import Dict, List, Any
import re

class DataQualityEngine:
    def __init__(self, snowflake_connection):
        self.conn = snowflake_connection
        self.rules = {}
        self.results = {}
    
    def register_quality_rule(self, rule_id: str, rule_config: Dict[str, Any]):
        """Register a data quality rule"""
        
        self.rules[rule_id] = {
            'table': rule_config['table'],
            'column': rule_config.get('column'),
            'rule_type': rule_config['rule_type'],
            'rule_logic': rule_config['rule_logic'],
            'threshold': rule_config.get('threshold', 100),
            'severity': rule_config.get('severity', 'HIGH'),
            'owner': rule_config.get('owner'),
            'enabled': rule_config.get('enabled', True)
        }
    
    def execute_completeness_check(self, table_name: str, column_name: str) -> Dict[str, Any]:
        """Check data completeness"""
        
        query = f"""
        SELECT 
            COUNT(*) as total_records,
            COUNT({column_name}) as non_null_records,
            (COUNT({column_name}) / COUNT(*)) * 100 as completeness_percentage
        FROM {table_name}
        """
        
        cursor = self.conn.cursor()
        cursor.execute(query)
        result = cursor.fetchone()
        
        return {
            'rule_type': 'completeness',
            'table': table_name,
            'column': column_name,
            'total_records': result[0],
            'non_null_records': result[1],
            'completeness_percentage': result[2],
            'passed': result[2] >= 95.0  # 95% threshold
        }
    
    def execute_validity_check(self, table_name: str, column_name: str, pattern: str) -> Dict[str, Any]:
        """Check data validity using regex pattern"""
        
        query = f"""
        SELECT 
            COUNT(*) as total_records,
            COUNT(CASE WHEN REGEXP_LIKE({column_name}, '{pattern}') THEN 1 END) as valid_records,
            (COUNT(CASE WHEN REGEXP_LIKE({column_name}, '{pattern}') THEN 1 END) / COUNT(*)) * 100 as validity_percentage
        FROM {table_name}
        WHERE {column_name} IS NOT NULL
        """
        
        cursor = self.conn.cursor()
        cursor.execute(query)
        result = cursor.fetchone()
        
        return {
            'rule_type': 'validity',
            'table': table_name,
            'column': column_name,
            'pattern': pattern,
            'total_records': result[0],
            'valid_records': result[1],
            'validity_percentage': result[2],
            'passed': result[2] >= 98.0  # 98% threshold
        }
    
    def execute_uniqueness_check(self, table_name: str, column_name: str) -> Dict[str, Any]:
        """Check data uniqueness"""
        
        query = f"""
        SELECT 
            COUNT(*) as total_records,
            COUNT(DISTINCT {column_name}) as unique_records,
            (COUNT(DISTINCT {column_name}) / COUNT(*)) * 100 as uniqueness_percentage
        FROM {table_name}
        WHERE {column_name} IS NOT NULL
        """
        
        cursor = self.conn.cursor()
        cursor.execute(query)
        result = cursor.fetchone()
        
        return {
            'rule_type': 'uniqueness',
            'table': table_name,
            'column': column_name,
            'total_records': result[0],
            'unique_records': result[1],
            'uniqueness_percentage': result[2],
            'passed': result[2] >= 99.0  # 99% threshold
        }
    
    def execute_consistency_check(self, primary_table: str, reference_table: str, join_key: str) -> Dict[str, Any]:
        """Check consistency between tables"""
        
        query = f"""
        WITH consistency_check AS (
            SELECT 
                p.{join_key},
                CASE WHEN r.{join_key} IS NOT NULL THEN 1 ELSE 0 END as is_consistent
            FROM {primary_table} p
            LEFT JOIN {reference_table} r ON p.{join_key} = r.{join_key}
        )
        SELECT 
            COUNT(*) as total_records,
            SUM(is_consistent) as consistent_records,
            (SUM(is_consistent) / COUNT(*)) * 100 as consistency_percentage
        FROM consistency_check
        """
        
        cursor = self.conn.cursor()
        cursor.execute(query)
        result = cursor.fetchone()
        
        return {
            'rule_type': 'consistency',
            'primary_table': primary_table,
            'reference_table': reference_table,
            'join_key': join_key,
            'total_records': result[0],
            'consistent_records': result[1],
            'consistency_percentage': result[2],
            'passed': result[2] >= 99.0  # 99% threshold
        }
    
    def execute_all_rules(self) -> Dict[str, Any]:
        """Execute all registered quality rules"""
        
        results = {}
        
        for rule_id, rule_config in self.rules.items():
            if not rule_config['enabled']:
                continue
                
            try:
                if rule_config['rule_type'] == 'completeness':
                    result = self.execute_completeness_check(
                        rule_config['table'], 
                        rule_config['column']
                    )
                elif rule_config['rule_type'] == 'validity':
                    result = self.execute_validity_check(
                        rule_config['table'], 
                        rule_config['column'],
                        rule_config['rule_logic']
                    )
                elif rule_config['rule_type'] == 'uniqueness':
                    result = self.execute_uniqueness_check(
                        rule_config['table'], 
                        rule_config['column']
                    )
                elif rule_config['rule_type'] == 'consistency':
                    result = self.execute_consistency_check(
                        rule_config['table'], 
                        rule_config['rule_logic']['reference_table'],
                        rule_config['rule_logic']['join_key']
                    )
                
                result['rule_id'] = rule_id
                result['severity'] = rule_config['severity']
                result['owner'] = rule_config['owner']
                result['execution_time'] = datetime.now().isoformat()
                
                results[rule_id] = result
                
            except Exception as e:
                results[rule_id] = {
                    'rule_id': rule_id,
                    'error': str(e),
                    'passed': False,
                    'execution_time': datetime.now().isoformat()
                }
        
        return results
    
    def generate_quality_report(self) -> str:
        """Generate data quality report"""
        
        results = self.execute_all_rules()
        
        # Calculate overall quality score
        passed_rules = sum(1 for r in results.values() if r.get('passed', False))
        total_rules = len(results)
        overall_score = (passed_rules / total_rules * 100) if total_rules > 0 else 0
        
        # Generate report
        report = f"""
# Data Quality Report
Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

## Summary
- Overall Quality Score: {overall_score:.1f}%
- Rules Passed: {passed_rules}/{total_rules}
- Critical Issues: {sum(1 for r in results.values() if r.get('severity') == 'CRITICAL' and not r.get('passed', True))}

## Detailed Results
"""
        
        for rule_id, result in results.items():
            status = "✅ PASSED" if result.get('passed', False) else "❌ FAILED"
            report += f"""
### {rule_id}
- Status: {status}
- Type: {result.get('rule_type', 'Unknown')}
- Table: {result.get('table', 'Unknown')}
- Severity: {result.get('severity', 'Unknown')}
"""
            
            if 'error' in result:
                report += f"- Error: {result['error']}\n"
            else:
                if 'completeness_percentage' in result:
                    report += f"- Completeness: {result['completeness_percentage']:.2f}%\n"
                if 'validity_percentage' in result:
                    report += f"- Validity: {result['validity_percentage']:.2f}%\n"
                if 'uniqueness_percentage' in result:
                    report += f"- Uniqueness: {result['uniqueness_percentage']:.2f}%\n"
                if 'consistency_percentage' in result:
                    report += f"- Consistency: {result['consistency_percentage']:.2f}%\n"
        
        return report
```

### Quality Monitoring Dashboard
```sql
-- Create quality monitoring views
CREATE OR REPLACE VIEW DATA_GOVERNANCE.QUALITY_DASHBOARD AS
WITH daily_quality_metrics AS (
    SELECT 
        table_name,
        check_date,
        rule_type,
        AVG(quality_score) as avg_quality_score,
        COUNT(*) as checks_performed,
        SUM(CASE WHEN passed = TRUE THEN 1 ELSE 0 END) as checks_passed
    FROM DATA_GOVERNANCE.QUALITY_CHECK_RESULTS
    WHERE check_date >= DATEADD(day, -30, CURRENT_DATE())
    GROUP BY table_name, check_date, rule_type
)
SELECT 
    table_name,
    rule_type,
    AVG(avg_quality_score) as monthly_avg_score,
    SUM(checks_performed) as total_checks,
    SUM(checks_passed) as total_passed,
    (SUM(checks_passed) / SUM(checks_performed)) * 100 as pass_rate
FROM daily_quality_metrics
GROUP BY table_name, rule_type
ORDER BY monthly_avg_score DESC;

-- Quality trend analysis
CREATE OR REPLACE VIEW DATA_GOVERNANCE.QUALITY_TRENDS AS
SELECT 
    table_name,
    DATE_TRUNC('week', check_date) as week_start,
    AVG(quality_score) as weekly_avg_score,
    LAG(AVG(quality_score)) OVER (PARTITION BY table_name ORDER BY week_start) as previous_week_score,
    (AVG(quality_score) - LAG(AVG(quality_score)) OVER (PARTITION BY table_name ORDER BY week_start)) as score_change
FROM DATA_GOVERNANCE.QUALITY_CHECK_RESULTS
WHERE check_date >= DATEADD(week, -12, CURRENT_DATE())
GROUP BY table_name, week_start
ORDER BY table_name, week_start;
```

## Data Security and Privacy

### Security Framework

#### Access Control Matrix
| Role | Public Data | Internal Data | Confidential Data | Restricted Data |
|------|-------------|---------------|-------------------|-----------------|
| **Guest** | Read | No Access | No Access | No Access |
| **Employee** | Read/Write | Read | No Access | No Access |
| **Manager** | Read/Write | Read/Write | Read | No Access |
| **Executive** | Read/Write | Read/Write | Read/Write | Read |
| **Admin** | Full Access | Full Access | Full Access | Read/Write |

#### Data Masking Policies
```sql
-- Email masking policy
CREATE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
  CASE 
    WHEN CURRENT_ROLE() IN ('ADMIN', 'DPO', 'CUSTOMER_SERVICE') THEN val
    WHEN CURRENT_ROLE() IN ('ANALYST', 'MANAGER') THEN 
      REGEXP_REPLACE(val, '(.{2})[^@]*(@.*)', '\\1***\\2')
    ELSE '***@***.com'
  END;

-- Apply to customer table
ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY email_mask;

-- SSN masking policy
CREATE MASKING POLICY ssn_mask AS (val STRING) RETURNS STRING ->
  CASE 
    WHEN CURRENT_ROLE() IN ('ADMIN', 'COMPLIANCE_OFFICER') THEN val
    WHEN CURRENT_ROLE() IN ('CUSTOMER_SERVICE') THEN 
      CONCAT('***-**-', RIGHT(val, 4))
    ELSE '***-**-****'
  END;

-- Phone number masking policy
CREATE MASKING POLICY phone_mask AS (val STRING) RETURNS STRING ->
  CASE 
    WHEN CURRENT_ROLE() IN ('ADMIN', 'SALES', 'CUSTOMER_SERVICE') THEN val
    WHEN CURRENT_ROLE() IN ('ANALYST') THEN 
      CONCAT(LEFT(val, 3), '-***-', RIGHT(val, 4))
    ELSE '***-***-****'
  END;
```

#### Row-Level Security
```sql
-- Region-based row access policy
CREATE ROW ACCESS POLICY region_access AS (user_region STRING) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('ADMIN', 'GLOBAL_MANAGER') 
  OR user_region = CURRENT_USER_REGION()
  OR user_region IS NULL;

-- Apply to customer data
ALTER TABLE customers ADD ROW ACCESS POLICY region_access ON (region);

-- Department-based access policy
CREATE ROW ACCESS POLICY department_access AS (department STRING) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('ADMIN', 'HR_MANAGER')
  OR department = CURRENT_USER_DEPARTMENT()
  OR (CURRENT_ROLE() = 'MANAGER' AND department IN 
      (SELECT managed_department FROM user_department_mapping 
       WHERE user_name = CURRENT_USER()));

-- Time-based access policy for sensitive data
CREATE ROW ACCESS POLICY time_sensitive_access AS (record_date DATE) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('ADMIN', 'COMPLIANCE')
  OR record_date >= DATEADD(year, -2, CURRENT_DATE())  -- Only recent data for regular users
  OR (CURRENT_ROLE() = 'ANALYST' AND record_date >= DATEADD(year, -5, CURRENT_DATE()));
```

### Privacy Management
```python
# privacy_manager.py
import hashlib
import uuid
from datetime import datetime, timedelta

class PrivacyManager:
    def __init__(self, snowflake_connection):
        self.conn = snowflake_connection
        
    def anonymize_dataset(self, table_name: str, pii_columns: List[str], method: str = 'hash'):
        """Anonymize PII data in a dataset"""
        
        if method == 'hash':
            # Create anonymized version using SHA-256 hashing
            anonymization_sql = f"""
            CREATE OR REPLACE TABLE {table_name}_anonymized AS
            SELECT 
                *,
                {', '.join([f"SHA2({col}) as {col}_hash" for col in pii_columns])}
            FROM {table_name}
            """
            
            # Remove original PII columns
            for col in pii_columns:
                anonymization_sql += f", {col} as {col}_original"
            
        elif method == 'synthetic':
            # Generate synthetic data while preserving statistical properties
            anonymization_sql = self.generate_synthetic_data_sql(table_name, pii_columns)
        
        cursor = self.conn.cursor()
        cursor.execute(anonymization_sql)
        
        # Log anonymization activity
        self.log_privacy_action('anonymization', table_name, pii_columns)
    
    def handle_data_subject_request(self, request_type: str, subject_id: str, table_name: str):
        """Handle GDPR/CCPA data subject requests"""
        
        if request_type == 'access':
            # Right to access - export all data for subject
            return self.export_subject_data(subject_id, table_name)
            
        elif request_type == 'deletion':
            # Right to erasure - delete all data for subject
            return self.delete_subject_data(subject_id, table_name)
            
        elif request_type == 'rectification':
            # Right to rectification - allow data correction
            return self.enable_data_correction(subject_id, table_name)
            
        elif request_type == 'portability':
            # Right to data portability - export in machine-readable format
            return self.export_portable_data(subject_id, table_name)
    
    def scan_for_pii(self, table_name: str) -> List[str]:
        """Automatically scan table for potential PII"""
        
        # Get column information
        column_query = f"""
        SELECT column_name, data_type, comment
        FROM information_schema.columns 
        WHERE table_name = '{table_name.upper()}'
        """
        
        cursor = self.conn.cursor()
        cursor.execute(column_query)
        columns = cursor.fetchall()
        
        pii_indicators = [
            'email', 'phone', 'ssn', 'social_security', 'address', 
            'first_name', 'last_name', 'full_name', 'credit_card',
            'passport', 'driver_license', 'birth_date', 'birthday'
        ]
        
        potential_pii = []
        for col_name, data_type, comment in columns:
            col_name_lower = col_name.lower()
            comment_lower = (comment or '').lower()
            
            # Check column name and comments for PII indicators
            if any(indicator in col_name_lower or indicator in comment_lower 
                   for indicator in pii_indicators):
                potential_pii.append(col_name)
        
        return potential_pii
    
    def create_consent_tracking(self):
        """Create consent tracking infrastructure"""
        
        consent_schema = """
        CREATE SCHEMA IF NOT EXISTS PRIVACY_MANAGEMENT;
        
        CREATE TABLE PRIVACY_MANAGEMENT.CONSENT_RECORDS (
            consent_id VARCHAR(50) PRIMARY KEY,
            subject_id VARCHAR(100) NOT NULL,
            purpose VARCHAR(200) NOT NULL,
            consent_given BOOLEAN NOT NULL,
            consent_date TIMESTAMP_NTZ NOT NULL,
            expiry_date TIMESTAMP_NTZ,
            withdrawal_date TIMESTAMP_NTZ,
            legal_basis VARCHAR(100),
            data_categories VARIANT,
            consent_method VARCHAR(50),
            created_by VARCHAR(100),
            created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
        );
        
        CREATE TABLE PRIVACY_MANAGEMENT.DATA_PROCESSING_LOG (
            log_id VARCHAR(50) PRIMARY KEY,
            subject_id VARCHAR(100),
            processing_purpose VARCHAR(200),
            data_categories VARIANT,
            processing_date TIMESTAMP_NTZ,
            legal_basis VARCHAR(100),
            processor VARCHAR(100),
            retention_period INTEGER,
            created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
        );
        
        CREATE TABLE PRIVACY_MANAGEMENT.SUBJECT_REQUESTS (
            request_id VARCHAR(50) PRIMARY KEY,
            subject_id VARCHAR(100) NOT NULL,
            request_type VARCHAR(50) NOT NULL,
            request_date TIMESTAMP_NTZ NOT NULL,
            status VARCHAR(50) DEFAULT 'PENDING',
            completion_date TIMESTAMP_NTZ,
            requester_identity_verified BOOLEAN DEFAULT FALSE,
            notes TEXT,
            created_by VARCHAR(100),
            created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
        );
        """
        
        cursor = self.conn.cursor()
        cursor.execute(consent_schema)
```

## Data Compliance and Regulations

### Regulatory Compliance Framework

#### GDPR Compliance
| Requirement | Implementation | Validation |
|-------------|----------------|------------|
| **Lawful Basis** | Consent tracking system | Regular audits |
| **Data Minimization** | Automated data retention | Quarterly reviews |
| **Right to Access** | Subject data export tools | Response time tracking |
| **Right to Erasure** | Automated deletion procedures | Deletion verification |
| **Data Portability** | Standardized export formats | Format validation |
| **Privacy by Design** | Default privacy settings | Design reviews |

#### CCPA Compliance
```sql
-- CCPA compliance tracking
CREATE TABLE PRIVACY_MANAGEMENT.CCPA_COMPLIANCE (
    compliance_check_id VARCHAR(50) PRIMARY KEY,
    check_date DATE NOT NULL,
    requirement VARCHAR(200) NOT NULL,
    status VARCHAR(50) NOT NULL,
    evidence TEXT,
    responsible_party VARCHAR(100),
    next_review_date DATE,
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Insert compliance requirements
INSERT INTO PRIVACY_MANAGEMENT.CCPA_COMPLIANCE VALUES
('CCPA-001', CURRENT_DATE(), 'Right to Know Categories', 'COMPLIANT', 'Data catalog implemented', 'Data Protection Officer', DATEADD(month, 3, CURRENT_DATE()), CURRENT_TIMESTAMP()),
('CCPA-002', CURRENT_DATE(), 'Right to Delete', 'COMPLIANT', 'Deletion procedures automated', 'Data Protection Officer', DATEADD(month, 3, CURRENT_DATE()), CURRENT_TIMESTAMP()),
('CCPA-003', CURRENT_DATE(), 'Right to Opt-Out', 'IN_PROGRESS', 'Opt-out mechanisms being implemented', 'Privacy Team', DATEADD(month, 1, CURRENT_DATE()), CURRENT_TIMESTAMP());
```

#### SOX Compliance for Financial Data
```sql
-- SOX compliance controls
CREATE TABLE DATA_GOVERNANCE.SOX_CONTROLS (
    control_id VARCHAR(50) PRIMARY KEY,
    control_description TEXT NOT NULL,
    control_type VARCHAR(50), -- PREVENTIVE, DETECTIVE, CORRECTIVE
    frequency VARCHAR(50),     -- DAILY, WEEKLY, MONTHLY, QUARTERLY
    owner VARCHAR(100),
    last_testing_date DATE,
    testing_result VARCHAR(50),
    deficiencies TEXT,
    remediation_plan TEXT,
    next_testing_date DATE
);

-- Implement automated SOX controls
CREATE OR REPLACE PROCEDURE implement_sox_controls()
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    -- Control 1: Financial data access logging
    CREATE OR REPLACE VIEW SOX_ACCESS_LOG AS
    SELECT 
        user_name,
        query_text,
        start_time,
        execution_status,
        database_name,
        schema_name
    FROM snowflake.account_usage.query_history
    WHERE database_name = 'FINANCIAL_DATA'
      AND start_time >= DATEADD(day, -90, CURRENT_TIMESTAMP());
    
    -- Control 2: Segregation of duties validation
    CREATE OR REPLACE TASK sox_segregation_check
    WAREHOUSE = GOVERNANCE_WH
    SCHEDULE = 'USING CRON 0 9 * * 1'  -- Weekly Monday 9 AM
    AS
    INSERT INTO DATA_GOVERNANCE.SOX_VIOLATIONS
    SELECT 
        user_name,
        'SEGREGATION_VIOLATION' as violation_type,
        'User has both read and write access to financial data' as description,
        CURRENT_TIMESTAMP() as detected_date
    FROM (
        SELECT user_name
        FROM snowflake.account_usage.grants_to_users
        WHERE privilege IN ('SELECT', 'INSERT', 'UPDATE', 'DELETE')
          AND granted_on = 'TABLE'
          AND name LIKE 'FINANCIAL_DATA.%'
        GROUP BY user_name
        HAVING COUNT(DISTINCT privilege) > 1
    );
    
    RETURN 'SOX controls implemented successfully';
END;
$$;
```

### Audit and Monitoring
```python
# compliance_auditor.py
from datetime import datetime, timedelta
import json

class ComplianceAuditor:
    def __init__(self, snowflake_connection):
        self.conn = snowflake_connection
    
    def run_gdpr_audit(self) -> Dict[str, Any]:
        """Run comprehensive GDPR compliance audit"""
        
        audit_results = {
            'audit_date': datetime.now().isoformat(),
            'compliance_score': 0,
            'findings': [],
            'recommendations': []
        }
        
        # Check 1: Consent tracking completeness
        consent_check = self.check_consent_completeness()
        audit_results['findings'].append(consent_check)
        
        # Check 2: Data retention compliance
        retention_check = self.check_data_retention_compliance()
        audit_results['findings'].append(retention_check)
        
        # Check 3: Subject rights implementation
        rights_check = self.check_subject_rights_implementation()
        audit_results['findings'].append(rights_check)
        
        # Check 4: Data processing log completeness
        processing_check = self.check_processing_log_completeness()
        audit_results['findings'].append(processing_check)
        
        # Calculate overall compliance score
        total_checks = len(audit_results['findings'])
        passed_checks = sum(1 for finding in audit_results['findings'] 
                          if finding.get('status') == 'COMPLIANT')
        audit_results['compliance_score'] = (passed_checks / total_checks) * 100
        
        return audit_results
    
    def check_consent_completeness(self) -> Dict[str, Any]:
        """Check if consent records are complete for all data subjects"""
        
        query = """
        WITH subjects_with_data AS (
            SELECT DISTINCT customer_id as subject_id 
            FROM customers
            WHERE created_date >= DATEADD(month, -12, CURRENT_DATE())
        ),
        subjects_with_consent AS (
            SELECT DISTINCT subject_id 
            FROM PRIVACY_MANAGEMENT.CONSENT_RECORDS
            WHERE consent_given = TRUE
              AND (expiry_date IS NULL OR expiry_date > CURRENT_DATE())
              AND withdrawal_date IS NULL
        )
        SELECT 
            COUNT(*) as total_subjects,
            COUNT(c.subject_id) as subjects_with_consent,
            (COUNT(c.subject_id) / COUNT(*)) * 100 as consent_coverage_percentage
        FROM subjects_with_data s
        LEFT JOIN subjects_with_consent c ON s.subject_id = c.subject_id
        """
        
        cursor = self.conn.cursor()
        cursor.execute(query)
        result = cursor.fetchone()
        
        status = 'COMPLIANT' if result[2] >= 95 else 'NON_COMPLIANT'
        
        return {
            'check_name': 'Consent Completeness',
            'status': status,
            'details': {
                'total_subjects': result[0],
                'subjects_with_consent': result[1],
                'coverage_percentage': result[2]
            },
            'recommendation': 'Ensure all data subjects have valid consent records' if status == 'NON_COMPLIANT' else None
        }
    
    def generate_compliance_report(self) -> str:
        """Generate comprehensive compliance report"""
        
        gdpr_audit = self.run_gdpr_audit()
        
        report = f"""
# Data Compliance Audit Report
Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

## Executive Summary
- Overall GDPR Compliance Score: {gdpr_audit['compliance_score']:.1f}%
- Total Findings: {len(gdpr_audit['findings'])}
- Critical Issues: {sum(1 for f in gdpr_audit['findings'] if f['status'] == 'NON_COMPLIANT')}

## Detailed Findings
"""
        
        for finding in gdpr_audit['findings']:
            status_emoji = "✅" if finding['status'] == 'COMPLIANT' else "❌"
            report += f"""
### {finding['check_name']} {status_emoji}
Status: {finding['status']}
"""
            if 'details' in finding:
                for key, value in finding['details'].items():
                    report += f"- {key.replace('_', ' ').title()}: {value}\n"
            
            if finding.get('recommendation'):
                report += f"**Recommendation**: {finding['recommendation']}\n"
        
        return report
```

## Metadata Management

### Metadata Repository
```sql
-- Create comprehensive metadata schema
CREATE SCHEMA IF NOT EXISTS METADATA_MANAGEMENT;

CREATE TABLE METADATA_MANAGEMENT.BUSINESS_GLOSSARY (
    term_id VARCHAR(50) PRIMARY KEY,
    term_name VARCHAR(200) NOT NULL,
    definition TEXT NOT NULL,
    context TEXT,
    synonyms VARIANT,
    related_terms VARIANT,
    data_domain VARCHAR(100),
    business_owner VARCHAR(100),
    subject_matter_expert VARCHAR(100),
    approval_status VARCHAR(50) DEFAULT 'DRAFT',
    approved_by VARCHAR(100),
    approved_date DATE,
    created_by VARCHAR(100),
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    updated_by VARCHAR(100),
    updated_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

CREATE TABLE METADATA_MANAGEMENT.DATA_DICTIONARY (
    column_id VARCHAR(50) PRIMARY KEY,
    database_name VARCHAR(100),
    schema_name VARCHAR(100),
    table_name VARCHAR(100),
    column_name VARCHAR(100),
    business_name VARCHAR(200),
    description TEXT,
    data_type VARCHAR(50),
    is_nullable BOOLEAN,
    default_value VARCHAR(500),
    business_rules TEXT,
    data_quality_rules VARIANT,
    glossary_term_id VARCHAR(50) REFERENCES BUSINESS_GLOSSARY(term_id),
    classification VARCHAR(50),
    sensitivity_level VARCHAR(50),
    retention_period INTEGER,
    data_steward VARCHAR(100),
    created_by VARCHAR(100),
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    updated_by VARCHAR(100),
    updated_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Populate business glossary
INSERT INTO METADATA_MANAGEMENT.BUSINESS_GLOSSARY VALUES
('CUST-001', 'Customer', 'An individual or organization that purchases goods or services', 'Used across all customer-facing systems', '["Client", "Account"]', '["Prospect", "Lead"]', 'Customer', 'VP Sales', 'Customer Success Manager', 'APPROVED', 'CDO', CURRENT_DATE(), 'Data Steward', CURRENT_TIMESTAMP(), NULL, NULL),
('ORD-001', 'Order', 'A request to purchase products or services', 'Includes both online and offline orders', '["Purchase", "Transaction"]', '["Invoice", "Receipt"]', 'Sales', 'VP Sales', 'Order Management Lead', 'APPROVED', 'CDO', CURRENT_DATE(), 'Data Steward', CURRENT_TIMESTAMP(), NULL, NULL),
('REV-001', 'Revenue', 'Income generated from business operations', 'Recognized according to GAAP principles', '["Income", "Sales"]', '["Profit", "Earnings"]', 'Finance', 'CFO', 'Finance Director', 'APPROVED', 'CDO', CURRENT_DATE(), 'Data Steward', CURRENT_TIMESTAMP(), NULL, NULL);
```

### Automated Metadata Discovery
```python
# metadata_discovery.py
import re
from typing import Dict, List, Any

class MetadataDiscovery:
    def __init__(self, snowflake_connection):
        self.conn = snowflake_connection
        
    def discover_table_metadata(self, database_name: str, schema_name: str) -> List[Dict[str, Any]]:
        """Discover and enrich table metadata"""
        
        # Get basic table information
        table_query = """
        SELECT 
            table_name,
            table_type,
            row_count,
            bytes,
            created,
            last_altered,
            comment
        FROM information_schema.tables
        WHERE table_catalog = %s 
          AND table_schema = %s
        ORDER BY table_name
        """
        
        cursor = self.conn.cursor()
        cursor.execute(table_query, (database_name, schema_name))
        tables = cursor.fetchall()
        
        enriched_metadata = []
        
        for table in tables:
            table_name = table[0]
            
            # Get column information
            columns = self.discover_column_metadata(database_name, schema_name, table_name)
            
            # Infer business purpose
            business_purpose = self.infer_business_purpose(table_name, columns)
            
            # Detect data patterns
            data_patterns = self.detect_data_patterns(database_name, schema_name, table_name)
            
            # Suggest data classification
            suggested_classification = self.suggest_classification(columns, data_patterns)
            
            enriched_metadata.append({
                'database': database_name,
                'schema': schema_name,
                'table_name': table_name,
                'table_type': table[1],
                'row_count': table[2],
                'size_bytes': table[3],
                'created_date': table[4],
                'last_modified': table[5],
                'existing_comment': table[6],
                'columns': columns,
                'inferred_business_purpose': business_purpose,
                'data_patterns': data_patterns,
                'suggested_classification': suggested_classification,
                'suggested_steward': self.suggest_data_steward(table_name, business_purpose)
            })
        
        return enriched_metadata
    
    def discover_column_metadata(self, database_name: str, schema_name: str, table_name: str) -> List[Dict[str, Any]]:
        """Discover detailed column metadata"""
        
        column_query = """
        SELECT 
            column_name,
            data_type,
            is_nullable,
            column_default,
            comment,
            ordinal_position
        FROM information_schema.columns
        WHERE table_catalog = %s 
          AND table_schema = %s 
          AND table_name = %s
        ORDER BY ordinal_position
        """
        
        cursor = self.conn.cursor()
        cursor.execute(column_query, (database_name, schema_name, table_name))
        columns = cursor.fetchall()
        
        enriched_columns = []
        
        for column in columns:
            column_name = column[0]
            
            # Analyze column content for patterns
            sample_data = self.get_column_sample(database_name, schema_name, table_name, column_name)
            
            enriched_columns.append({
                'column_name': column_name,
                'data_type': column[1],
                'is_nullable': column[2],
                'default_value': column[3],
                'existing_comment': column[4],
                'position': column[5],
                'sample_values': sample_data['samples'],
                'null_percentage': sample_data['null_percentage'],
                'unique_values': sample_data['unique_count'],
                'suggested_business_name': self.suggest_business_name(column_name),
                'detected_patterns': self.detect_column_patterns(column_name, sample_data['samples']),
                'suggested_classification': self.classify_column(column_name, sample_data['samples'])
            })
        
        return enriched_columns
    
    def infer_business_purpose(self, table_name: str, columns: List[Dict[str, Any]]) -> str:
        """Infer business purpose from table name and columns"""
        
        table_name_lower = table_name.lower()
        column_names = [col['column_name'].lower() for col in columns]
        
        # Business domain patterns
        domain_patterns = {
            'customer': ['customer', 'client', 'account', 'contact'],
            'order': ['order', 'purchase', 'transaction', 'sale'],
            'product': ['product', 'item', 'inventory', 'catalog'],
            'employee': ['employee', 'staff', 'personnel', 'user'],
            'finance': ['payment', 'invoice', 'billing', 'revenue', 'cost'],
            'marketing': ['campaign', 'lead', 'prospect', 'marketing'],
            'logistics': ['shipment', 'delivery', 'warehouse', 'location']
        }
        
        # Check table name against patterns
        for domain, patterns in domain_patterns.items():
            if any(pattern in table_name_lower for pattern in patterns):
                return f"Manages {domain} data"
        
        # Check column names for clues
        for domain, patterns in domain_patterns.items():
            column_matches = sum(1 for col in column_names 
                               if any(pattern in col for pattern in patterns))
            if column_matches >= 2:  # Multiple columns suggest this domain
                return f"Stores {domain}-related information"
        
        return "Purpose to be determined"
    
    def suggest_data_steward(self, table_name: str, business_purpose: str) -> str:
        """Suggest appropriate data steward based on table characteristics"""
        
        steward_mapping = {
            'customer': 'Customer Success Manager',
            'order': 'Sales Operations Manager',
            'product': 'Product Manager',
            'employee': 'HR Manager',
            'finance': 'Finance Manager',
            'marketing': 'Marketing Manager',
            'logistics': 'Operations Manager'
        }
        
        purpose_lower = business_purpose.lower()
        
        for domain, steward in steward_mapping.items():
            if domain in purpose_lower:
                return steward
        
        return 'Data Governance Team'
    
    def generate_metadata_documentation(self, metadata: List[Dict[str, Any]]) -> str:
        """Generate comprehensive metadata documentation"""
        
        doc = f"""
# Data Dictionary - {metadata[0]['database']}.{metadata[0]['schema']}
Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

## Schema Overview
- Database: {metadata[0]['database']}
- Schema: {metadata[0]['schema']}
- Total Tables: {len(metadata)}
- Total Columns: {sum(len(table['columns']) for table in metadata)}

## Table Inventory
"""
        
        for table in metadata:
            doc += f"""
### {table['table_name']}
**Purpose**: {table['inferred_business_purpose']}
**Suggested Steward**: {table['suggested_steward']}
**Classification**: {table['suggested_classification']}
**Rows**: {table['row_count']:,}
**Size**: {table['size_bytes'] / (1024*1024*1024):.2f} GB

#### Columns
| Column | Type | Business Name | Classification | Description |
|--------|------|---------------|----------------|-------------|
"""
            
            for column in table['columns']:
                doc += f"| {column['column_name']} | {column['data_type']} | {column['suggested_business_name']} | {column['suggested_classification']} | {column['existing_comment'] or 'TBD'} |\n"
        
        return doc
```

## Implementation Roadmap

### Phase 1: Foundation (Months 1-3)
- [ ] Establish governance structure and roles
- [ ] Create data classification schema
- [ ] Implement basic data inventory
- [ ] Set up security policies and access controls
- [ ] Create compliance tracking framework

### Phase 2: Quality and Metadata (Months 4-6)
- [ ] Deploy data quality monitoring
- [ ] Implement automated quality checks
- [ ] Build metadata repository
- [ ] Create business glossary
- [ ] Establish data lineage tracking

### Phase 3: Privacy and Compliance (Months 7-9)
- [ ] Implement privacy controls and masking
- [ ] Set up consent management
- [ ] Deploy compliance monitoring
- [ ] Create audit trails
- [ ] Establish subject rights procedures

### Phase 4: Optimization and Automation (Months 10-12)
- [ ] Automate governance processes
- [ ] Implement self-service capabilities
- [ ] Create advanced analytics on governance metrics
- [ ] Establish continuous improvement processes
- [ ] Conduct governance maturity assessment

---
**Document Control**
- Version: 1.0
- Created: [Date]
- Last Updated: [Date]
- Author: Data Governance Team
- Reviewers: [Names]
- Next Review: [Date]