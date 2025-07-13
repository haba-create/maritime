# CRM Integration Design

## Overview
This document outlines the bi-directional integration between the customer data platform and CRM systems.

## Integration Architecture

### Integration Flow
```
┌─────────────────┐          ┌──────────────────┐          ┌─────────────────┐
│  Data Platform  │ <------> │  Integration Hub │ <------> │   CRM System    │
│   (Snowflake)   │          │    (API/ETL)     │          │  (Salesforce)   │
└─────────────────┘          └──────────────────┘          └─────────────────┘
```

## Inbound Integration (CRM → Data Platform)

### Real-time Integration
```python
# Webhook Handler for Real-time Updates
@app.route('/webhooks/crm/customer', methods=['POST'])
def handle_crm_webhook():
    """Process real-time customer updates from CRM"""
    
    payload = request.get_json()
    
    # Validate webhook signature
    if not validate_webhook_signature(request):
        return {'error': 'Invalid signature'}, 401
    
    # Process based on event type
    event_type = payload.get('event_type')
    
    if event_type == 'customer.created':
        process_new_customer(payload['data'])
    elif event_type == 'customer.updated':
        process_customer_update(payload['data'])
    elif event_type == 'customer.deleted':
        process_customer_deletion(payload['data'])
    
    # Queue for batch processing
    queue_for_snowflake(payload)
    
    return {'status': 'accepted'}, 202
```

### Batch Integration
```sql
-- Daily CRM Data Sync
CREATE OR REPLACE PROCEDURE sync_crm_customers()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
  -- Extract from CRM staging
  CREATE OR REPLACE TEMPORARY TABLE tmp_crm_extract AS
  SELECT 
    Id as crm_id,
    FirstName as first_name,
    LastName as last_name,
    Email as email,
    Phone as phone,
    AccountId as account_id,
    CreatedDate as created_date,
    LastModifiedDate as modified_date,
    IsDeleted as is_deleted
  FROM CRM_STAGING.SALESFORCE.CONTACT
  WHERE LastModifiedDate > (
    SELECT MAX(last_sync_timestamp) 
    FROM ETL_CONTROL.SYNC_LOG 
    WHERE source_system = 'SALESFORCE'
  );
  
  -- Apply transformations
  INSERT INTO RAW.CRM.CUSTOMERS
  SELECT 
    crm_id,
    first_name,
    last_name,
    email,
    phone,
    account_id,
    created_date,
    modified_date,
    is_deleted,
    CURRENT_TIMESTAMP() as extracted_timestamp
  FROM tmp_crm_extract;
  
  -- Log sync completion
  INSERT INTO ETL_CONTROL.SYNC_LOG
  VALUES ('SALESFORCE', 'CUSTOMERS', CURRENT_TIMESTAMP(), 
          (SELECT COUNT(*) FROM tmp_crm_extract));
  
  RETURN 'Sync completed successfully';
END;
$$;
```

## Outbound Integration (Data Platform → CRM)

### API Integration Pattern
```python
# Snowflake to CRM Sync
class CRMSyncService:
    def __init__(self, crm_config, snowflake_config):
        self.crm = CRMClient(crm_config)
        self.snowflake = SnowflakeClient(snowflake_config)
    
    def sync_customer_segments(self):
        """Sync customer segments from Snowflake to CRM"""
        
        # Get updated segments from Snowflake
        query = """
        SELECT 
            customer_id,
            crm_id,
            customer_segment,
            lifetime_value,
            churn_risk_score,
            last_activity_date
        FROM PRESENTATION.CUSTOMER_360
        WHERE segment_updated_date > DATEADD(hour, -1, CURRENT_TIMESTAMP())
        """
        
        customers = self.snowflake.execute_query(query)
        
        # Batch update to CRM
        batch_size = 200
        for i in range(0, len(customers), batch_size):
            batch = customers[i:i+batch_size]
            
            # Prepare CRM update payload
            updates = []
            for customer in batch:
                updates.append({
                    'Id': customer['crm_id'],
                    'Customer_Segment__c': customer['customer_segment'],
                    'Lifetime_Value__c': customer['lifetime_value'],
                    'Churn_Risk__c': customer['churn_risk_score'],
                    'Last_Platform_Activity__c': customer['last_activity_date']
                })
            
            # Send to CRM
            response = self.crm.bulk_update('Contact', updates)
            
            # Log results
            self.log_sync_results(response)
```

