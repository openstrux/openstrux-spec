# private-data rod — BDSG Specialization

**Applies to:** `private-data { framework: gdpr.bdsg }` only.

BDSG (German Federal Data Protection Act, *Bundesdatenschutzgesetz*) inherits **all** GDPR
requirements from [private-data-gdpr.md](private-data-gdpr.md) and adds further constraints.
BDSG never relaxes GDPR requirements — it only tightens them.

---

## §26 — Employee Data Rules

When `framework: gdpr.bdsg` is used with `employee_data: true`, the compiler applies elevated
protection rules per BDSG Section 26:

### Sensitivity elevation

All `identifying` and `quasi_identifying` fields are treated as `sensitivity: special_category`
for the purposes of pseudonymization and encryption, regardless of their declared sensitivity:

```
// Declared:
{ field: "salary", category: financial, sensitivity: standard }

// Compiler treats as:
{ field: "salary", category: financial, sensitivity: special_category }
// → encryption_required forced to true
// → included in Art. 9 special category list in Art. 30 record
```

### Encryption default

Under `framework: gdpr.bdsg`, `encryption_required` is **always** `true`. Explicit
`encryption_required: false` is a compile error:

```
E_BDSG_ENCRYPTION_REQUIRED: encryption_required cannot be false under framework: gdpr.bdsg.
  BDSG §26 requires encryption for employee personal data.
```

### employee_category required

When `employee_data: true`, `cfg.framework.employee_category` is **required**:

```
E_BDSG_EMPLOYEE_CATEGORY_REQUIRED: employee_category is required when employee_data is true
  (BDSG §26). Valid values: applicant, employee, former_employee, contractor, trainee.
```

---

## Betriebsrat Consent Tracking

The `betriebsrat_consent` field records the works council consent reference. It is optional
but required when the processing purpose involves monitoring, performance evaluation, or
behavioral analysis (per BetrVG §87 — Mitbestimmungsrecht):

```
W_BDSG_BETRIEBSRAT_CONSENT_RECOMMENDED: processing purpose "<value>" may involve
  employee monitoring or performance evaluation. Betriebsrat consent reference is
  recommended under BDSG/BetrVG §87. Add betriebsrat_consent to the framework config.
```

When `betriebsrat_consent` is provided, the value is included in the manifest privacy record
as `betriebsratConsent`.

**Purpose patterns that trigger the warning:**
- Purpose contains "monitoring", "performance", "evaluation", "tracking", "surveillance",
  "productivity", or "behavior"

---

## §26 Stricter Pseudonymization Defaults

Under `framework: gdpr.bdsg`, the expanded `pseudonymize` rod uses stronger defaults:

| Setting | GDPR base | GDPR + BDSG |
|---|---|---|
| Default algorithm | `sha256` (one-way) | `sha256_hmac` (keyed HMAC) |
| Key reference required | No | Yes — compile error if missing |
| `quasi_identifying` masked | No (opt-in) | Yes (always) |

**Key reference requirement:**

```
E_BDSG_PSEUDONYMIZE_KEY_REQUIRED: sha256_hmac pseudonymization requires a key reference.
  Add pseudonymize_key_ref to the rod config or to strux.context @privacy block.
  The key reference must point to a secret (e.g., secret_ref { provider: vault, path: "..." }).
```

The key reference is derived from the rod config or context. It is NOT stored in the manifest —
only the key reference path is recorded.

---

## BDSG Manifest Extensions

The manifest Art. 30 record for `framework: gdpr.bdsg` includes all GDPR Art. 30 fields plus
the following BDSG-specific extensions:

| Field | Source | Required |
|---|---|---|
| `bdsgSection26` | `true` when `employee_data: true` | When employee_data |
| `employeeCategory` | `cfg.framework.employee_category` | When employee_data |
| `betriebsratConsent` | `cfg.framework.betriebsrat_consent` | When provided |
| `dataProtectionOfficer` | `@dp.dpo` | Always (BDSG §38 — mandatory DPO) |

**DPO requirement (BDSG §38):** When `framework: gdpr.bdsg` is used, a `@dp.dpo` field is
**required**. Missing DPO is a compile error:

```
E_BDSG_DPO_REQUIRED: @dp.dpo is required when framework: gdpr.bdsg is used.
  BDSG §38 mandates a Data Protection Officer when processing employee data.
```

---

## Relationship to GDPR Requirements

All Art. 5, 6, 9, 25, and 30 requirements from [private-data-gdpr.md](private-data-gdpr.md)
apply **without exception** to `framework: gdpr.bdsg`. Key interactions:

- **Art. 5 purpose + retention:** Both still required under BDSG.
- **Art. 9 special categories:** BDSG §26 expands the scope of what counts as special category
  for employee data — the Art. 9 basis restrictions apply to the elevated fields.
- **Art. 30 records:** BDSG extends the record but does not replace it. The manifest entry
  contains the full Art. 30 record plus BDSG extensions in the same object.
- **Art. 25 by-design defaults:** BDSG strengthens these — all quasi-identifying fields are
  included, not just identifying ones.

---

## Golden Conformance Fixture

See [`conformance/golden/private-data-bdsg-manifest.json`](../../../../conformance/golden/private-data-bdsg-manifest.json)
for the expected manifest output from [conformance/valid/private-data-bdsg.strux](../../../../conformance/valid/private-data-bdsg.strux).
