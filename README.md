# Marlink Data Architecture Project

## Overview

This repository contains documentation and proof-of-concepts for Marlink's data architecture initiatives, focusing on:
- Migration from Azure Databricks to Snowflake
- Customer data harmonization and cleansing
- CRM integration architecture

## Current Focus: Customer Domain Assessment

We are currently working on assessing the effort required to implement a comprehensive customer data harmonization solution that will:
- Consolidate multiple customer data sources
- Establish data quality standards
- Create a unified customer schema
- Enable seamless CRM integration

## Project Structure

```
maritime/
├── index.html                  # Main documentation hub (open in browser)
├── assessments/               # Assessment documents
│   └── customer/             # Customer domain specific assessments
├── templates/                # Document templates
├── poc/                      # Proof of concept implementations
├── automation/               # Documentation generation scripts
├── guides/                   # Technical guides
└── deliverables/            # Final deliverables
```

## Getting Started

1. Open `index.html` in your browser to access the main documentation hub
2. Follow the phased approach outlined in the Customer Domain Assessment section
3. Use the provided templates for consistent documentation

## Key Stakeholders

- **Stephen** - Data Architect (Lead)
- **Silamir** - CRM Integration Lead
- **Dipesh** - Project Sponsor
- **Reem** - Business Stakeholder
- **Group IT** - Implementation Team

## Technology Stack

- **Current**: Azure Databricks
- **Target**: Snowflake
- **Integration**: CRM (TBD)

## Documentation Approach

This project uses an automated documentation pipeline to:
- Generate assessment documents from structured data
- Maintain consistency across deliverables
- Track progress and dependencies
- Provide effort estimates based on actual analysis

## Contributing

Please follow these guidelines:
1. Use the provided templates for all documentation
2. Update the deliverables tracker in `index.html` as work progresses
3. Document all POC results in the designated folders
4. Keep technical decisions logged in the ADR format

## License

Internal Marlink project - Confidential