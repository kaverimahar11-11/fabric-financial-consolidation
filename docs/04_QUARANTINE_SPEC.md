# 04. DEAD-LETTER QUEUE (DLQ) & GOVERNANCE SPECIFICATION

## 1. Dead-Letter Queue Architecture (`quarantine_transactions`)

To maintain non-blocking batch execution and satisfy strict SLA targets, data quality violations will not trigger pipeline exceptions. Instead, failing records are isolated into a DLQ table for audit and remediation.

### Target Schema: `lh_finance_silver.quarantine_transactions`

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `quarantine_id` | `StringType()` | UUID generated for quarantine tracking. |
| `source_region` | `StringType()` | Feeder region identifier (`US`, `EU`, `APAC`). |
| `raw_record` | `StringType()` | JSON representation of the original unparsed row. |
| `error_code` | `StringType()` | Primary business validation rule violated. |
| `error_message` | `StringType()` | Detailed diagnosis of the failure condition. |
| `quarantined_at` | `TimestampType()` | System ingestion timestamp when isolated. |

---

## 2. Error Code Matrix & Routing Rules

| Error Code | Trigger Condition | Severity |
| :--- | :--- | :--- |
| `ERR_NULL_KEY` | `transaction_id` is NULL or empty string. | Critical |
| `ERR_INVALID_DATE` | Date parsing produces NULL (e.g., malformed format). | High |
| `ERR_INVALID_AMOUNT` | `amount_local` is NULL or `<= 0`. | High |
| `ERR_UNSUPPORTED_CURRENCY` | `currency_local` is missing from FX rate table. | High |

---

## 3. GDPR Compliance & PII Obfuscation Rule

To comply with GDPR and financial data privacy regulations:
* Raw bank account numbers (`from_account`, `to_account`) must never land in clear text within Silver or Gold tables.
* Hashing Transformation Rule: Apply `sha2(col("account_number"), 256)` during Silver ingestion.