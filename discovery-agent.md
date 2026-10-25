
# MySQL to PostgreSQL Data Migration and Column Mapping Specification

## 1. Purpose

Define source-to-target mappings for migrating the financial database from MySQL
to PostgreSQL using a Python ETL process.

The specification includes:

- Direct column mapping
- Combined column mapping
- Lookup and join rules
- Category and enumeration mapping
- Transformation status and confidence scores
- Referential integrity validation
- Source-to-target reconciliation
- Data-quality acceptance criteria

## 2. Source and target locations

| Property | Value |
|---|---|
| Source schema directory | `agents/schemas/source-mysql` |
| Target schema directory | `agents/schemas/target-postgresql` |
| Source database | MySQL |
| Target database | PostgreSQL / Google Cloud SQL |
| Migration method | Python ETL |
| Expected transaction volume | 100,000 rows in the sample dataset |
| Mapping specification | `agents/mysql_postgresql_column_mapping.md` |

**Important:** The schema directories must be parsed by the ETL project before
production migration. The mappings below are based on the schemas currently
available in the project discussion.

## 3. Mapping status and score definitions

### Mapping status

| Status | Meaning |
|---|---|
| PROPOSED | Mapping designed but not yet tested |
| SCHEMA_VERIFIED | Source and target column definitions checked |
| TRANSFORMATION_TESTED | Transformation tested against representative data |
| DATA_VALIDATED | Data-quality and reconciliation checks passed |
| APPROVED | Mapping approved for production |
| BLOCKED | Mapping has an unresolved issue |

Do not mark a mapping `DATA_VALIDATED` or `APPROVED` until the relevant checks
have actually been executed and their results recorded.

### Confidence score

Score each mapping from 0 to 100.

| Score | Interpretation |
|---|---|
| 95–100 | Clear, lossless mapping with strong validation |
| 85–94 | Well-defined mapping with limited transformation risk |
| 70–84 | Mapping requires additional edge-case testing |
| 50–69 | Significant ambiguity or possible information loss |
| 0–49 | Unresolved mapping; block production migration |

A score is a design-confidence estimate, not a measured data-quality result.
The scores below are provisional.

## 4. Source-to-target table mapping

| MySQL source table | PostgreSQL target table | Load order | Join/lookup dependency |
|---|---|---:|---|
| `customers` | `customers` | 1 | None |
| `accounts` | `accounts` | 2 | `accounts.customer_id` → customers |
| `merchants` | `merchants` | 1 | None |
| `cards` | `cards` | 3 | `cards.account_id` → accounts |
| `transactions` | `transactions` | 4 | Accounts, cards and merchants |

Customers and merchants can be loaded independently. Accounts must be loaded
after customers; cards after accounts; transactions after their referenced
entities.

Preserve source primary-key values during migration unless a separately
approved key-remapping strategy is introduced.

---

## 5. Customer column mapping

Source: MySQL `customers`  
Target: PostgreSQL `customers`

| Source column | Target column | Category | Transformation / join rule | Status | Score |
|---|---|---|---|---|---:|
| `customer_id` | `client_key` | Direct map / primary key | Preserve source ID; cast to PostgreSQL BIGINT | PROPOSED | 99 |
| `first_name` + `last_name` | `client_full_name` | Combined map | Trim each component and join with one space; handle NULL/empty values | PROPOSED | 90 |
| `email` | `contact_email` | Direct map / unique key | Preserve value; validate uniqueness and target length | PROPOSED | 99 |
| `phone` | `contact_phone` | Direct map | Preserve as text; normalize only under an approved phone-number policy | PROPOSED | 96 |
| `date_of_birth` | `birth_date` | Direct map | Convert DATE without changing the calendar date | PROPOSED | 99 |
| `created_at` | `registration_timestamp` | Direct map | Convert DATETIME to TIMESTAMP WITHOUT TIME ZONE | PROPOSED | 95 |
| `customer_status` | `client_state` | Category map | Validate ACTIVE, INACTIVE, SUSPENDED; preserve or explicitly translate values | PROPOSED | 98 |

