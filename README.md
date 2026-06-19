# Enterprise Data Integration & CDC Engine

A Python ELT/ETL project scaffold for consolidating sales activity from an
acquired PostgreSQL commerce platform and a core SQL Server platform into a
single warehouse-ready sales fact table.

The design goal is an incremental, object-oriented pipeline that avoids full
source scans, standardizes messy source data, and preserves reporting history
while reflecting source-side updates and deletes.

## Problem Statement

An enterprise retail company has two transactional sales systems:

| Source | Platform | Shape | Integration challenge |
| --- | --- | --- | --- |
| Core sales | SQL Server | Historical and append-heavy | Extract only new or changed rows by checkpoint timestamp |
| Acquired sales | PostgreSQL | Regional sales in local currencies | Detect hard-deleted cancelled orders without destroying warehouse history |

The warehouse must produce a unified sales fact table suitable for daily
consolidated revenue reporting.

## Repository Layout

```text
.
├── config/
│   └── settings.py              # Runtime configuration and connection settings
├── data/
│   ├── exchange_rates.csv       # Currency reference data
│   ├── expected_rejects.csv     # Expected rejected records
│   ├── expected_sales_fact_run2.csv
│   ├── postgres_mock.csv        # PostgreSQL source snapshot, run 1
│   ├── postgres_mock_run2.csv   # PostgreSQL source snapshot, run 2
│   └── sqlserver_mock.csv       # Mock SQL Server source data
├── src/
│   ├── extractors/
│   │   ├── base.py              # Abstract extractor interface
│   │   ├── postgres.py          # PostgreSQL incremental extractor
│   │   └── sqlserver.py         # SQL Server incremental extractor
│   ├── loaders/
│   │   └── staging.py           # Raw/staging load utilities
│   ├── sql/
│   │   ├── staging_acquired.sql # PostgreSQL source standardization
│   │   ├── staging_core.sql     # SQL Server source standardization
│   │   └── warehouse_merge.sql  # CDC merge into warehouse fact table
│   └── transformers/
│       └── sql_runner.py        # SQL script execution helper
├── tests/
│   ├── test_extractors.py
│   └── test_transformers.py
├── main.py                      # Pipeline entry point
├── requirements.txt
└── README.md
```

## Target Architecture

1. Read the latest successful pipeline checkpoint.
2. Use source-specific extractor classes to pull only incremental records.
3. Load raw records into staging tables.
4. Execute SQL transformations to normalize schema, dates, SKUs, currency, and
   customer fields.
5. Reconcile hard deletes from PostgreSQL by marking missing warehouse records
   as `is_deleted = TRUE`.
6. Merge standardized rows into the final warehouse sales fact table.
7. Persist the new checkpoint after a successful run.

## Engineering Requirements

This project is intended to demonstrate:

- Abstract base classes for source extractor contracts.
- Polymorphic orchestration across SQL Server and PostgreSQL sources.
- Encapsulated loader and SQL execution classes.
- Incremental extraction with checkpoint timestamps.
- Warehouse CDC logic that preserves historical reporting rows.
- SQL-based transformation scripts for database-native processing.

Candidates should implement the following concrete behavior:

- `BaseExtractor` must be an abstract class with an incremental extraction
  method such as `extract_since(checkpoint)`.
- SQL Server and PostgreSQL extractors must be concrete subclasses.
- The orchestrator in `main.py` must process extractors polymorphically instead
  of branching through source-specific procedural code.
- SQL scripts in `src/sql/` must be executable by a reusable SQL runner class.
- The pipeline must be runnable locally without live SQL Server or PostgreSQL
  instances by using the provided CSV fixtures.
- Secrets and connection strings must not be hard-coded into source files.

## Data Quality Rules

The transformation layer should handle:

- Primary key collisions by namespacing transaction identifiers by source.
- Currency conversion into USD using `data/exchange_rates.csv`.
- JSON parsing of SQL Server `customer_blob` fields.
- Mixed date string normalization into ISO timestamps.
- SKU cleanup across prefixes such as `PROD-` and `SKU-`.
- Duplicate source transaction detection.
- Soft-delete marking for PostgreSQL rows that disappear from the source.

Rejected records should be written to a reject output with a source identifier,
transaction identifier, and reason.

## Provided Mock Datasets

### Source 1: SQL Server `core_sales`

The core source appends new transactions and tracks returns with an
`is_refunded` flag.

| tx_id | checkout_timestamp | customer_blob | sku_code | gross_amt_usd | tax_amt | is_refunded |
| --- | --- | --- | --- | ---: | ---: | ---: |
| 9001 | 2026-01-10 14:05:22 | `{"id":44, "country":"US"}` | PROD-9921 | 120.00 | 12.00 | 0 |
| 9002 | 2026-01-10 14:06:01 | `{"id":12, "country":"CA"}` | PROD-1102 | 45.50 | 4.55 | 0 |
| 9003 | 2026-01-11 09:12:00 | `{"id":88, "country":"US"}` | PROD-5541 | 300.00 | 30.00 | 1 |
| 9005 | 2026-01-11 10:15:00 | `{"id":103, "country":"US"}` | SKU-8821 | 75.00 | 7.50 | 0 |
| 9006 | 2026-01-11 10:20:00 | `{bad json` | PROD-7781 | 22.00 | 2.20 | 0 |
| 9007 | 2026-01-11 10:25:00 | `{"id":55, "country":"MX"}` |  | 18.00 | 1.80 | 0 |

