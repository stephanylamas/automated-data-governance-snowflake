# Automated Enterprise Data Governance and Role Based Access Control Framework with Snowflake

An end-to-end Snowflake security, access governance, and compliance pipeline implementing Role-Based Access Control (RBAC), Dynamic Data Masking, Object Tagging, and Access Auditing across a Medallion architecture (`BRONZE_RAW` → `SILVER_SECURE`).

## Industry Problem & Architecture Overview

Modern data platforms must balance self-service analytics with strict compliance requirements (GDPR, CCPA, HIPAA). Granting broad access to raw landing zones exposes Personally Identifiable Information (PII), while manual permission management creates privilege creep and administrative bottlenecks.

This project demonstrates a Separation of Duties model in Snowflake:
1. **Infrastructure & Data Ingestion (`SYSADMIN`):** Provisions raw landing zones (`BRONZE_RAW`) without holding authority to alter security policies.
2. **Governance & Policy Enforcement (`GOVERNANCE_ADMIN`):** Owns the protected production layer (`SILVER_SECURE`), defines classification metadata, and attaches dynamic column-level security policies.
3. **Role-Based Consumption (`HR_ANALYST`, `MARKETING_ANALYST`):** Queries the exact same production table while receiving dynamically masked or unmasked data based on session role context.
4. **Compliance Auditing (`ACCOUNTADMIN`):** Monitors historical PII access via Snowflake's `ACCOUNT_USAGE.ACCESS_HISTORY`.

## Enterprise Tooling Alignment (Immuta & Alation)

While built using Snowflake-native primitives, this architecture is structured to serve as the underlying enforcement layer for enterprise data governance and cataloging suites:

* **Immuta (Attribute-Based & Dynamic Access Control):** Policies are decoupled from table DDL via explicit `ALTER TABLE ... SET MASKING POLICY` statements. This mirrors how policy engines like **Immuta** dynamically push down and manage centralized masking rules without disrupting upstream ETL/ELT pipelines.
* **Alation (Automated Data Cataloging & Discovery):** Sensitive fields are tagged using custom Snowflake object tags (`PII_CLASSIFICATION = 'NAME' | 'EMAIL' | 'CONFIDENTIAL'`). This metadata structure enables automated catalog harvesting tools like **Alation** to discover, classify, and surface PII lineage to data stewards.

## Repository Structure

```text
├── 01_infrastructure_and_seeding.sql   # Warehouse, DB/Schemas, RBAC hierarchy, and mock PII seeding
├── 02_governance_and_masking.sql       # Dynamic masking policies, PII tags, and secure table provisioning
├── 03_security_testing_and_audit.sql   # Role-isolated verification tests and ACCESS_HISTORY audit queries
└── README.md                           # Architecture documentation and execution guide
```

## Security & Governance Controls Implemented

### 1. Hierarchical RBAC & Separation of Duties
* **`GOVERNANCE_ADMIN`:** Isolated security role owning `SILVER_SECURE`, masking policies, and tags.
* **`DATA_ENGINEER`:** Inherits consumer roles for pipeline troubleshooting without owning security policies.
* **`HR_ANALYST`:** Authorized to view unmasked employee/customer emails and compensation data.
* **`MARKETING_ANALYST`:** Restricted to masked PII and redacted financial figures.

### 2. Dynamic Column-Level Security (CLS)

| Column | Classification Tag | `HR_ANALYST` Output | `MARKETING_ANALYST` Output |
| :--- | :--- | :--- | :--- |
| `FULL_NAME` | `NAME` | `Amanda Zepher` | `Amanda Zepher` |
| `EMAIL` | `EMAIL` | `azepher@mockenterprise.com` | `az*****@mockenterprise.com` |
| `SALARY` | `CONFIDENTIAL` | `75000` | `0` |

### 3. Access Governance & Audit Trail
* Leverages `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` to trace `USER_NAME`, `ROLE_NAME`, `QUERY_START_TIME`, and `BASE_OBJECTS_ACCESSED` for incident response and compliance audits.

## How to Run

1. Open a SQL Worksheet in **Snowflake (Enterprise Edition)**.
2. Run `01_infrastructure_and_seeding.sql` as `ACCOUNTADMIN` / `SYSADMIN` to initialize the warehouse, RBAC roles, and `BRONZE_RAW.CUSTOMER_LANDING`.
3. Run `02_governance_and_masking.sql` to create tags, bind dynamic masking policies, and populate `SILVER_SECURE.CUSTOMER_PROFILES`.
4. Run `03_security_testing_and_audit.sql` (using `SET SECONDARY_ROLES = NONE;` when testing lower-privileged roles) to verify dynamic masking and query the audit logs.
