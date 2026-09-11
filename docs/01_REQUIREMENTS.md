# Project Requirements: Multi-Currency Financial Consolidation & P&L Platform

## 1. Executive Summary & Background
A global holding company headquartered in Belgium operates three regional operating entities (US, EU, and APAC). Each regional entity manages its accounting operations on disparate software backends and reports locally in regional currencies.

Currently, the Corporate Finance team performs manual month-end financial consolidation across regional entities using Excel spreadsheets. This legacy workflow takes over 20 days per close cycle, lacks point-in-time auditability, and introduces substantial risk of human error during currency conversions and chart-of-accounts mapping.

The goal of this project is to architect and deploy an automated, enterprise-grade Data Consolidation Platform in **Microsoft Fabric** using a **Medallion Architecture (Bronze -> Silver -> Gold)**. The platform ingests nightly transactional feeds, cleanses and standardizes unstructured payloads, performs multi-currency conversion using dynamic market exchange rates, and delivers a unified USD-denominated Profit & Loss (P&L) dataset accessible via Power BI Direct Lake mode.

---

## 2. Regional Source Systems Overview

| Regional Entity | Geographic Region | System Architecture / Style | Local Reporting Currency |
| :--- | :--- | :--- | :--- |
| **US HQ** | United States | Clean relational database (Azure SQL exports) | **USD** |
| **EU Ops** | Europe (Belgium HQ) | Flat-file file gateway exports | **EUR** |
| **APAC** | India & SE Asia | Legacy core system with unstructured narratives | **INR** |

### Raw Ingestion Source Specifications
1. `us_transactions.csv`: Well-structured CSV format, ISO timestamps (`YYYY-MM-DD`), native USD amounts, explicit General Ledger (GL) account codes.
2. `eu_transactions.csv`: Flat CSV exports, European date formatting (`DD/MM/YYYY`), EUR amounts, regional credit/debit sign conventions requiring standardization.
3. `apac_transactions.csv`: Unstructured exports, INR amounts, inconsistent date strings (`DD-MM-YYYY`, `YYYY/MM/DD`, or corrupt values), free-text payment narratives (e.g., `NEFT-CR-OFFICE_SUPPLIES`, `RTGS/VENDOR_PAYMENT`).

---

## 3. Project Objectives & Target Output

### Primary Core Objective
Automate the nightly end-to-end ingestion, cleansing, transformation, and dimensional modeling of all global financial transactions to support an executive USD Profit & Loss report.

### Target Gold Analytics Engine (`wh_financial_gold`)
A Kimball Star Schema hosted in a Fabric Synapse Data Warehouse containing:
* **`Fact_ConsolidatedTransactions`**: Fact table holding individual transactional line items normalized to USD while retaining local host currency attributes (`amount_local`, `amount_usd`, `exchange_rate`).
* **`Dim_Account`**: Standardized Chart of Accounts (COA) hierarchy mapping regional system codes to master P&L categories.
* **`Dim_Currency`**: Currency reference dimension with ISO codes and regional metadata.
* **`Dim_Date`**: Standard corporate fiscal date dimension.
* **Power BI Serving Layer**: Direct Lake semantic model providing sub-second P&L reporting by Account Category, Regional Entity, and Month.

---

## 4. Data Quality Contract & Schema Validation Rules

All raw incoming feeds must comply with the master Data Contract upon ingestion into Silver. Records failing strict validation checks are isolated into a Dead-Letter Queue (`quarantine_transactions`) to ensure downstream P&L reports are never corrupted.

| Field Name | Data Type | Nullable | Validation & Governance Rules |
| :--- | :--- | :--- | :--- |
| `transaction_id` | String | **No** | Must be non-null and unique within the originating source system. |
| `amount_local` | Double | **No** | Must be > 0 (numerical sign handled via debit/credit classification). |
| `currency` | String | **No** | Must be a recognized ISO currency code (`USD`, `EUR`, `INR`). |
| `transaction_date` | Date | **No** | Must parse cleanly to a valid calendar date (`YYYY-MM-DD`). |
| `account_category` | String | Yes | Mapped via `Dim_Account`; unmapped codes route to quarantine. |
| `payment_narrative` | String | Yes | Raw free-text string parsed in Silver via regex to extract expense categories. |

---

## 5. Architectural & Technical Acceptance Criteria

1. **Bronze Ingestion:** All raw source files (`us_transactions`, `eu_transactions`, `apac_transactions`) land in `lh_finance_bronze` as unmodified, raw payloads preserving history.
2. **Silver Medallion Pipeline:**
   * Cleanses and standardizes date formats across all regions into standard ANSI `DateType()`.
   * Parses unstructured APAC payment narratives using PySpark regular expressions (Regex).
   * Applies Last Observation Carried Forward (LOCF) windowing to convert local currencies (`EUR`, `INR`) to `base_amount_usd` using daily market exchange rates.
   * Achieves **< 2% quarantine routing rate** once initial GL account mappings are populated.
3. **Gold Analytics Warehousing:** Exposes a fully modeled Kimball Star Schema enabling instantaneous multi-dimensional aggregation (P&L by region, account category, and fiscal period).
4. **Direct Lake Integration:** Serves Power BI dashboards via Direct Lake mode against OneLake Delta Parquet files with zero import schedules or DirectQuery performance bottlenecks.

---

## 6. Out-of-Scope (Version 1 Release)

To preserve focus on core P&L financial consolidation, the following items are explicitly excluded from V1 scope:
* Real-time / streaming ingestion (batch nightly pipeline runs are sufficient).
* Expansion beyond the three primary currencies (`USD`, `EUR`, `INR`).
* Automated Anti-Money Laundering (AML) / Suspicious Activity Report (SAR) compliance filings (this platform is built exclusively for financial P&L consolidation).
