# AQL Syntax & Specification

**Applies to:** VaultKit 0.9.0-alpha (control plane and funl runtime)

AQL (Access Query Language) is the JSON format for asking VaultKit for data. A query
says which table, columns, filters and aggregates you want; it never contains SQL.

How a query runs:

1. The **control plane** checks it, applies the row limit, adds the tenant filter for
   tenant-scoped datasets, evaluates your policies, and decides: allow, mask, require
   approval or deny.
2. If it's allowed, the control plane issues a grant and sends the query, with the
   fields to mask, to **funl**.
3. **funl** translates it into parameterised SQL, applies the masking, runs it against
   the datasource, and returns rows.

Only PostgreSQL datasources are supported today.

---

## Where queries are sent

| Who | Endpoint | Body |
| --- | --- | --- |
| People and scripts (`vkit request`) | `POST /api/v1/orgs/<org>/requests` | `{"aql": { … }}` — the full format on this page |
| Agents (Python SDK, MCP server) | `POST /api/v1/orgs/<org>/intent/requests` | `{"request": { … }}` — the [intent format](#the-intent-format-for-agents), a simpler subset |

---

## Top-level structure

```json
{
  "source_table": "orders",
  "columns": ["status"],
  "joins": [],
  "filters": [{"field": "status", "operator": "neq", "value": "refunded"}],
  "aggregates": [{"func": "count", "field": "id", "alias": "orders"}],
  "group_by": ["status"],
  "having": [],
  "order_by": {"column": "status", "direction": "ASC"},
  "limit": 100,
  "offset": 0
}
```

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `source_table` | string | yes | A dataset in your registry. `dataset` is accepted as an alias. |
| `columns` | array of strings | no | Columns, or `table.column` with joins. Aggregate expressions such as `"MAX(amount)"` are also allowed. With no columns and no aggregates, every column is selected. |
| `joins` | array | no | See [Joins](#joins). |
| `filters` | array | no | Combined with AND. See [Filters](#filters). |
| `aggregates` | array | no | See [Aggregates](#aggregates). |
| `group_by` | array of strings or time buckets | no | See [Grouping](#grouping) and [Time buckets](#time-buckets). |
| `having` | array | no | Conditions on aggregates. |
| `order_by` | object | no | One `{"column", "direction"}`, not a list. |
| `limit` | integer | no | Default 100, maximum 10,000. |
| `offset` | integer | no | Default 0. |

Table and column names must be plain identifiers (letters, digits, underscores,
optionally `table.column`). Quoted or schema-qualified names are rejected. Write them
exactly as the registry has them, case included: `firstName`, `riskScore`. Every value
in filters and `having`, and the limit and offset, is sent as a bound parameter.

Every dataset and field a query names must be in the deployed registry (0.12.0-alpha):
in columns, filters, aggregates, grouping, sorting, `having` and joins.

- **An unknown field** is refused with `422`, listing the fields you can use:
  "Unknown field `brand` on `payment_methods`. Valid fields: id, card_brand, …".
- **A dataset that isn't in the registry, or that is hidden from you,** gets the same
  `404 {"error": "Dataset not found", "request_id": "…"}` on both APIs. So does a
  tenant-scoped dataset for an agent with no tenant.

---

## Joins

```json
"joins": [
  {"type": "LEFT", "table": "customers", "alias": "c",
   "left_field": "orders.customer_id", "right_field": "c.id"}
]
```

- `type`: `INNER` (the default), `LEFT`, `RIGHT` or `FULL`.
- `alias` is optional. Once a joined table has one, references written as
  `customers.email` are rewritten to `c.email` for you.
- Every joined table must be in your registry. Policies are evaluated for each table
  the query touches, using the fields it references on that table.

---

## Filters

A filter compares one field to a value. Top-level filters are combined with AND.

```json
"filters": [
  {"field": "status", "operator": "eq", "value": "failed"},
  {"logic": "OR", "conditions": [
    {"field": "amount", "operator": "gte", "value": 1000},
    {"field": "region", "operator": "in", "value": ["EU", "CA"]}
  ]}
]
```

| Operator | Meaning |
| --- | --- |
| `eq`, `neq` | equal, not equal |
| `gt`, `gte`, `lt`, `lte` | greater or less than (or equal) |
| `like` | contains: the value is matched as `%value%` |
| `in` | one of a non-empty array of values |
| `is_null`, `is_not_null` | no `value` needed |

- A group is `{"logic": "AND" | "OR", "conditions": [ … ]}`, nested as deep as you
  need, and is always parenthesised in the SQL.
- Filters can use `hash`-masked fields: the column is hashed before it's compared.
  Fields masked `full` or `partial` can be returned masked, but not used to filter,
  group or sort.

---

## Aggregates

```json
"aggregates": [
  {"func": "count", "field": "id", "alias": "orders"},
  {"func": "sum", "field": "total", "alias": "revenue"}
]
```

- `func`: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, in any case.
- `field`: a column, `table.column`, or `*` (only with `COUNT`).
- `alias`: the output column name, which `order_by` can use.
- Over a masked field, only `COUNT` is allowed: `SUM`, `AVG`, `MIN` and `MAX` would
  reveal the raw values, so the request is denied.

---

## Grouping

With aggregates, the plain `columns` are the grouping columns. `group_by` makes the
grouping explicit. Grouping or sorting by a field masked `full` or `partial` is
denied; `hash`-masked fields are allowed.

### Time buckets

A `group_by` entry is a column name, or a **time bucket** of a timestamp or date
column. A bucket groups rows by the day, week, month, quarter or year they fall in,
so a trend ("failed payments per quarter") is one grouped query instead of one query
per period.

```json
{
  "source_table": "payments",
  "aggregates": [{ "func": "count", "field": "*", "alias": "payments" }],
  "filters": [{ "operator": "eq", "field": "status", "value": "failed" }],
  "group_by": [{ "field": "created_at", "bucket": "quarter", "alias": "quarter", "timezone": "America/Toronto" }],
  "order_by": { "column": "quarter", "direction": "ASC" }
}
```

Generated SQL (PostgreSQL):

```sql
SELECT to_char(date_trunc('quarter', payments.created_at AT TIME ZONE 'America/Toronto'), 'YYYY-MM-DD') AS quarter,
       COUNT(*) AS payments
FROM payments
WHERE payments.status = $1
GROUP BY to_char(date_trunc('quarter', payments.created_at AT TIME ZONE 'America/Toronto'), 'YYYY-MM-DD')
ORDER BY quarter ASC
```

Each row has the first day of its period under the alias, as a `YYYY-MM-DD` date in
the bucket's time zone (for example `2024-04-01` for the second quarter of 2024),
and the aggregates. Periods with no rows are not returned.

| Key | Required | Description |
|-----|----------|-------------|
| `field` | yes | A timestamp or date column of the dataset (or of a joined table, qualified). |
| `bucket` | yes | `day`, `week`, `month`, `quarter` or `year`. Weeks start on Monday (ISO 8601). |
| `alias` | no | The result column's name. Defaults to `<field>_<bucket>`, e.g. `created_at_quarter`. `order_by` can use it. |
| `timezone` | no | An IANA time zone such as `UTC` or `America/Toronto`. Periods start at midnight in that zone. Defaults to `UTC`. |

Rules:

- A bucket needs `aggregates`, and can be mixed with plain columns:
  `"group_by": ["status", {"field": "created_at", "bucket": "month"}]`.
- The bucketed column itself can't also be selected in `columns` (it isn't grouped).
- `timezone` applies only to `timestamp with time zone` columns. Dates and timestamps
  without a time zone are bucketed as stored; `UTC` is accepted for them and ignored.
- The alias must be a simple name and differ from the other columns and aggregate
  aliases.
- An unknown bucket, a column that isn't a timestamp or date, or an unknown time zone
  is rejected with `422` and a message that says which.
- **Masking:** a bucket follows the same rules as grouping by its column. Bucketing a
  fully or partially masked column is denied (`GROUP BY date_of_birth` in the reason).
  A hash-masked column can be bucketed; its bucket values are returned hashed.
- **Grants and approvals:** the bucket, alias and time zone are part of the query's
  fingerprint, so a grant for quarters doesn't cover months, and a grant for UTC
  quarters doesn't cover Toronto quarters.
- Translation is per connector: PostgreSQL uses `date_trunc` and `to_char`. Other
  engines map the same units to their own functions.

---

## Having

```json
"having": [{"field": "COUNT(id)", "operator": "gt", "value": 5}]
```

Operators: `gt`, `gte`, `lt`, `lte`, `eq`. Conditions are combined with AND.

---

## Order by

```json
"order_by": {"column": "revenue", "direction": "DESC"}
```

One column, an aggregate alias or an aggregate expression. `direction` is `ASC`
(the default) or `DESC`.

---

## Limit and offset

The control plane sets a limit of 100 rows when none is given, and refuses more than
10,000, zero, negatives and non-integers. `offset` defaults to 0.

---

## Masking

Policies decide which fields are masked and how; queries don't ask for it. Masking is
applied to the returned values:

| Method | Returns |
| --- | --- |
| `full` | `*****` |
| `partial` | the first three characters, then `****` |
| `hash` | a SHA-256 hex digest |

Aggregates over masked fields are masked too, except `COUNT`. See the
[Policy Guide](policy_guide.md) for how masking policies combine.

---

## Tenant-scoped datasets

For a dataset marked with `tenant_column`, the control plane adds
`<table>.<tenant_column> = <your tenant>` to every query that reads it, ANDed with
your filters, for the source table and every joined table. A requester without a
tenant can't read it: an agent gets "Dataset not found" (0.12.0-alpha), a person a
`403` that says why. See [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets).

---

## Generated SQL

The query at the top of this page, minus the filter, becomes:

```sql
SELECT "public"."orders"."status" AS "status", COUNT("id") AS "orders"
FROM "public"."orders"
GROUP BY "status"
ORDER BY "status" ASC
LIMIT $1
```

Tables are qualified with their schema, and each column gets an alias. Every name is
quoted (0.12.0-alpha), so mixed-case columns keep their case and reserved words such
as `order` or `user` work. Results are keyed by the names you asked for, case
included. When two selected columns share a name, they come back as
`<table>_<column>`.

### When a query fails to run

The database's errors are reported clearly (0.12.0-alpha), with a `code` and the
`correlation_id`, and never with SQL, parameter values or connection details:

| Problem | Response |
|---|---|
| A column or table the database doesn't have (`undefined_column`, `undefined_table`) | `422`: "Column `x` doesn't exist on `dataset`", asking for a rescan |
| A value of the wrong type, or an incompatible comparison (`type_mismatch`) | `422`, for example "A filter value isn't a valid bigint." |
| The datasource's login can't read the data (`permission_denied`) | `403` |
| The datasource is busy (`datasource_busy`) | `503` with `Retry-After` |
| The query ran past the timeout (`query_timeout`) | `504` |
| Anything else | `502` "Query engine error" |

The failed run is recorded on the request. Admins see its SQL, with placeholders and
no parameters, on the Activity page.

---

## The intent format for agents

Agents send a simpler request through the intent API, which the Python SDK and MCP
server use:

```json
{
  "request": {
    "dataset": "orders",
    "fields": ["status"],
    "aggregates": [{"func": "sum", "field": "total", "alias": "revenue"}],
    "group_by": ["status"],
    "filters": [{"field": "created_at", "operator": "gte", "value": "2026-10-01"}],
    "order_by": {"column": "revenue", "direction": "DESC"},
    "limit": 50,
    "purpose": "Weekly revenue summary for the finance channel"
  }
}
```

Differences from full AQL:

- **No joins,** and `offset` is always 0.
- **Operators** are the named ones only (no `=` or `>` shorthands). `logic` must be
  `AND` or `OR`, groups can't be empty, and nesting is limited to 5 levels.
- **Fields:**
  - `fields` can't contain `*`, and a request needs fields or aggregates;
  - fields are bare names or qualified with the request's own dataset;
  - fields used only in aggregates or grouping must still be visible to the agent;
  - every field must be in the registry. Call `vaultkit_describe` (or
    `GET /aql/datasets/:name/schema`) to list them.
- **Aggregates and having:**
  - aggregate aliases must be unique, simple names;
  - with an explicit `group_by`, every selected field must be in it;
  - `having` needs aggregates, a numeric value, and an aggregate expression as its
    field.
- **`purpose`** is recorded with the request and shown to approvers. Rewording it
  doesn't trigger a new approval.

---

## Example: a join with a filter

```json
{
  "source_table": "orders",
  "columns": ["orders.id", "orders.total", "c.region"],
  "joins": [{"type": "INNER", "table": "customers", "alias": "c",
             "left_field": "orders.customer_id", "right_field": "c.id"}],
  "filters": [{"field": "orders.status", "operator": "eq", "value": "completed"}],
  "order_by": {"column": "orders.total", "direction": "DESC"},
  "limit": 20
}
```
