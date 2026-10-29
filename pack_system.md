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
  version: "2.0.0"
  layer: 10
  description: "Secure defaults: secrets denied, PII and high-sensitivity fields masked, approval for high-sensitivity data in production, unreviewed columns flagged."
  dependencies: []
  priority_band:
    min: 0
    max: 100
```

### Fields

| Field | Required | Description |
| --- | --- | --- |
| `name` | ✅ | Pack identifier |
| `version` | ✅ | Semantic version string |
| `layer` | ✅ | A number; lower layers are foundations for higher ones |
| `description` | ❌ | Human-readable summary |
| `dependencies` | ❌ | Required packs |
| `priority_band` | ❌ | Enforced priority range |

---

## 3. Pack Layers

A pack's `layer` is a number. Lower layers are foundations; higher layers build on
them.

| Layer | Shipped pack | Purpose |
| --- | --- | --- |
| 10 | `starter` | Core safety defaults |
| 30 | `ai_safety` | Controls for AI agents |
| 40 | `financial_compliance` | Financial data controls |

The CLI upgrades installed packs from the lowest layer up, and refuses a pack that
depends on a pack at a higher layer than its own. Layers don't decide which policy
wins at request time: the strongest action does (see
[Which policy wins](policy_guide.md#which-policy-wins)).

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
starter__01__deny_credentials.yaml
starter__02__mask_pii_by_default.yaml
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
    version: "2.0.0"
    layer: 10
    installed_at: "2026-10-29T15:00:00Z"
    pack_checksum: "abc123..."
    files:
      - path: config/policies/starter__02__mask_pii_by_default.yaml
        policy_id: mask_pii_by_default
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
⚠ starter (installed v1.0.0, available v2.0.0)
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
    "version": "2.0.0"
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

Priority orders policies inside a bundle; it doesn't decide which action wins. When
several policies match a request, the strongest action wins whatever its priority:
deny, then require approval, then mask, then allow. See the
[Policy Guide](policy_guide.md#which-policy-wins).

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

## 15. Shipped Packs (2.0.0)

| Pack | Depends on | What it does |
| --- | --- | --- |
| `starter` | — | Denies credentials. Masks personal data and high-sensitivity fields, also after approval. High-sensitivity data in production needs approval from `data_owner`. Columns rated `unknown` are allowed but flagged in the audit log. Other grants last 4 hours. |
| `ai_safety` | `starter` | Agents need approval from `security` for personal, financial or `unknown` data in production, and for high-sensitivity data in any environment. |
| `financial_compliance` | `starter` | Financial data in production needs approval from `finance_manager`, and grants with financial fields last 30 minutes. |

```
starter (layer 10)
 ├── ai_safety (layer 30)
 └── financial_compliance (layer 40)
custom company rules (your own files, outside any pack)
```

Before relying on the packs, create the approval groups they name (`data_owner`,
`security`, `finance_manager`), and review your registry labels: the packs act on
each field's sensitivity and category.

### Upgrading from 1.x

The 1.x packs had policies that could never fire: clearance and region rules that
matched values VaultKit didn't set, and financial rules that used an unsupported
`category` key and tags nothing produced. Run `vkit policy pack upgrade` (vkit 1.2.0
or later) to replace them.

`vkit policy bundle` now refuses any policy that can never match, so a project still
on the 1.x `financial_compliance` or `ai_safety` packs won't bundle until it's
upgraded. The error lists each policy and says why.

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