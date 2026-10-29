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
| `group_by` | array of strings | no | See [Grouping](#grouping). |
| `having` | array | no | Conditions on aggregates. |
| `order_by` | object | no | One `{"column", "direction"}`, not a list. |
| `limit` | integer | no | Default 100, maximum 10,000. |
| `offset` | integer | no | Default 0. |

Table and column names must be plain identifiers (letters, digits, underscores,
optionally `table.column`). Quoted or schema-qualified names are rejected. Every value
in filters and `having`, and the limit and offset, is sent as a bound parameter.

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
tenant is denied. See [Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets).

---

## Generated SQL

The query at the top of this page, minus the filter, becomes:

```sql
SELECT "public"."orders".status AS status, COUNT(id) AS orders
FROM "public"."orders"
GROUP BY status
ORDER BY status ASC
LIMIT $1
```

Tables are qualified with their schema, and each column gets an alias. When two
selected columns share a name, they come back as `<table>_<column>`.

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
  - fields used only in aggregates or grouping must still be visible to the agent.
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
