# Customer Data Quality Rules

## Overview
This document defines data quality rules and validation criteria for customer data.

## Data Quality Dimensions

### Completeness Rules
| Rule ID | Field | Rule Description | Severity | Action |
|---------|-------|------------------|----------|--------|
| DQ-001 | customer_id | Must not be NULL | Critical | Reject |
| DQ-002 | email | Must not be NULL for active customers | High | Flag |
| DQ-003 | phone | At least one phone number required | Medium | Flag |
| DQ-004 | address | Complete address for billing customers | High | Flag |

### Validity Rules
| Rule ID | Field | Rule Description | Severity | Action |
|---------|-------|------------------|----------|--------|
| DQ-101 | email | Must match email pattern | High | Reject |
| DQ-102 | phone | Must be valid phone format | Medium | Clean |
| DQ-103 | postal_code | Must match country format | Medium | Clean |
| DQ-104 | country_code | Must be valid ISO code | High | Reject |

### Consistency Rules
| Rule ID | Fields | Rule Description | Severity | Action |
|---------|--------|------------------|----------|--------|
| DQ-201 | first_name, last_name | Cannot both be NULL | High | Flag |
| DQ-202 | country, postal_code | Postal code must match country | Medium | Flag |
| DQ-203 | created_date, updated_date | Updated >= Created | Low | Flag |

### Uniqueness Rules
| Rule ID | Field(s) | Rule Description | Severity | Action |
|---------|----------|------------------|----------|--------|
| DQ-301 | customer_id | Must be unique across system | Critical | Reject |
| DQ-302 | email | Should be unique for active customers | High | Flag |
| DQ-303 | tax_id | Must be unique within country | High | Flag |

## Rule Implementation

### SQL Examples
```sql
-- DQ-001: Customer ID Not Null
SELECT COUNT(*) as violation_count
FROM customers
WHERE customer_id IS NULL;

-- DQ-101: Valid Email Format
SELECT customer_id, email
FROM customers
WHERE email NOT REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
  AND email IS NOT NULL;

-- DQ-201: Name Consistency
SELECT customer_id
FROM customers
WHERE first_name IS NULL 
  AND last_name IS NULL;
```

## Data Quality Metrics

### Target Thresholds
| Dimension | Current | Target | Timeline |
|-----------|---------|--------|----------|
| Completeness | [%] | 95% | [Date] |
| Validity | [%] | 98% | [Date] |
| Consistency | [%] | 99% | [Date] |
| Uniqueness | [%] | 100% | [Date] |

## Monitoring and Reporting

### Dashboard Metrics
- Daily violation counts by rule
- Trend analysis by dimension
- Source system quality scores
- Business impact assessment

### Alerting Rules
- Critical violations: Immediate notification
- High violations > threshold: Daily summary
- Medium/Low violations: Weekly report

## Remediation Process
1. **Detection**: Automated rule execution
2. **Classification**: Severity assessment
3. **Notification**: Alert appropriate team
4. **Resolution**: Fix at source or in pipeline
5. **Verification**: Re-run rules post-fix