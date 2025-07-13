# ETL Pattern Template: [Pattern Name]

## Pattern Overview

### Basic Information
| Property | Value |
|----------|-------|
| Pattern Name | [Descriptive name] |
| Pattern Type | [Batch/Streaming/Micro-batch/Event-driven] |
| Complexity Level | [Simple/Medium/Complex] |
| Use Cases | [Primary scenarios where this pattern applies] |
| Technology Stack | [Tools/platforms used] |
| Maturity Level | [Experimental/Stable/Deprecated] |

### Pattern Description
[Detailed description of what this pattern does and when to use it]

### Benefits
- [Benefit 1]
- [Benefit 2]
- [Benefit 3]

### Limitations
- [Limitation 1]
- [Limitation 2]
- [Limitation 3]

## Architecture Pattern

### High-Level Design
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Source    │────▶│   Extract   │────▶│ Transform   │────▶│    Load     │
│   Systems   │     │   Layer     │     │   Layer     │     │  Target     │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### Component Breakdown
| Component | Purpose | Technology | Scalability |
|-----------|---------|------------|-------------|
| [Component 1] | [Purpose] | [Technology] | [Horizontal/Vertical] |
| [Component 2] | [Purpose] | [Technology] | [Horizontal/Vertical] |
| [Component 3] | [Purpose] | [Technology] | [Horizontal/Vertical] |

## Implementation Details

### Extract Phase
```python
# Extract pattern implementation
def extract_data(source_config):
    """
    Extract data from source system
    
    Args:
        source_config: Configuration for source system
        
    Returns:
        DataFrame: Extracted data
    """
    # Implementation logic
    pass
```

```sql
-- SQL-based extraction
CREATE OR REPLACE PROCEDURE extract_source_data()
RETURNS TABLE
LANGUAGE SQL
AS
$$
  SELECT 
    [columns],
    CURRENT_TIMESTAMP() as extracted_at
  FROM [source_table]
  WHERE [incremental_condition];
$$;
```

### Transform Phase
```python
# Transform pattern implementation
def transform_data(raw_data, business_rules):
    """
    Apply transformations to raw data
    
    Args:
        raw_data: Raw extracted data
        business_rules: Business transformation rules
        
    Returns:
        DataFrame: Transformed data
    """
    # Cleansing
    cleaned_data = apply_data_quality_rules(raw_data)
    
    # Business transformations
    transformed_data = apply_business_rules(cleaned_data, business_rules)
    
    # Enrichment
    enriched_data = apply_enrichment(transformed_data)
    
    return enriched_data
```

### Load Phase
```python
# Load pattern implementation
def load_data(transformed_data, target_config):
    """
    Load transformed data to target system
    
    Args:
        transformed_data: Transformed data
        target_config: Target system configuration
    """
    # Validate data before loading
    validation_results = validate_data(transformed_data)
    
    if validation_results.is_valid:
        # Load data
        load_to_target(transformed_data, target_config)
        
        # Update metadata
        update_lineage_metadata(transformed_data, target_config)
    else:
        # Handle validation failures
        handle_validation_errors(validation_results)
```

## Configuration

### Configuration Template
```yaml
# ETL Pattern Configuration
pattern_config:
  name: "[pattern_name]"
  version: "1.0"
  
  source:
    type: "[database/api/file/stream]"
    connection:
      host: "[hostname]"
      port: [port]
      database: "[database]"
      schema: "[schema]"
    
    extraction:
      method: "[full/incremental/cdc]"
      batch_size: [size]
      incremental_field: "[field_name]"
      schedule: "[cron_expression]"
  
  transform:
    rules:
      - name: "[rule_name]"
        type: "[validation/cleansing/enrichment]"
        config: {}
    
    quality_checks:
      - type: "[completeness/validity/consistency]"
        threshold: [percentage]
        action: "[fail/warn/continue]"
  
  target:
    type: "[database/file/api]"
    connection: {}
    load_strategy: "[insert/upsert/merge]"
    parallelism: [number_of_threads]

  monitoring:
    metrics:
      - processing_time
      - record_count
      - error_rate
    
    alerts:
      - condition: "error_rate > 0.05"
        severity: "high"
        recipients: ["team@company.com"]
```

## Data Flow Patterns

### Pattern 1: Full Refresh
```mermaid
graph LR
    A[Source] --> B[Extract All]
    B --> C[Transform]
    C --> D[Truncate Target]
    D --> E[Load All]
```

**Use Case**: Small datasets, no history requirements
**Pros**: Simple, consistent state
**Cons**: High resource usage, data loss risk

### Pattern 2: Incremental Load
```mermaid
graph LR
    A[Source] --> B[Extract Delta]
    B --> C[Transform]
    C --> D[Merge/Upsert]
    D --> E[Update Metadata]
```

**Use Case**: Large datasets, regular updates
**Pros**: Efficient, scalable
**Cons**: Complex logic, potential for missed data

### Pattern 3: Change Data Capture
```mermaid
graph LR
    A[Source] --> B[CDC Stream]
    B --> C[Real-time Transform]
    C --> D[Stream Load]
    D --> E[Update Target]
```

**Use Case**: Real-time requirements, high-frequency changes
**Pros**: Near real-time, efficient
**Cons**: Complex setup, ordering challenges

## Error Handling Patterns

### Error Categories
| Error Type | Handling Strategy | Recovery Method | Notification |
|------------|------------------|-----------------|--------------|
| Connection Error | Retry with backoff | Automatic retry | Log warning |
| Data Quality Error | Quarantine record | Manual review | Email team |
| Transformation Error | Skip record | Log and continue | Daily report |
| Target Error | Fail pipeline | Manual intervention | Alert on-call |

