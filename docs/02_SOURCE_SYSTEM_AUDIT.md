# 02. SOURCE SYSTEM AUDIT REPORT

## Source Feeds Inventory

### 1. US Region (`us_transactions.csv`)
* **Format:** CSV
* **Key Observations:** Ingested via OneLake raw landing zone; standard tabular structure.
* **Currency:** USD

### 2. EU Region (`eu_transactions.csv`)
* **Format:** CSV
* **Key Observations:** Contains non-USD currencies (`EUR`, `GBP`).
* **Date Parsing:** Standardized string parsing required for Silver ingestion.

### 3. APAC Region (`apac_region_parquet`)
* **System ID:** `APAC_SINGAPORE_CORE`
* **Format:** Parquet
* **Currencies Discovered:** Multi-currency (`AUD`, `SGD`, `JPY`).
* **Data Quality & Schema Structure:** Clean, fully structured columnar Parquet feed. No free-text payment narratives present (regex parsing not required).
* **Governance / PII Requirements:** Raw banking account fields (`from_account`, `to_account`) contain IBANs and must be obfuscated using SHA-256 hashing prior to landing in Silver.