### Customer combined mapping rule

```python
def combine_full_name(first_name, last_name):
    parts = [
        str(value).strip()
        for value in (first_name, last_name)
        if value is not None and str(value).strip()
    ]
    return " ".join(parts) or None
```

**Information-loss warning:** `client_full_name` cannot reliably reconstruct
the original `first_name` and `last_name` for every person. If reversibility
is required, retain the original components in additional target columns or
in an approved migration staging table.

---

## 6. Account column mapping

Source: MySQL `accounts`  
Target: PostgreSQL `accounts`

| Source column | Target column | Category | Transformation / join rule | Status | Score |
|---|---|---|---|---|---:|
| `account_id` | `ledger_key` | Direct map / primary key | Preserve source ID | PROPOSED | 99 |
| `customer_id` | `client_key` | Join / foreign key | Preserve ID and verify it exists in target `customers.client_key` | PROPOSED | 98 |
| `account_number` | `ledger_reference` | Direct map / unique key | Preserve value; validate uniqueness | PROPOSED | 99 |
| `account_type` | `ledger_category` | Category map | CURRENT, SAVINGS, CREDIT | PROPOSED | 98 |
| `currency_code` | `ledger_currency` | Direct map | Validate three-character currency code | PROPOSED | 98 |
| `current_balance` | `available_funds` | Direct map / numeric | Convert to NUMERIC(15,2); never use binary floating point | PROPOSED | 99 |
| `opened_at` | `activation_date` | Direct map | Preserve calendar date | PROPOSED | 99 |
| `account_status` | `ledger_state` | Category map | OPEN, FROZEN, CLOSED | PROPOSED | 98 |

### Account join rule

```sql
SELECT a.account_id, a.customer_id
FROM source_accounts AS a
LEFT JOIN source_customers AS c
    ON c.customer_id = a.customer_id
WHERE c.customer_id IS NULL;
```

The query should return zero rows. Run the equivalent orphan check on the
target after migration.

---

## 7. Merchant column mapping

Source: MySQL `merchants`  
Target: PostgreSQL `merchants`

| Source column | Target column | Category | Transformation / join rule | Status | Score |
|---|---|---|---|---|---:|
| `merchant_id` | `payee_key` | Direct map / primary key | Preserve source ID | PROPOSED | 99 |
| `merchant_name` | `payee_details` | Direct map | Preserve text; trim only if approved | PROPOSED | 96 |
| `merchant_type` | `payee_category` | Category map | GROCERY, RETAIL, UTILITIES, TRAVEL, DINING, OTHER | PROPOSED | 98 |
| `country_code` | `operating_country` | Direct map | Validate two-character country code | PROPOSED | 96 |

Merchant names are descriptive attributes, not reliable join keys. Use
`merchant_id` → `payee_key` for transaction references. If source and target
IDs are not preserved, a separate deterministic crosswalk is mandatory.

---

## 8. Card column mapping

Source: MySQL `cards`  
Target: PostgreSQL `cards`

| Source column | Target column | Category | Transformation / join rule | Status | Score |
|---|---|---|---|---|---:|
| `card_id` | `payment_instrument_key` | Direct map / primary key | Preserve source ID | PROPOSED | 99 |
| `account_id` | `ledger_key` | Join / foreign key | Map to `accounts.ledger_key`; validate referenced account exists | PROPOSED | 98 |
| `card_last_four` | `instrument_suffix` | Direct map | Preserve exactly four characters, including leading zeroes | PROPOSED | 99 |
| `card_type` | `instrument_category` | Category map | DEBIT, CREDIT | PROPOSED | 98 |
| `issued_at` | `activation_date` | Direct map | Preserve calendar date | PROPOSED | 99 |
| `expires_at` | `expiry_date` | Direct map | Preserve calendar date | PROPOSED | 99 |
| `card_status` | `instrument_state` | Category map | ACTIVE, BLOCKED, EXPIRED; preserve default semantics | PROPOSED | 98 |

