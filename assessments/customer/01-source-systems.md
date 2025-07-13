# Customer Data Source Systems Assessment

## Overview
This document catalogs all source systems containing customer data and their characteristics.

## Source Systems Inventory

### System 1: [System Name]
- **Type**: [CRM/ERP/Database/API]
- **Owner**: [Business Unit/Team]
- **Data Volume**: [Records/Size]
- **Update Frequency**: [Real-time/Daily/Weekly]
- **Access Method**: [API/Database/File Export]
- **Authentication**: [Method]

### System 2: [System Name]
- **Type**: [Type]
- **Owner**: [Owner]
- **Data Volume**: [Volume]
- **Update Frequency**: [Frequency]
- **Access Method**: [Method]
- **Authentication**: [Auth]

## Data Elements by System

### Critical Customer Attributes
| Attribute | System 1 | System 2 | System 3 | Notes |
|-----------|----------|----------|----------|-------|
| Customer ID | ✓ | ✓ | ✓ | Primary key |
| Name | ✓ | ✓ | ✓ | |
| Email | ✓ | ✓ | | |
| Phone | ✓ | | ✓ | |
| Address | ✓ | | ✓ | |

## Technical Specifications

### API Endpoints
```
System 1: https://api.example.com/v2/customers
System 2: [Endpoint]
```

### Database Connections
```sql
-- System 1 Connection
Server: [server]
Database: [database]
Schema: [schema]
```

## Data Quality Observations
- [Key finding 1]
- [Key finding 2]
- [Key finding 3]

## Next Steps
- [ ] Validate data volumes
- [ ] Test connectivity
- [ ] Document data lineage
- [ ] Identify golden records