# VaultKit Policy Schema (Reference)

## Overview
Each policy file in `config/policies/*.yaml` defines **one** rule that governs
how VaultKit handles access to a specific dataset under specific conditions.

A policy contains four major sections:

1. **id** — unique policy identifier  
2. **description** — human-readable explanation  
3. **match** — what data + contexts trigger this policy  
4. **action** — what VaultKit should do when it matches  

Below is the full reference schema.

---

## VaultKit Policy Schema (YAML)

```yaml
id: <string>                # REQUIRED — unique identifier
description: <string>       # REQUIRED — human-readable explanation

priority: <integer>         # OPTIONAL — higher evaluated first

match:                      # OPTIONAL — data matching rules
  dataset: <dataset_name>   # OPTIONAL
  fields:                   # OPTIONAL
    sensitivity: <tag>
    category: <tag>
    contains: [<tag>]
    any: [field1, field2]
    all: [field1, field2]

context:                    # OPTIONAL — identity & environment constraints
  requester_role: <role>
  requester_clearance: <string>
  requester_region: <region>
  dataset_region: <region>
  environment: <env>

  time:
    after: "HH:MM"
    before: "HH:MM"
    timezone: "UTC"

action:                     # REQUIRED — exactly ONE must be true
  deny: true
  require_approval: true
  mask: true
  allow: true

  approver_role: <role>     # REQUIRED if require_approval
  reason: <string>          # REQUIRED for deny / require_approval
  ttl: "1h"                 # OPTIONAL — grant duration
```

---

## Priority Rules
When multiple policies match, VaultKit resolves using the following priority:

| Priority | Action             |
|---------|--------------------|
| 3       | deny               |
| 2       | require_approval   |
| 1       | mask               |
| 0       | allow              |

So a `deny` rule always overrides all other matched policies.

---

## Example Minimal Template

```yaml
id: example_policy
description: "Example showing every top-level field."

match:
  dataset: customers
  fields:
    sensitivity: pii
  context:
    requester_role: analyst
    environment: production
    time:
      after: "09:00"
      before: "17:00"

action:
  mask: true
  reason: "PII is masked for analysts during business hours."
  ttl: "1h"
```
