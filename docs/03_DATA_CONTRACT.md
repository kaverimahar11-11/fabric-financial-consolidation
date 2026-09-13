# 03. DATA CONTRACT & MULTI-CURRENCY SCHEMA SPECIFICATION

## 1. Target Schema Contract (`lh_finance_silver.stg_consolidated_transactions`)

All regional raw feeds (`US`, `EU`, `APAC`) must be transformed, type-cast, and standardized into this unified Silver target schema.

| Column Name | Target Data Type | Nullable | Validation & Business Logic |
| :--- | :--- | :--- | :--- |
| `transaction_id` | `StringType()` | **NO** | Primary key. Must be non-null and unique. |
| `source_region` | `StringType()` | **NO** | Hardcoded origin identifier (`US`, `EU`, `APAC`). |
| `transaction_date` | `DateType()` | **NO** | Cast to ANSI standard date (`YYYY-MM-DD`). |
| `amount` | `DoubleType()` | **NO** | Raw numerical amount in original native currency. Must be > 0. |
| `currency_local` | `StringType()` | **NO** | ISO 4217 currency code (`USD`, `EUR`, `GBP`, `AUD`, `SGD`, `JPY`). |
| `exchange_rate` | `DoubleType()` | **NO** | Applied FX market conversion rate to USD base currency via LOCF. |
| `base_amount_usd` | `DoubleType()` | **NO** | Calculated field: `amount_local * exchange_rate`. |
| `system_id` | `StringType()` | YES | Source system identifier (e.g., `APAC_SINGAPORE_CORE`). |
| `from_account_hash`| `StringType()` | YES | SHA-256 hashed account identifier for GDPR compliance. |
| `to_account_hash`  | `StringType()` | YES | SHA-256 hashed account identifier for GDPR compliance. |
| `ingested_at` | `TimestampType()` | **NO** | System timestamp recording when the record landed in Silver. |

---

## 2. Multi-Currency Engine Specification (PySpark LOCF)

To normalize non-USD currencies (`EUR`, `GBP`, `AUD`, `SGD`, `JPY`) into `base_amount_usd`:

1. **Market Rate Matching:** Join transaction records with daily market exchange rates on `transaction_date = rate_date` and `currency_local = currency`.
2. **Weekend & Holiday Handling (LOCF):** Foreign exchange markets close on weekends and holidays. To avoid `NULL` exchange rates, missing daily exchange rates must be forward-filled using PySpark windowing:
   `last("exchange_rate", ignorenulls=True).over(Window.partitionBy("currency").orderBy("date"))`
3. **USD Transactions:** Transactions already in `USD` bypass conversion with an explicit `exchange_rate = 1.0`.

---

## 3. Quality Gate & Dead-Letter Queue (DLQ) Trigger Rules

A record violates the contract and is routed to `quarantine_transactions` if:
* `transaction_id` is `NULL` or empty.
* `transaction_date` fails parsing (`NULL`).
* `amount_local` is `NULL` or `<= 0`.
* `currency_local` is not a supported ISO currency code.