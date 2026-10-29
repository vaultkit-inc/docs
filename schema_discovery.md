# Schema Discovery and the Registry

**Applies to:** VaultKit 0.9.0-alpha with vkit 1.2.0

Policies act on labels, not column names. The **registry** lists every dataset VaultKit
knows, with each field's **sensitivity** and **category**. A scan fills it in for you;
you review it in git and deploy it with your policies.

## The workflow

```bash
vkit scan prod_db                    # what changed since the last scan
vkit scan prod_db --mode apply       # save the scan to the control plane
vkit registry export                 # write it to datasets/registry.yaml
# review and edit datasets/registry.yaml, then:
git commit -am "Review registry labels"
vkit policy bundle && vkit policy deploy
```

The deployed bundle carries the registry, so the labels you commit are the labels
policies use.

## registry.yaml

```yaml
customers:
  datasource: prod_db
  fields:
    id:          { type: integer, sensitivity: low,     category: internal }
    email:       { type: text,    sensitivity: medium,  category: pii }
    ssn:         { type: text,    sensitivity: high,    category: pii }
    preferences: { type: jsonb,   sensitivity: unknown, category: unknown }
tickets:
  datasource: prod_db
  tenant_column: organization_id
  fields:
    organization_id: { type: uuid, sensitivity: low, category: internal }
```

A field's category is the tag that policies match with `contains`. You can use your own
categories, for example `marketing`, and match them in policies the same way.
`tenant_column` marks a dataset as tenant-scoped; see
[Tenant-scoped datasets](policy_guide.md#tenant-scoped-datasets).

## The labels a scan suggests

| Sensitivity | Category | Columns like |
| --- | --- | --- |
| `high` | `credential` | `password`, `password_digest`, `api_key`, `access_token`, `client_secret`, `private_key`; the value column of a table of secrets |
| `high` | `pii` | government IDs (`ssn`, `passport_number`, `tax_id`), payment cards (`card_number`, `cvv`, `cc_last4`), health (`diagnosis`, `medical_history`), special-category data (`ethnicity`, `religion`) |
| `high` | `financial` | bank accounts (`iban`, `account_number`, `routing_number`) |
| `medium` | `pii` | email, phone, names, addresses, date of birth, IP addresses and user agents, location, gender |
| `medium` | `financial` | amounts (`amount`, `total`, `balance`, `price`), compensation (`salary`, `income`) |
| `low` | `internal` | IDs, timestamps, flags, counts, `status`, `sort_order`, `product_name`, `currency` |
| `unknown` | `unknown` | text or JSON columns it couldn't place: `data`, `value`, `payload`, `meta`, `notes`, `body`, `description` |

How it decides:

- **Whole words.** `last_login_ip`, `lastLoginIp` and `IPAddress` are all read as
  words, so `description` isn't mistaken for an IP address, or `company` for a card
  number.
- **The table matters.** A plain `name` is a person only in a table of people
  (`customers.name`, not `products.name`).
- **The type matters.** Flags and timestamps about a value aren't the value:
  `email_verified` and `password_changed_at` are `low`.
- **`low` needs evidence.** A column is `low` only if its type can't hold free text,
  or its name is clearly harmless. Otherwise it's `unknown`.

The scan reads column names and types only, never data. A column named `data` that
holds emails is `unknown`, not `pii`: that's what review is for.

## Reviewing

- **Give every `unknown` column a real label.** The `starter` pack allows `unknown`
  columns but records them in the audit log, and `ai_safety` makes agents ask for
  approval to read them in production.
- **Raise anything the scan underrated.** Policies only protect what's labelled.
- **Lower anything it overrated** if you're sure, for example a `notes` column that
  never holds personal data.

## Rescans

A rescan never overwrites labels you've reviewed.

- **Reviewed labels** come from a deployed bundle or a manual edit. A rescan keeps
  them, and where the classifier now disagrees, it lists a suggestion:

  ```
    api_keys                 ✓
      ? key_value  low/internal → high/credential (suggested)
  ```

  To accept a suggestion, edit `datasets/registry.yaml` and redeploy.
- **Unreviewed labels** are `unknown` labels, and labels that only came from an earlier
  scan. A rescan replaces them with the current classification.
- **Columns:** a rescan adds new columns, drops removed ones and updates types.
