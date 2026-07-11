![Power BI](https://img.shields.io/badge/Power%20BI-Reporting-F2C811?logo=powerbi&logoColor=black)
![SQL Server](https://img.shields.io/badge/SQL%20Server-2022-CC2927?logo=microsoftsqlserver&logoColor=white)
![DAX](https://img.shields.io/badge/DAX-Measures-blue)
![Metadata Governance](https://img.shields.io/badge/Metadata-Governance-2E7D32)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

# Enterprise Metadata Governance Platform

## Project Overview

This project implements an enterprise metadata governance solution using Microsoft SQL Server and Power BI, built on top of Microsoft's AdventureWorks2022 sample database.

Rather than working with a small synthetic dataset, this project uses Microsoft's AdventureWorks2022 sample database as the operational source system and builds a metadata governance layer on top of it. The project demonstrates core metadata governance concepts commonly found in enterprise platforms such as Microsoft Purview and Collibra, including business glossary management, data dictionaries, stewardship, classification, lineage, and data quality reporting.

The result is a 5-page Power BI dashboard that reports on the health, ownership, and structure of the governed metadata — the kind of reporting a Metadata Management Analyst would maintain and present to stakeholders.

📊 **[Download the full Power BI report (.pbix)](power-bi/Metadata%20Dashboard.pbix)**

## Dashboard Preview

### Executive Overview
![Overview](screenshots/overview.png)

### Metadata Catalog
![Catalog](screenshots/catalog.png)

### Business Glossary
![Glossary](screenshots/glossary.png)

### Data Dictionary
![Dictionary](screenshots/dictionary.png)

### Data Lineage & Quality
![Lineage](screenshots/lineage.png)

## Semantic Data Model

The Power BI semantic model connects the six reporting views to the `DataAsset` table, which serves as the central table for the entire report:

![Data Model](docs/data_model.png)

## Project Metrics

* 18 governed data assets
* 172 metadata dictionary records
* 15 business glossary terms
* 10 lineage relationships
* 10 data quality rules
* 6 SQL reporting views
* 5 interactive Power BI report pages

## Business Problem

Organizations that operate across multiple business domains and systems often struggle with:

* Unclear ownership and stewardship of data assets
* Undocumented or inconsistent business terminology
* Sensitive data that isn't consistently classified
* No visibility into how data flows between systems
* No structured way to track data quality rules and their severity

This project addresses those problems by building a governed metadata repository on top of a real relational database, then exposing it through a governance reporting layer.

## Source Database

**AdventureWorks2022** (Microsoft's official OLTP sample database), restored in SQL Server 2022. 18 business-critical tables were selected as governed data assets across four business domains:

| Business Domain | Tables |
|---|---|
| Sales | Customer, SalesOrderHeader, SalesOrderDetail, SalesPerson, SalesTerritory, CreditCard, Store |
| Production | Product, ProductCategory, ProductSubcategory, ProductInventory |
| Person | Person, Address, EmailAddress, PersonPhone, Password |
| Human Resources | Employee, Department |

*AdventureWorks2022 is Microsoft's publicly distributed sample database — its schema was not designed as part of this project. This project's original contribution is the metadata governance layer built on top of it (below).*

## Metadata Repository — Data Model

A separate database, **MetadataManagementDB**, was designed and built from scratch to store the governance layer independently of the operational AdventureWorks data.

**Core tables:**

| Table | Rows | Purpose |
|---|---|---|
| DataAsset | 18 | Inventory of governed data assets — domain, owner, steward, classification, criticality |
| DataDictionary | 172 | Column-level technical metadata, auto-discovered from INFORMATION_SCHEMA.COLUMNS and enriched with classification |
| BusinessGlossary | 15 | Business term definitions, mapped to domains, owners, and stewards |
| DataLineage | 10 | Source-to-target relationships and the business processes they support |
| DataQualityRule | 10 | Data quality rules by severity, domain, and status |

**Reporting views** (built for the Power BI semantic layer):

* vw_MetadataExecutiveSummary
* vw_DataAssetSummary
* vw_DataDictionarySummary
* vw_BusinessGlossarySummary
* vw_DataLineageSummary
* vw_DataQualitySummary

## Tools and Technologies

* Microsoft SQL Server 2022
* T-SQL (DDL, DML, system catalog views, aggregation views)
* Microsoft Power BI Desktop
* DAX
* GitHub

## Dashboard Pages

### 1. Overview
Executive-level snapshot of the governed metadata: total assets, metadata attributes, business terms, lineage links, and quality rules, alongside asset breakdowns by business domain, classification, criticality, and rule severity.

### 2. Metadata Catalog
A searchable, filterable inventory of all 18 governed data assets — business domain, asset type, description, owner, steward, classification, and criticality — with classification and criticality shown via color-coded conditional formatting, and filterable by domain, classification, criticality, and steward.

### 3. Business Glossary
Business term coverage by domain, approval status, business owner, and data steward — showing how enterprise terminology maps to ownership and governance responsibility.

### 4. Data Dictionary
Technical column-level inventory across all 18 assets — data type, nullability, key-column flags, and classification — filterable by business domain, with the same classification color coding used throughout the report.

### 5. Data Lineage & Quality
Source-to-target relationships by business process, and data quality rules by domain, severity, and status — surfacing where governance risk is concentrated.

## Key DAX Measures

```DAX
Total Data Assets = DISTINCTCOUNT(DataAsset[AssetID])

Critical Assets =
CALCULATE(
    DISTINCTCOUNT(DataAsset[AssetID]),
    DataAsset[Criticality] = "Critical"
)

Confidential Assets =
CALCULATE(
    DISTINCTCOUNT(DataAsset[AssetID]),
    DataAsset[Classification] = "Confidential"
)

Business Domains Covered = DISTINCTCOUNT(BusinessGlossary[BusinessDomain])

Total Glossary Terms = SUM(vw_BusinessGlossarySummary[GlossaryTermCount])

Approved Terms =
CALCULATE(
    SUM(vw_BusinessGlossarySummary[GlossaryTermCount]),
    vw_BusinessGlossarySummary[Status] = "Approved"
)

Total Lineage Relationships = SUM(vw_DataLineageSummary[RelationshipCount])

Total Quality Rules = SUM(vw_DataQualitySummary[RuleCount])

Critical Rules =
CALCULATE(
    SUM(vw_DataQualitySummary[RuleCount]),
    vw_DataQualitySummary[Severity] = "Critical"
)
```

## Example SQL — Automated Metadata Discovery

Rather than hand-typing all 172 dictionary records, column-level metadata was discovered directly from SQL Server's system catalog and enriched with governance attributes inherited from the parent asset:

```sql
INSERT INTO DataDictionary
    (AssetID, BusinessDomain, AssetName, ColumnName, DataType,
     IsNullable, MaxLength, BusinessDefinition, Classification,
     IsKeyColumn, SourceSystem)
SELECT
    da.AssetID,
    c.TABLE_SCHEMA,
    c.TABLE_NAME,
    c.COLUMN_NAME,
    c.DATA_TYPE,
    c.IS_NULLABLE,
    c.CHARACTER_MAXIMUM_LENGTH,
    'Business definition to be reviewed by data steward',
    da.Classification,
    CASE WHEN c.COLUMN_NAME LIKE '%ID' THEN 'Yes' ELSE 'No' END,
    'AdventureWorks2022'
FROM AdventureWorks2022.INFORMATION_SCHEMA.COLUMNS c
JOIN DataAsset da
    ON da.BusinessDomain = c.TABLE_SCHEMA
    AND da.AssetName = c.TABLE_NAME;
```

## Project Architecture

```text
AdventureWorks2022 (SQL Server OLTP database)
          |
          v
   Business domain & asset selection
   (18 critical assets across 4 domains)
          |
          v
     MetadataManagementDB
     DataAsset | DataDictionary | BusinessGlossary
     DataLineage | DataQualityRule
          |
          v
      SQL reporting views
   (vw_MetadataExecutiveSummary, vw_DataAssetSummary,
    vw_DataDictionarySummary, vw_BusinessGlossarySummary,
    vw_DataLineageSummary, vw_DataQualitySummary)
          |
          v
      Power BI semantic model
          |
          v
  5-page Metadata Governance Dashboard
```

## Key Insights the Dashboard Surfaces

* Which business domains carry the most Critical or Restricted assets
* Classification distribution across the governed asset inventory (Confidential / Internal / Restricted)
* Business glossary coverage and approval status by domain and steward
* Column-level classification and nullability across all 172 dictionary entries
* Data quality rules concentrated by severity and domain
* Source-to-target lineage relationships grouped by business process

## Data Privacy

All governance metadata (glossary definitions, classifications, ownership, quality rules) was authored for this project. The underlying business data is Microsoft's publicly distributed AdventureWorks2022 sample database and contains no real customer or organizational information.

## Skills Demonstrated

* Metadata Management & Data Governance
* Business Glossary & Data Taxonomy Design
* Data Classification & Stewardship Assignment
* Data Lineage Documentation
* Data Quality Rule Definition
* SQL Server Metadata Repository Design (T-SQL, system catalog views, reporting views)
* Power BI Dashboard Development
* DAX Measure Design
* Data Modeling

## Potential Business Use Cases

* Metadata Management Analyst / Data Governance Analyst reporting
* Data Steward asset review and classification audits
* Business glossary maintenance and approval tracking
* Data quality monitoring by domain and severity

## Future Enhancements

* Foreign key relationships linking DataDictionary columns directly to BusinessGlossary terms, enabling a true "metadata completeness %" measure
* Metadata change request / approval workflow tables
* Automated re-scan of INFORMATION_SCHEMA to detect schema drift
* Microsoft Purview or Collibra API integration
* Row-level security by business domain

## Repository Structure

```text
Enterprise-Metadata-Governance-Platform/
│
├── README.md
├── LICENSE
│
├── sql/
│   ├── 01_Business_Domain_Inventory.sql
│   ├── 02_Data_Asset_Catalog.sql
│   ├── 03_Technical_Metadata_Inventory.sql
│   ├── 05_Data_Quality_Summary.sql
│   └── 06_Metadata_Governance_Dashboard_View.sql
│
├── power-bi/
│   └── Metadata Dashboard.pbix
│
├── screenshots/
│   ├── overview.png
│   ├── catalog.png
│   ├── glossary.png
│   ├── dictionary.png
│   └── lineage.png
│
└── docs/
    └── data_model.png
```

## Keywords

Metadata Governance · Metadata Management · Data Governance · Business Glossary · Data Stewardship · Data Dictionary · Data Lineage · Data Classification · SQL Server · Power BI · DAX · Enterprise Metadata · Microsoft Purview · Collibra · Data Catalog

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

## Author

**Mohammed Abdullah Abdul Basith Qureshi**
Data Governance | Data Analytics | SQL | Power BI
