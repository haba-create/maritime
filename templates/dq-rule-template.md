# Data Quality Rule Template

## Rule Definition

### Basic Information
| Property | Value |
|----------|-------|
| Rule ID | DQ-[###] |
| Rule Name | [Descriptive name] |
| Category | [Completeness/Validity/Consistency/Uniqueness/Timeliness/Accuracy] |
| Priority | [Critical/High/Medium/Low] |
| Status | [Active/Inactive/Development/Deprecated] |
| Created Date | [Date] |
| Last Modified | [Date] |
| Owner | [Name/Team] |

### Business Context
- **Purpose**: [Why this rule is needed]
- **Business Impact**: [What happens if this rule fails]
- **Data Domain**: [Customer/Product/Financial/etc.]
- **Regulatory Requirement**: [GDPR/SOX/etc. if applicable]

## Rule Specification

### Rule Description
[Detailed description of what the rule validates]

### Affected Data
| Property | Value |
|----------|-------|
| Source System | [System name] |
| Database/Schema | [Database.Schema] |
| Table/Entity | [Table name] |
| Column(s) | [Column name(s)] |
| Data Type | [String/Number/Date/Boolean] |

### Rule Logic
```sql
-- SQL representation of the rule
SELECT 
    [validation_logic]
FROM [table_name]
WHERE [conditions];
```

### Rule Parameters
| Parameter | Value | Description |
|-----------|-------|-------------|
| [param1] | [value] | [Description] |
| [param2] | [value] | [Description] |

## Implementation

### Rule Implementation
```sql
-- Validation Query (returns violations)
SELECT 
    [primary_key],
    [relevant_columns],
    '[Rule ID]' as rule_id,
    '[Rule Name]' as rule_name,
    'VIOLATION' as status,
    CURRENT_TIMESTAMP() as check_timestamp
FROM [table_name]
WHERE [violation_condition];

-- Count Query (returns violation count)
SELECT 
    COUNT(*) as violation_count,
    '[Rule ID]' as rule_id
FROM [table_name]
WHERE [violation_condition];
```

### Alternative Implementations
```python
# Python/Pandas implementation
def validate_rule(df):
    """
    Validate data quality rule
    """
    violations = df[df[condition]]
    return {
        'rule_id': 'DQ-[###]',
        'violation_count': len(violations),
        'violations': violations
    }
```

## Thresholds and Actions

### Quality Thresholds
| Threshold Type | Value | Action |
|----------------|-------|--------|
| Pass Threshold | [%] | Continue processing |
| Warning Threshold | [%] | Log warning, notify team |
| Fail Threshold | [%] | Stop processing, alert |
| Critical Threshold | [%] | Immediate escalation |

### Actions by Severity
| Violation Count/% | Severity | Automatic Actions | Manual Actions |
|------------------|----------|-------------------|----------------|
| 0-[%] | None | Continue | None |
| [%]-[%] | Low | Log | Weekly review |
| [%]-[%] | Medium | Notify team | Daily review |
| [%]-[%] | High | Stop pipeline | Immediate fix |
| >[%] | Critical | Alert on-call | Emergency response |

## Monitoring and Reporting

### Execution Schedule
- **Frequency**: [Real-time/Hourly/Daily/Weekly]
- **Time**: [Specific time if batch]
- **Dependencies**: [Other rules or processes]

### Metrics to Track
- Violation count/percentage
- Trend over time
- Performance impact
- Resolution time

### Alert Configuration
```yaml
# Alert configuration
alert:
  rule_id: DQ-[###]
  thresholds:
    warning: [%]
    critical: [%]
  notifications:
    - type: email
      recipients: [email_list]
    - type: slack
      channel: #data-quality
```

### Dashboard Visualization
- [ ] Violation trend chart
- [ ] Current status indicator
- [ ] Top violations table
- [ ] Impact assessment

## Remediation

### Root Cause Analysis
| Potential Cause | Likelihood | Investigation Steps |
|-----------------|------------|-------------------|
| [Cause 1] | [H/M/L] | [Steps to investigate] |
| [Cause 2] | [H/M/L] | [Steps to investigate] |

### Remediation Options
1. **Fix at Source**
   - Description: [How to fix in source system]
   - Effort: [Hours/Days]
   - Owner: [Team]

2. **Fix in Pipeline**
   - Description: [How to fix during ETL]
   - Effort: [Hours/Days]
   - Owner: [Team]

3. **Business Rule Change**
   - Description: [Change business logic]
   - Effort: [Hours/Days]
   - Owner: [Team]

### Data Cleansing Rules
```sql
-- Example cleansing logic
UPDATE [table_name]
SET [column] = [corrected_value]
WHERE [condition];
```

## Testing

### Test Cases
| Test Case | Input | Expected Output | Actual Output | Status |
|-----------|-------|-----------------|---------------|--------|
| Valid data | [Sample] | 0 violations | [Result] | [Pass/Fail] |
| Invalid data | [Sample] | X violations | [Result] | [Pass/Fail] |
| Edge case | [Sample] | [Expected] | [Result] | [Pass/Fail] |

### Test Data
```sql
-- Create test data for rule validation
INSERT INTO test_table VALUES
    ([valid_record]),
    ([invalid_record]),
    ([edge_case]);
```

## Documentation

### Change History
| Version | Date | Change | Author |
|---------|------|--------|--------|
| 1.0 | [Date] | Initial creation | [Name] |
| 1.1 | [Date] | [Description] | [Name] |

### Related Rules
- **Dependent Rules**: [Rules that depend on this one]
- **Conflicting Rules**: [Rules that might conflict]
- **Related Rules**: [Similar or complementary rules]

### Business Glossary
| Term | Definition |
|------|------------|
| [Term1] | [Definition] |
| [Term2] | [Definition] |

## Approval

### Review and Approval
| Role | Name | Date | Status |
|------|------|------|--------|
| Business Owner | [Name] | [Date] | [Approved/Pending] |
| Data Steward | [Name] | [Date] | [Approved/Pending] |
| Technical Owner | [Name] | [Date] | [Approved/Pending] |
| DQ Manager | [Name] | [Date] | [Approved/Pending] |

### Sign-off
- [ ] Business requirements validated
- [ ] Technical implementation reviewed
- [ ] Test cases passed
- [ ] Monitoring configured
- [ ] Documentation complete

---
**Document Control**
- Template Version: 1.0
- Created: [Date]
- Last Updated: [Date]
- Next Review: [Date]