# VaultKit Policy Pack System

> Versioned, layered, dependency-aware governance distribution for VaultKit.

The Policy Pack system allows VaultKit to ship secure, production-ready governance defaults in a structured, upgradeable format.

Policy Packs solve one core problem:

> Organizations should not have to design secure governance from scratch.

Instead, they can install curated, versioned packs that enforce secure defaults and domain-specific protections.

---

## 1. What is a Policy Pack?

A Policy Pack is:

- A versioned collection of policies
- Distributed with the VaultKit CLI
- Layer-aware
- Dependency-aware
- Upgradeable
- Drift-detectable
- Deterministically installable
- Audit-traceable

Each pack lives inside:
```
lib/vkit/policy/packs/<pack_name>/
```

Structure:
```
<pack_name>/
  metadata.yaml
  policies/
    01_policy.yaml
    02_policy.yaml
```

---

## 2. Pack Metadata Schema

Each pack must include `metadata.yaml`.

Example:
```yaml
__pack_meta:
  name: starter
  version: "1.0.0"
  layer: foundation
  description: "Secure baseline governance rules."
  dependencies: []
  priority_band:
    min: 100
    max: 199
```

### Fields

| Field | Required | Description |
| --- | --- | --- |
| `name` | ✅ | Pack identifier |
| `version` | ✅ | Semantic version string |
| `layer` | ✅ | `foundation` / `domain` / `custom` |
| `description` | ❌ | Human-readable summary |
| `dependencies` | ❌ | Required packs |
| `priority_band` | ❌ | Enforced priority range |

---

## 3. Pack Layers

Layers define enforcement precedence.

- Lower layers form baseline security.
- Higher layers refine behavior.

| Layer | Purpose |
| --- | --- |
| `foundation` | Core safety defaults |
| `domain` | Domain-specific governance |
| `custom` | Organization-authored rules |

Installation order must respect:
```
foundation → domain → custom
```

VaultKit enforces dependency presence but does not auto-install.

---

## 4. Policy Installation Model

When a pack is installed:

1. Policies are copied into `config/policies/`
2. Files are namespaced
3. Ownership is tracked
4. Priority is validated
5. Pack version + checksum recorded

Installed file naming pattern:
```
<pack_name>__<index>__<policy_slug>.yaml
```

Example:
```
starter__01__mask_pii.yaml
starter__02__cross_region.yaml
```

This guarantees:

- Safe removal
- Deterministic upgrades
- No accidental overwrite
- Namespace isolation

---

## 5. Installed Pack State Tracking

VaultKit records installed packs in:
```
.vkit/packs.yaml
```

Example:
```yaml
format_version: v1
installed_packs:
  starter:
    name: starter
    version: "1.0.0"
    layer: foundation
    installed_at: "2026-02-15T15:00:00Z"
    pack_checksum: "abc123..."
    files:
      - path: config/policies/starter__01__mask_pii.yaml
        policy_id: mask_pii
```

This enables:

- Drift detection
- Safe removal
- Controlled upgrade
- Bundle traceability

---

## 6. Drift Detection

Drift is detected when:
```
installed_version != shipped_version
```

CLI output:
```
⚠ starter (installed v1.0.0, available v1.1.0)
```

Drift detection does NOT auto-upgrade.

Upgrades require explicit command:
```bash
vkit policy pack upgrade
```

---

## 7. Pack Checksum Model

Each pack generates a SHA256 checksum over:

- `metadata.yaml`
- `policies/**/*.yaml`

Checksum ensures:

- Version integrity
- No silent modification
- Audit traceability
- Bundle reproducibility

Checksum stored in state file and embedded in bundle metadata.

---

## 8. Bundle Embedding

When compiling a policy bundle, installed packs are embedded:
```json
"installed_packs": [
  {
    "name": "starter",
    "version": "1.0.0"
  }
]
```

This ensures:

- Compliance traceability
- Governance reproducibility
- Deployment auditing
- Safe rollback validation

---

## 9. Upgrade Semantics

Upgrade flow:

1. Compare installed version to shipped version
2. If identical → no-op
3. If different:
   - Remove installed files
   - Reinstall from new version
   - Update state file
   - Preserve namespace isolation

Upgrade command:
```bash
vkit policy pack upgrade
```

Options:

| Flag | Behavior |
| --- | --- |
| `--dry-run` | Preview without writing |
| `--force` | Overwrite existing pack files |

VaultKit never overwrites non-pack files.

---

## 10. Dependency Enforcement

Before installation:
```ruby
ensure_dependencies!
```

If missing:
```
DependencyMissing: Pack requires dependencies not installed: starter
```

VaultKit does NOT auto-install dependencies to prevent surprise governance changes.

---

## 11. Priority Band Enforcement

If `priority_band` is defined:

All policies inside the pack must respect:
```
min <= policy.priority <= max
```

Violation raises:
```
InvalidPack: Policy priority outside allowed band
```

This prevents cross-pack priority collisions.

---

## 12. Safe Removal

When removing a pack:

1. Only tracked files are deleted
2. Non-pack files are untouched
3. State entry removed
4. Namespace preserved

Removal command:
```bash
vkit policy pack remove starter
```

---

## 13. Security Guarantees

Policy Packs guarantee:

- No silent overwrites
- No silent upgrades
- No cross-namespace file deletion
- Deterministic install order
- Layer-safe enforcement
- Bundle-embedded traceability
- Compliance auditability

---

## 14. Design Principles

The Pack System follows:

- Explicit over implicit
- Immutable versioned artifacts
- Human-controlled upgrades
- Deterministic governance
- Audit-first design

---

## 15. Example Pack Hierarchy
```
starter (foundation)
    ↓
ai_safety (domain)
    ↓
financial_compliance (domain)
    ↓
custom_company_rules (custom)
```

Each layer builds on the one below.

---

## 16. Future Enhancements

Planned improvements:

- Remote pack registry
- Signed pack verification
- Semantic version constraint resolution
- Auto-dependency resolution (optional)
- Pack marketplace

---

## Conclusion

Policy Packs are not templates.

They are **versioned governance modules**.

They allow VaultKit to scale from:

> "Write your own policies"

to

> "Install secure governance instantly."

They are a foundational component of VaultKit's long-term compliance and distribution strategy.