### Reverse ETL Configuration
```yaml
# Hightouch/Census Configuration
models:
  - name: customer_segments
    source:
      type: snowflake
      query: |
        SELECT 
          crm_id,
          customer_segment,
          lifetime_value,
          propensity_scores
        FROM PRESENTATION.CUSTOMER_ANALYTICS
        WHERE sync_to_crm = TRUE
    
    destination:
      type: salesforce
      object: Contact
      mode: upsert
      unique_identifier: crm_id
      
    mappings:
      - source: customer_segment
        destination: Customer_Segment__c
      - source: lifetime_value
        destination: LTV__c
      - source: propensity_scores.upsell
        destination: Upsell_Score__c
    
    schedule:
      interval: hourly
      timezone: UTC
```

## Field Mapping

### CRM to Data Platform
| CRM Field | Data Platform Field | Transformation | Notes |
|-----------|-------------------|----------------|-------|
| Id | crm_id | Direct | Primary key |
| FirstName | first_name | INITCAP() | Standardize |
| LastName | last_name | INITCAP() | Standardize |
| Email | email | LOWER() | Standardize |
| Phone | phone_primary | Clean regex | Remove formatting |
| AccountId | account_id | Direct | Foreign key |
| Custom_Field__c | custom_field | Parse JSON | Custom data |

### Data Platform to CRM
| Data Platform Field | CRM Field | Transformation | Notes |
|-------------------|-----------|----------------|-------|
| customer_segment | Customer_Segment__c | Direct | Picklist |
| lifetime_value | LTV__c | Round to 2 decimals | Currency |
| churn_risk_score | Churn_Risk__c | Scale 0-100 | Percentage |
| last_activity_date | Last_Activity__c | Format date | Date field |

## Conflict Resolution

### Merge Rules
```sql
-- Conflict Resolution Logic
CREATE OR REPLACE VIEW INTEGRATION.CUSTOMER_MASTER AS
WITH conflict_detection AS (
  SELECT 
    dp.customer_id,
    dp.email as dp_email,
    crm.email as crm_email,
    CASE 
      WHEN dp.modified_date > crm.modified_date THEN 'DATA_PLATFORM'
      WHEN crm.modified_date > dp.modified_date THEN 'CRM'
      ELSE 'MANUAL_REVIEW'
    END as master_source
  FROM DATA_PLATFORM.CUSTOMERS dp
  FULL OUTER JOIN CRM.CUSTOMERS crm
    ON dp.customer_id = crm.customer_id
  WHERE dp.email != crm.email
)
SELECT 
  customer_id,
  CASE master_source
    WHEN 'DATA_PLATFORM' THEN dp_email
    WHEN 'CRM' THEN crm_email
    ELSE dp_email -- Default to data platform
  END as email,
  master_source,
  CURRENT_TIMESTAMP() as resolution_timestamp
FROM conflict_detection;
```

## Monitoring and Alerting

### Integration Health Checks
```sql
-- Monitor sync delays
CREATE OR REPLACE VIEW MONITORING.CRM_SYNC_HEALTH AS
SELECT 
  'CRM_TO_DP' as sync_direction,
  MAX(modified_date) as last_crm_update,
  MAX(extracted_timestamp) as last_sync_time,
  DATEDIFF(minute, last_crm_update, last_sync_time) as lag_minutes,
  CASE 
    WHEN lag_minutes > 60 THEN 'CRITICAL'
    WHEN lag_minutes > 30 THEN 'WARNING'
    ELSE 'HEALTHY'
  END as health_status
FROM RAW.CRM.CUSTOMERS

UNION ALL

SELECT 
  'DP_TO_CRM' as sync_direction,
  MAX(segment_updated_date) as last_dp_update,
  MAX(crm_sync_timestamp) as last_sync_time,
  DATEDIFF(minute, last_dp_update, last_sync_time) as lag_minutes,
  CASE 
    WHEN lag_minutes > 120 THEN 'CRITICAL'
    WHEN lag_minutes > 60 THEN 'WARNING'
    ELSE 'HEALTHY'
  END as health_status
FROM SYNC_CONTROL.OUTBOUND_LOG;
```

### Alert Configuration
```yaml
alerts:
  - name: crm_sync_delay
    condition: lag_minutes > 60
    severity: high
    notification:
      - email: data-team@company.com
      - slack: #data-alerts
    
  - name: sync_failure_rate
    condition: failure_rate > 0.05
    severity: critical
    notification:
      - pagerduty: data-oncall
```

## Security Considerations

### API Security
- OAuth 2.0 authentication
- API rate limiting: 1000 requests/hour
- IP whitelisting for production
- Encrypted data in transit (TLS 1.2+)

### Data Security
- PII field encryption
- Audit logging for all changes
- Role-based access control
- Data retention policies