### Error Handling Implementation
```python
def handle_errors(error_type, error_data, config):
    """
    Centralized error handling
    """
    error_handlers = {
        'connection_error': handle_connection_error,
        'data_quality_error': handle_dq_error,
        'transformation_error': handle_transform_error,
        'target_error': handle_target_error
    }
    
    handler = error_handlers.get(error_type)
    if handler:
        return handler(error_data, config)
    else:
        raise UnknownErrorType(error_type)
```

## Performance Optimization

### Optimization Strategies
1. **Parallel Processing**
   ```python
   # Process data in parallel chunks
   from concurrent.futures import ThreadPoolExecutor
   
   def process_in_parallel(data_chunks, transform_func):
       with ThreadPoolExecutor(max_workers=4) as executor:
           results = list(executor.map(transform_func, data_chunks))
       return results
   ```

2. **Batch Optimization**
   ```sql
   -- Optimized batch processing
   CREATE OR REPLACE PROCEDURE process_batch(batch_size INT)
   AS
   $$
   DECLARE
     offset_val INT := 0;
   BEGIN
     WHILE TRUE LOOP
       -- Process batch
       INSERT INTO target_table
       SELECT * FROM staging_table
       LIMIT batch_size OFFSET offset_val;
       
       -- Exit if no more records
       IF ROW_COUNT = 0 THEN EXIT; END IF;
       
       offset_val := offset_val + batch_size;
     END LOOP;
   END;
   $$;
   ```

3. **Memory Management**
   ```python
   # Use generators for large datasets
   def process_large_dataset(file_path):
       def data_generator():
           with open(file_path, 'r') as f:
               for line in f:
                   yield process_line(line)
       
       return data_generator()
   ```

## Monitoring and Metrics

### Key Metrics
| Metric | Description | Target | Critical Threshold |
|--------|-------------|--------|--------------------|
| Processing Time | End-to-end pipeline duration | [X minutes] | [Y minutes] |
| Throughput | Records processed per hour | [X records/hour] | [Y records/hour] |
| Error Rate | Percentage of failed records | <1% | >5% |
| Data Freshness | Time between source update and target availability | [X minutes] | [Y minutes] |

### Monitoring Implementation
```python
# Monitoring decorator
def monitor_pipeline(func):
    def wrapper(*args, **kwargs):
        start_time = time.time()
        
        try:
            result = func(*args, **kwargs)
            
            # Log success metrics
            duration = time.time() - start_time
            record_count = len(result) if hasattr(result, '__len__') else 0
            
            log_metrics({
                'pipeline': func.__name__,
                'status': 'success',
                'duration': duration,
                'record_count': record_count
            })
            
            return result
            
        except Exception as e:
            # Log error metrics
            log_metrics({
                'pipeline': func.__name__,
                'status': 'error',
                'error_type': type(e).__name__,
                'error_message': str(e)
            })
            raise
    
    return wrapper
```

## Testing Strategy

### Test Types
1. **Unit Tests**: Individual components
2. **Integration Tests**: End-to-end pipeline
3. **Performance Tests**: Scalability and throughput
4. **Data Quality Tests**: Validation rules

### Test Implementation
```python
import pytest

class TestETLPattern:
    def test_extract_phase(self):
        # Test data extraction
        source_config = get_test_config()
        result = extract_data(source_config)
        
        assert result is not None
        assert len(result) > 0
        assert 'extracted_at' in result.columns
    
    def test_transform_phase(self):
        # Test data transformation
        raw_data = get_test_data()
        business_rules = get_test_rules()
        
        result = transform_data(raw_data, business_rules)
        
        assert result is not None
        assert validate_schema(result)
        assert check_data_quality(result)
    
    def test_load_phase(self):
        # Test data loading
        transformed_data = get_transformed_test_data()
        target_config = get_test_target_config()
        
        load_data(transformed_data, target_config)
        
        # Verify data was loaded correctly
        loaded_data = query_target_data()
        assert len(loaded_data) == len(transformed_data)
```

## Deployment and Operations

### Deployment Checklist
- [ ] Code reviewed and tested
- [ ] Configuration validated
- [ ] Infrastructure provisioned
- [ ] Monitoring configured
- [ ] Error handling tested
- [ ] Documentation updated
- [ ] Team trained
- [ ] Rollback plan ready

### Operational Procedures
```bash
# Start pipeline
./scripts/start_pipeline.sh --config production.yaml

# Monitor pipeline
./scripts/monitor_pipeline.sh --pipeline customer_etl

# Stop pipeline
./scripts/stop_pipeline.sh --pipeline customer_etl --graceful

# Restart pipeline
./scripts/restart_pipeline.sh --pipeline customer_etl
```

## Best Practices

### Design Principles
1. **Idempotency**: Pipeline can be re-run safely
2. **Modularity**: Components are loosely coupled
3. **Observability**: Comprehensive logging and monitoring
4. **Fault Tolerance**: Graceful error handling
5. **Scalability**: Can handle growing data volumes

### Implementation Guidelines
- Use configuration files for environment-specific settings
- Implement comprehensive logging at each stage
- Build in data quality checkpoints
- Design for parallel processing where possible
- Include metadata tracking for lineage

### Common Pitfalls
- Not handling schema evolution
- Insufficient error handling
- Poor performance for large datasets
- Lack of monitoring and alerting
- Inadequate testing coverage

---
**Document Control**
- Version: 1.0
- Created: [Date]
- Last Updated: [Date]
- Author: [Name]
- Next Review: [Date]