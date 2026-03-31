# private-data rod — GDPR Specialization

**Applies to:** `private-data { framework: gdpr }` and `private-data { framework: gdpr.bdsg }`
(BDSG inherits all rules below and adds further constraints — see [private-data-bdsg.md](private-data-bdsg.md))

---

## Art. 5 — Principles Enforcement (Compile-Time)

GDPR Article 5 requires that personal data be processed according to defined principles.
The `private-data` rod enforces the following at compile time:

### Purpose limitation (Art. 5(1)(b))

`cfg.purpose` is **required**. A `private-data` rod without `purpose` is a compile error:

```
E_GDPR_PURPOSE_REQUIRED: cfg.purpose is required for GDPR framework (Art. 5(1)(b)).
  Set a human-readable purpose string on the private-data rod.
```

### Data minimization (Art. 5(1)(c))

The expanded `pseudonymize` rod masks all fields classified as `identifying` or `quasi_identifying`.
An optional `arg.predicate` further restricts which records are processed before pseudonymization.
Authors MUST NOT pass fields beyond what is necessary for the declared purpose.

### Storage limitation (Art. 5(1)(e))

`cfg.retention` is **required**. A `private-data` rod without `retention` (and no inherited
`retention` from `@privacy` in `strux.context`) is a compile error:

```
E_GDPR_RETENTION_REQUIRED: cfg.retention is required for GDPR framework (Art. 5(1)(e)).
  Specify retention duration and basis, or declare @privacy { retention: ... } in strux.context.
```

---

## Art. 25 — Data Protection by Design and by Default

When `framework: gdpr` is used, the expanded `pseudonymize` rod automatically masks fields
based on their classification — no explicit listing required.

| Field classification | GDPR base default |
|---|---|
| `identifying` | Always pseudonymized |
| `quasi_identifying` | Not pseudonymized by default (explicit opt-in only) |
| `financial` | Always pseudonymized |
| `health`, `biometric`, `genetic`, `political`, `religious`, `trade_union`, `sexual_orientation`, `criminal`, `sensitive_special` | Always pseudonymized |

**Note:** Under `framework: gdpr.bdsg`, `quasi_identifying` fields are also pseudonymized by
default (stricter BDSG protection). See [private-data-bdsg.md](private-data-bdsg.md).

---

## Art. 6 — Lawful Basis Validation

`cfg.framework.lawful_basis` (in `GdprBaseConfig`) is required and validated:

| Basis | Usage note |
|---|---|
| `consent` | Compiler checks that no `arg.predicate` contradicts scope of consent |
| `contract` | Accepted without additional compile-time checks |
| `legal_obligation` | Accepted; DPIA not required |
| `vital_interests` | Accepted; rare — compiler emits informational note |
| `public_task` | Accepted without additional compile-time checks |
| `legitimate_interest` | Accepted; compiler emits a **warning** if no `dpia_ref` is provided |

```
W_GDPR_LI_DPIA_RECOMMENDED: processing under legitimate_interest without dpia_ref (Art. 35).
  Consider adding dpia_ref to document the legitimate interest assessment.
```

---

## Art. 9 — Special Category Data Restrictions

When any field has `sensitivity: special_category` or `sensitivity: highly_sensitive`, the
compiler enforces additional rules:

1. **Encryption forced:** `encryption_required` is set to `true` regardless of explicit config.
2. **Lawful basis restricted:** Only `consent`, `legal_obligation`, and `vital_interests` are
   valid bases. Other bases produce:
   ```
   E_GDPR_INVALID_BASIS_SPECIAL_CATEGORY: lawful_basis <value> is not permitted for
     special category data under Art. 9(2). Use consent, legal_obligation, or vital_interests.
   ```
3. **Art. 30 record includes special categories:** The manifest entry lists these fields
   separately under `specialCategories`.

---

## Art. 30 — Records of Processing Activities (Manifest)

When a `private-data { framework: gdpr }` rod is compiled, the manifest includes an Art. 30
record entry. The compiler derives the record from rod config and surrounding panel declarations.

### Record derivation

| Art. 30 field | Source |
|---|---|
| `controller` | `@dp.controller` (panel or inherited from context) |
| `controllerId` | `@dp.controller_id` |
| `dpo` | `@dp.dpo` |
| `record` | `@dp.record` |
| `purpose` | `cfg.purpose` |
| `lawfulBasis` | `cfg.framework.lawful_basis` |
| `dataSubjectCategories` | `cfg.framework.data_subject_categories` |
| `personalDataCategories` | Derived from `cfg.fields[].category` (deduplicated) |
| `specialCategories` | Derived from fields where `sensitivity: special_category` or `highly_sensitive` |
| `recipients` | Derived from downstream `write-data` targets in the panel |
| `retention` | `cfg.retention` |
| `technicalMeasures` | `["pseudonymization"]` + `["encryption"]` if `encryption_required: true` |
| `dpiaRef` | `cfg.framework.dpia_ref` (null if not provided) |
| `crossBorderTransfer` | `cfg.framework.cross_border_transfer` (null if not provided) |

### Multiple records

A panel with multiple `private-data` rods produces one Art. 30 entry per rod instance.
Each entry corresponds to the specific rod's config.

### Determinism

Privacy records MUST be stable across compilations with the same source and lock file.
Field order within each record is fixed (alphabetical). Array values are sorted.

---

## Golden Conformance Fixture

See [`conformance/golden/private-data-gdpr-manifest.json`](../../../../conformance/golden/private-data-gdpr-manifest.json)
for the expected manifest output from [conformance/valid/private-data-gdpr.strux](../../../../conformance/valid/private-data-gdpr.strux).