### Card validation rules

- `(ledger_key, instrument_suffix)` must remain unique.
- Every `ledger_key` must reference an existing account.
- `expiry_date` must be later than `activation_date`.
- Do not infer a card's status from its expiry date unless a separate business
  rule explicitly requires this.
- Treat card suffixes as strings, not numbers.

---

## 9. Transaction column mapping

Source: MySQL `transactions`  
Target: PostgreSQL `transactions`

| Source column | Target column | Category | Transformation / join rule | Status | Score |
|---|---|---|---|---|---:|
| `transaction_id` | `event_key` | Direct map / primary key | Preserve source ID | PROPOSED | 99 |
| `account_id` | `ledger_key` | Join / foreign key | Map to `accounts.ledger_key` | PROPOSED | 98 |
| `related_account_id` | `counterparty_ledger_key` | Join / nullable foreign key | Preserve NULL; otherwise map to accounts | PROPOSED | 97 |
| `card_id` | `payment_instrument_key` | Join / nullable foreign key | Preserve NULL; otherwise map to cards | PROPOSED | 97 |
| `merchant_id` | `payee_key` | Join / nullable foreign key | Preserve NULL; otherwise map to merchants | PROPOSED | 97 |
| `transaction_type` | `event_category` | Category map | DEPOSIT, WITHDRAWAL, CARD_PAYMENT, TRANSFER, FEE, REFUND | PROPOSED | 98 |
| `amount` + `currency_code` | `money_details` | Combined map | JSONB object containing exact decimal amount and currency code | PROPOSED | 85 |
| `transaction_status` | `event_state` | Category map | PENDING, COMPLETED, FAILED, REVERSED | PROPOSED | 98 |
| `description` | `event_details` | Direct map | Preserve text; validate target length | PROPOSED | 96 |
| `transaction_at` | `event_timestamp` | Direct map | Convert DATETIME to TIMESTAMP WITHOUT TIME ZONE | PROPOSED | 95 |

### Transaction combined mapping

Recommended JSONB representation:

```json
{
  "amount": "125.50",
  "currency": "GBP"
}
```

Store the decimal amount as a decimal string in the JSON object to avoid
introducing binary floating-point rounding. Validate the JSON representation
and use explicit decimal conversion for calculations.

**Financial reporting consideration:** JSONB is convenient for combining
fields, but it is less convenient than separate typed columns for aggregation,
indexing and currency-specific financial calculations. If reports and
reconciliation are important, prefer separate `amount_value NUMERIC(15,2)`
and `currency_unit CHAR(3)` columns, or retain these typed columns alongside
the JSONB representation.

### Transaction join rules

- `account_id` must resolve to a target account.
- A non-NULL `related_account_id` must resolve to a target account.
- A non-NULL `card_id` must resolve to a target card.
- A non-NULL `merchant_id` must resolve to a target merchant.
- Preserve nullable references as NULL; do not replace them with zero or an
  arbitrary default key.
- Do not assume the transaction account must equal the card's linked account
  unless that rule exists in the source business model.

---

## 10. Data-type conversion rules

| MySQL type | PostgreSQL target type | Required handling |
|---|---|---|
| BIGINT UNSIGNED | BIGINT | Verify every source value is within signed BIGINT range |
| INT UNSIGNED | INTEGER or BIGINT | Verify target range before conversion |
| VARCHAR(n) | VARCHAR(n) or TEXT | Check maximum length and character encoding |
| CHAR(n) | CHAR(n) or VARCHAR(n) | Preserve meaningful leading zeroes and codes |
| ENUM | VARCHAR + CHECK constraint | Validate allowed values before insert |
| DECIMAL(p,s) | NUMERIC(p,s) | Use Decimal, never float, in ETL |
| DATE | DATE | Preserve date |
| DATETIME | TIMESTAMP WITHOUT TIME ZONE | Preserve local date/time fields; do not apply an arbitrary timezone shift |
| NULL | NULL | Preserve nullability and distinguish NULL from empty strings |
| Combined values | JSONB or typed target fields | Define schema, null handling, precision and validation |

