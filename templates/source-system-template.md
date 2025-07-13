# Source System Profile: [System Name]

## System Overview

### Basic Information
| Property | Value |
|----------|-------|
| System Name | [Name] |
| System Type | [CRM/ERP/Database/API/File/Other] |
| Owner/Team | [Business owner and technical team] |
| Environment | [Production/Staging/Development] |
| Criticality | [Critical/High/Medium/Low] |
| Last Updated | [Date] |

### Business Context
- **Purpose**: [Primary business function]
- **Business Process**: [Key processes supported]
- **User Base**: [Number and types of users]
- **Business Hours**: [Operating schedule]
- **SLA Requirements**: [Availability, performance requirements]

## Technical Specifications

### Infrastructure
| Component | Details |
|-----------|---------|
| Platform/Technology | [e.g., Oracle Database, Salesforce, SAP] |
| Version | [Current version] |
| Hosting | [On-premise/Cloud/Hybrid] |
| Environment | [Production details] |
| Hardware Specs | [Server specifications if applicable] |

### Connectivity
| Access Method | Details | Authentication |
|---------------|---------|----------------|
| Database Connection | [Connection string format] | [Auth method] |
| API Endpoint | [Base URL and version] | [API key/OAuth/etc.] |
| File Export | [FTP/SFTP/File share location] | [Credentials] |
| Real-time Streaming | [Kafka/Event hub details] | [Auth method] |

### Data Architecture
```sql
-- Example schema structure
DATABASE: [database_name]
  SCHEMA: [schema_name]
    TABLES:
      - [table1]: [description]
      - [table2]: [description]
      - [table3]: [description]
```

## Data Catalog

### Key Entities
| Entity | Table/Object | Primary Key | Description |
|--------|--------------|-------------|-------------|
| [Entity 1] | [table_name] | [key_field] | [Description] |
| [Entity 2] | [table_name] | [key_field] | [Description] |
| [Entity 3] | [table_name] | [key_field] | [Description] |

### Critical Data Elements
| Field Name | Data Type | Length | Nullable | Description | Business Rules |
|------------|-----------|--------|----------|-------------|----------------|
| [field1] | [type] | [length] | [Y/N] | [Description] | [Rules] |
| [field2] | [type] | [length] | [Y/N] | [Description] | [Rules] |
| [field3] | [type] | [length] | [Y/N] | [Description] | [Rules] |

### Data Relationships
```
[Entity1] --< [Entity2] --< [Entity3]
    |             |
    v             v
[EntityA]     [EntityB]
```

## Data Characteristics

### Volume Metrics
| Metric | Current | Growth Rate | Projection (1 year) |
|--------|---------|-------------|-------------------|
| Total Records | [#] | [%/month] | [#] |
| Daily Inserts | [#] | [%/month] | [#] |
| Daily Updates | [#] | [%/month] | [#] |
| Daily Deletes | [#] | [%/month] | [#] |
| Average Record Size | [KB] | [%/month] | [KB] |

### Update Patterns
- **Frequency**: [Real-time/Hourly/Daily/Weekly/Monthly]
- **Peak Times**: [When most updates occur]
- **Batch Windows**: [Available windows for bulk operations]
- **Change Detection**: [How to identify changed records]

### Data Quality Assessment
| Dimension | Score (1-10) | Issues | Impact |
|-----------|--------------|--------|--------|
| Completeness | [#] | [Description] | [Impact] |
| Accuracy | [#] | [Description] | [Impact] |
| Consistency | [#] | [Description] | [Impact] |
| Timeliness | [#] | [Description] | [Impact] |
| Validity | [#] | [Description] | [Impact] |
| Uniqueness | [#] | [Description] | [Impact] |

## Integration Considerations

### Extraction Options
| Method | Feasibility | Pros | Cons | Recommended |
|--------|-------------|------|------|-------------|
| Full Extract | [H/M/L] | [Pros] | [Cons] | [Y/N] |
| Incremental Extract | [H/M/L] | [Pros] | [Cons] | [Y/N] |
| Real-time CDC | [H/M/L] | [Pros] | [Cons] | [Y/N] |
| API Polling | [H/M/L] | [Pros] | [Cons] | [Y/N] |

### Technical Constraints
- **Rate Limits**: [API/database connection limits]
- **Security Restrictions**: [Firewall/VPN requirements]
- **Maintenance Windows**: [When system is unavailable]
- **Performance Impact**: [Acceptable load on source system]

### Dependencies
- **Upstream Systems**: [Systems that feed this one]
- **Downstream Systems**: [Systems that depend on this one]
- **Shared Resources**: [Databases, networks, etc.]

## Security and Compliance

### Data Classification
| Data Element | Classification | Compliance Requirements |
|--------------|----------------|------------------------|
| [Element 1] | [Public/Internal/Confidential/Restricted] | [GDPR/HIPAA/PCI/etc.] |
| [Element 2] | [Classification] | [Requirements] |

### Access Controls
- **Authentication**: [Method used]
- **Authorization**: [Role-based/attribute-based]
- **Audit Logging**: [What is logged]
- **Data Encryption**: [At rest/in transit]

## Contacts and Documentation

### Key Contacts
| Role | Name | Email | Phone | Responsibilities |
|------|------|-------|-------|------------------|
| Business Owner | [Name] | [Email] | [Phone] | [Responsibilities] |
| Technical Owner | [Name] | [Email] | [Phone] | [Responsibilities] |
| DBA/Admin | [Name] | [Email] | [Phone] | [Responsibilities] |

### Documentation Links
- **System Documentation**: [URL/Location]
- **API Documentation**: [URL/Location]
- **Database Schema**: [URL/Location]
- **Data Dictionary**: [URL/Location]
- **User Manuals**: [URL/Location]

## Testing and Validation

### Test Scenarios
- [ ] Connection test
- [ ] Sample data extraction
- [ ] Performance test (full load)
- [ ] Incremental extraction test
- [ ] Error handling test
- [ ] Security/authentication test

### Validation Queries
```sql
-- Record count validation
SELECT COUNT(*) FROM [main_table];

-- Data quality checks
SELECT COUNT(*) FROM [main_table] WHERE [key_field] IS NULL;

-- Latest timestamp
SELECT MAX([timestamp_field]) FROM [main_table];
```

## Action Items
- [ ] [Action item 1]
- [ ] [Action item 2]
- [ ] [Action item 3]

---
**Document Control**
- Version: 1.0
- Created: [Date]
- Updated: [Date]
- Owner: [Name]
- Next Review: [Date]