### Source 2: PostgreSQL `acquired_sales`

The acquired source has transaction IDs that can overlap with the core system,
logs sales in regional currencies, and hard-deletes cancelled orders from the
source database.

Run 1 is stored in `data/postgres_mock.csv`.

| id | sale_date | customer_id | product_sku | total_price | currency | order_status |
| --- | --- | ---: | --- | ---: | --- | --- |
| 9001 | 10/01/2026 15:30:00 | 881 | SKU-9921 | 110.00 | EUR | COMPLETED |
| 9004 | 11/01/2026 11:22:10 | 402 | SKU-7721 | 85.00 | GBP | COMPLETED |
| 9005 | 11/01/2026 12:10:00 | 713 | PROD-8821 | 75.00 | USD | COMPLETED |
| 9005 | 11/01/2026 12:10:00 | 713 | PROD-8821 | 75.00 | USD | COMPLETED |
| 9006 | 2026/01/11 13:45:00 | 999 | SKU-1999 | 10000.00 | JPY | COMPLETED |

Run 2 is stored in `data/postgres_mock_run2.csv`. Transaction `9004` is missing
from run 2 and should be marked as deleted in the warehouse rather than
physically removed.

| id | sale_date | customer_id | product_sku | total_price | currency | order_status |
| --- | --- | ---: | --- | ---: | --- | --- |
| 9001 | 10/01/2026 15:30:00 | 881 | SKU-9921 | 110.00 | EUR | COMPLETED |
| 9005 | 11/01/2026 12:10:00 | 713 | PROD-8821 | 75.00 | USD | COMPLETED |
| 9008 | 12/01/2026 09:05:00 | 444 | SKU-3333 | 42.00 | GBP | COMPLETED |
| 9006 | 2026/01/11 13:45:00 | 999 | SKU-1999 | 10000.00 | JPY | COMPLETED |

## Expected Outputs

The final fact table produced after processing SQL Server data, PostgreSQL run
1, and PostgreSQL run 2 should match `data/expected_sales_fact_run2.csv`.

Required warehouse columns:

| Column | Description |
| --- | --- |
| `source_system` | `core` or `acquired` |
| `source_transaction_id` | Original transaction identifier from the source |
| `warehouse_transaction_id` | Source-prefixed unique key, such as `core:9001` |
| `checkout_timestamp` | Normalized timestamp in `YYYY-MM-DD HH:MM:SS` format |
| `customer_id` | Parsed or source-provided customer identifier |
| `customer_country` | Parsed SQL Server country when available |
| `sku` | Prefix-stripped product identifier |
| `gross_amount_usd` | Amount normalized to USD |
| `tax_amount_usd` | Tax amount in USD, or `0.00` when unavailable |
| `is_refunded` | Refund flag from SQL Server; false for acquired records |
| `is_deleted` | True only when a previously seen acquired record disappears |

Rejected rows should match `data/expected_rejects.csv`.

## Deliverables

Recommended time box: 3 to 5 hours. Favor correctness, clarity, and testability
over production infrastructure.

Candidates should submit:

- A runnable Python pipeline with `main.py` as the entry point.
- Implemented extractor, loader, and SQL runner classes.
- SQL scripts in `src/sql/` for staging, standardization, and warehouse merge.
- Tests covering extractor behavior, transformation behavior, and the final
  expected output.
- A short note in the README describing assumptions and how to run the project.
- A brief AI collaboration note if generative AI tools were used.

## AI Collaboration Policy

Candidates may use generative AI tools such as ChatGPT, Claude, GitHub Copilot,
or similar assistants. AI use is permitted because these tools are common in
modern engineering workflows.

However, **candidates are responsible for every line they submit.** They **should be
able to explain** the Python design, SQL transformations, CDC behavior, test
strategy, and tradeoffs in a follow-up technical discussion. AI-generated code
that cannot be explained, tested, or debugged by the candidate should be treated
as incomplete.

## Evaluation Rubric

| Area | Weight | What to look for |
| --- | ---: | --- |
| Python design | 30% | Clean OOP, abstraction, testable classes, clear orchestration |
| SQL correctness | 35% | Accurate transformations, joins, currency math, date parsing, merge logic |
| CDC and deletes | 15% | Incremental checkpointing and correct hard-delete reconciliation |
| Tests | 10% | Meaningful unit and integration coverage using fixtures |
| Operability | 10% | Clear setup, deterministic local run, readable documentation |

## Setup

Create and activate a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

## Running the Pipeline

The intended entry point is:

```bash
python main.py
```

The implementation modules are placeholders and should be filled in by the
candidate. A complete solution should produce outputs equivalent to
`data/expected_sales_fact_run2.csv` and `data/expected_rejects.csv`.

## Testing

Run the test suite with:

```bash
pytest
```

The test files are currently placeholders. Recommended coverage includes:

- Extractor checkpoint filtering.
- Source-specific ID namespacing.
- Currency normalization.
- SQL script discovery and execution order.
- PostgreSQL hard-delete reconciliation behavior.

## Current Status

- Repository structure is in place.
- Currency reference data is available in `data/exchange_rates.csv`.
- Mock source CSVs and expected output CSVs are populated in `data/`.
- Python modules, SQL scripts, and tests still need implementation.