MySQL `BIGINT UNSIGNED` can exceed PostgreSQL's signed BIGINT maximum
(9,223,372,036,854,775,807). Validate the actual source range before migration.

---

## 11. Join and lookup strategy

### Preferred strategy: preserve keys

When source primary keys are preserved, the ETL can map foreign keys directly:

| Source reference | Target reference |
|---|---|
| `accounts.customer_id` | `customers.client_key` |
| `cards.account_id` | `accounts.ledger_key` |
| `transactions.account_id` | `accounts.ledger_key` |
| `transactions.related_account_id` | `accounts.ledger_key` |
| `transactions.card_id` | `cards.payment_instrument_key` |
| `transactions.merchant_id` | `merchants.payee_key` |

### If keys are regenerated

Create explicit crosswalk tables, for example:

- `customer_key_map(source_customer_id, target_client_key)`
- `account_key_map(source_account_id, target_ledger_key)`
- `card_key_map(source_card_id, target_payment_instrument_key)`
- `merchant_key_map(source_merchant_id, target_payee_key)`

Never use names, email addresses, account numbers or card suffixes as a
replacement for primary-key mapping unless the migration design explicitly
defines and validates the business key.

---

## 12. Transformation quality validation

Run validation against a consistent source snapshot and the completed target
load. Record the actual query results in the migration execution report.

### 12.1 Row-count reconciliation

Compare source and target row counts for every table.

| Table | Expected source count | Target count | Difference | Acceptance |
|---|---:|---:|---:|---|
| customers | Measure at runtime | Measure at runtime | Source - target | 0 |
| accounts | Measure at runtime | Measure at runtime | Source - target | 0 |
| merchants | Measure at runtime | Measure at runtime | Source - target | 0 |
| cards | Measure at runtime | Measure at runtime | Source - target | 0 |
| transactions | 100,000 in sample dataset | Measure at runtime | Source - target | 0 |

Do not hard-code sample counts as production expectations. Obtain source counts
from the actual migration snapshot.

### 12.2 Primary-key validation

For each table, verify:

- No NULL primary keys
- No duplicate primary keys
- Source and target primary-key sets match when IDs are preserved
- No unexpected IDs were generated during the load

A matching row count alone is insufficient; different records can have the
same total count.

### 12.3 Foreign-key validation

Verify all non-NULL foreign keys resolve:

- Accounts → customers
- Cards → accounts
- Transactions → accounts
- Transactions → counterparty accounts
- Transactions → cards
- Transactions → merchants

Acceptance criterion: zero orphan references.

### 12.4 Combined-name validation

Recompute `client_full_name` from the source `first_name` and `last_name`
using the documented rule. Compare the normalized result with the target.

Acceptance criterion: zero mismatches, excluding only explicitly documented
and approved normalization exceptions.

### 12.5 Money validation

For every transaction:

1. Convert source `amount` to a decimal without using float.
2. Read the target amount from `money_details`.
3. Compare decimal values exactly at the source scale.
4. Compare currency codes exactly after any approved normalization.
5. Verify the JSON object has the expected keys and types.

Acceptance criterion: zero unexplained amount or currency mismatches.

For aggregate checks, group by currency. Never add amounts across different
currencies without a defined exchange-rate conversion.

### 12.6 Date and timestamp validation

Check that source and target date/time values represent the same values under
the documented conversion policy.

Acceptance criterion: zero unexplained date or timestamp differences.
Do not apply timezone conversion to MySQL DATETIME values unless the source
business rules establish the timezone semantics.

### 12.7 Category validation

For every enum-derived target column, verify that all target values belong
to the allowed category set.

Acceptance criterion: zero invalid category values.

### 12.8 Uniqueness and business constraints

Validate:

- `customers.contact_email` uniqueness
- `accounts.ledger_reference` uniqueness
- `(cards.ledger_key, cards.instrument_suffix)` uniqueness
- `cards.expiry_date > cards.activation_date`
- Numeric values fit the target NUMERIC precision and scale
- All text values fit target column lengths without silent truncation

### 12.9 Identity/sequence validation

If source IDs are explicitly inserted into PostgreSQL identity columns,
advance each PostgreSQL identity sequence beyond the maximum loaded ID.
Test that a subsequent generated insert does not collide with existing keys.

### 12.10 Financial reconciliation

Compare source and target transaction totals by currency and, where useful,
by transaction type and status.

Example reconciliation dimensions:

- Currency
- Transaction category
- Transaction status
- Transaction date or reporting period

Acceptance criterion: exact matching totals for each equivalent grouping,
subject to identical filtering and decimal semantics.

Do not compare a single total across currencies as if all amounts shared one
unit.

---

## 13. Mapping score calculation

Use a documented score instead of treating confidence as a subjective label.

Suggested weighted score:

| Dimension | Weight |
|---|---:|
| Semantic meaning is equivalent | 30% |
| Data type and precision compatibility | 20% |
| Nullability and constraint compatibility | 15% |
| Join and key integrity | 20% |
| Transformation is tested and reproducible | 15% |
| Total | 100% |

Each dimension receives a score from 0 to 100.

`mapping_score = sum(dimension_score * dimension_weight)`

Before testing, use provisional design estimates and label them as such.
After testing, record the measured result separately from the design score.

A combined mapping that loses source information should not receive a perfect
reversibility score even if its output is otherwise valid.

---

## 14. ETL execution controls

Recommended migration stages:

1. Parse and compare source and target DDL.
2. Verify every source column is mapped, intentionally excluded or approved
   as a source-only field.
3. Capture source row counts and source key ranges.
4. Extract a consistent source snapshot.
5. Transform data using explicit, version-controlled mapping functions.
6. Load parent tables before child tables.
7. Use bounded batches and transaction-safe retry handling.
8. Record rejected rows and errors without silently discarding records.
9. Run target constraint and data-quality validation.
10. Reconcile counts, keys, amounts and category distributions.
11. Reset identity sequences.
12. Produce a migration report and obtain approval before cutover.

Recommended audit fields in the ETL execution log:

- `run_id`
- Source and target schema versions
- Mapping specification version
- Start and finish timestamps
- Source and target table names
- Rows read, inserted, updated, rejected and skipped
- Error count and representative error details
- Validation results
- Final status

Never store database passwords or sensitive card data in the mapping document
or logs.

---

## 15. Mapping coverage and sign-off

Before production migration, generate a mapping inventory from both schema
directories and ensure every source column has exactly one documented outcome:

- Direct map
- Combined map
- Lookup/join
- Derived value
- Explicitly excluded with justification

A source column may contribute to a combined mapping, but it must not be
accidentally omitted from coverage checks.

### Acceptance criteria

- [ ] All source tables and columns inventoried
- [ ] All target tables and columns inventoried
- [ ] Every source column mapped or explicitly excluded
- [ ] All combined transformations documented and tested
- [ ] All foreign-key mappings verified
- [ ] Primary-key preservation or crosswalk strategy confirmed
- [ ] No unexplained row-count differences
- [ ] No unexplained primary-key differences
- [ ] No orphan foreign keys
- [ ] No unexplained monetary differences
- [ ] No invalid category values
- [ ] No silent truncation or numeric overflow
- [ ] Identity sequences reset and tested
- [ ] Validation report reviewed
- [ ] Production migration approved

## 16. Current status

**Overall status: PROPOSED — not yet validated against the actual schema files
or live data.**

The mapping becomes validated only after the source and target DDL are read,
the ETL transformation is executed, and the quality checks produce recorded
results.
