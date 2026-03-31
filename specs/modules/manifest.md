# OpenStrux v0.6 — Manifest Specification

The manifest (`mf.strux.json`) is the compiled artifact that records the fully-resolved
configuration of a panel. It is the authoritative input for emitters and the primary
surface for certification, auditing, and compliance tooling.

---

## 1. Core Manifest Schema

```json
{
  "schemaVersion": "0.6",
  "version": "0.6.0",
  "panel": "<panel-name>",
  "contentHash": "<sha256-of-source>",
  "certificationScope": [...],
  "timestamp": "<ISO-8601>",
  "lockRef": "<lock-file-hash>",
  "dp": { ... },
  "access": { ... },
  "audit": { "entries": [...] },
  "rods": { ... },
  "privacyRecords": [...]
}
```

All fields except `privacyRecords` are present in all manifests. `privacyRecords` is included
only when one or more `private-data` rods are present; otherwise the field is omitted entirely.

---

## 2. `privacyRecords` Field

### When included

`privacyRecords` is an array included in the manifest when the compiled panel contains one or
more `private-data` rods. Each entry corresponds to one `private-data` rod instance.

When no `private-data` rods are present, `privacyRecords` is omitted (not `null`, not `[]`).

### Array entry schema

Each `privacyRecords` entry has the following structure:

```json
{
  "rodId": "<rod-name-in-panel>",
  "framework": "gdpr" | "gdpr.bdsg",
  "article30": { ... },
  "bdsgExtensions": { ... } | null,
  "expansionHash": "<sha256-of-expansion-config>",
  "fields": [...]
}
```

---

## 3. Art. 30 Record Structure

The `article30` object maps directly to GDPR Article 30 obligations (Records of Processing
Activities). All fields are derived at compile time from the panel source and its context:

```json
{
  "controller": "string — from @dp.controller",
  "controllerId": "string — from @dp.controller_id",
  "dpo": "string — from @dp.dpo",
  "record": "string | null — from @dp.record",
  "purpose": "string — from rod cfg.purpose",
  "lawfulBasis": "string — from cfg.framework.lawful_basis",
  "dataSubjectCategories": ["string", "..."] ,
  "personalDataCategories": ["string", "..."],
  "specialCategories": ["string", "..."],
  "recipients": [
    {
      "type": "string — DataTarget type path",
      "db_name": "string | null",
      "topic": "string | null"
    }
  ],
  "retention": {
    "duration": "string",
    "basis": "string",
    "reviewCycle": "string | null"
  },
  "technicalMeasures": ["pseudonymization", "encryption"],
  "dpiaRef": "string | null — from cfg.framework.dpia_ref",
  "crossBorderTransfer": {
    "mechanism": "string",
    "destinationCountries": ["string", "..."]
  } | null
}
```

### Field derivation rules

| Field | Derived from | Notes |
|---|---|---|
| `controller` | `@dp.controller` (panel or context) | Required; compile warning if missing |
| `controllerId` | `@dp.controller_id` | Optional |
| `dpo` | `@dp.dpo` | Required for BDSG (compile error if missing) |
| `record` | `@dp.record` | Optional |
| `purpose` | `cfg.purpose` | Required for GDPR (compile error if missing) |
| `lawfulBasis` | `cfg.framework.lawful_basis` | Required for GDPR |
| `dataSubjectCategories` | `cfg.framework.data_subject_categories` | Sorted array |
| `personalDataCategories` | Deduplicated `cfg.fields[].category` values | Sorted array |
| `specialCategories` | Fields where `sensitivity: special_category` or `highly_sensitive` | Sorted array of category names |
| `recipients` | Downstream `write-data` targets in the panel | Derived from snap graph |
| `retention` | `cfg.retention` | `reviewCycle` is null if not specified |
| `technicalMeasures` | Derived from expansion: always includes `pseudonymization`; includes `encryption` when `encryption_required: true` | Sorted array |
| `dpiaRef` | `cfg.framework.dpia_ref` | `null` if not specified |
| `crossBorderTransfer` | `cfg.framework.cross_border_transfer` | `null` if not specified |

---

## 4. BDSG Extension Fields

When `framework: gdpr.bdsg`, the `bdsgExtensions` object is included alongside `article30`:

```json
{
  "bdsgSection26": true | false,
  "employeeCategory": "applicant" | "employee" | "former_employee" | "contractor" | "trainee" | null,
  "betriebsratConsent": "string | null",
  "dataProtectionOfficer": "string — from @dp.dpo"
}
```

`bdsgExtensions` is `null` for `framework: gdpr` (not BDSG). For `framework: gdpr.bdsg`, it is
always present.

---

## 5. `fields` Array in Privacy Record

Each `privacyRecords` entry includes a `fields` array listing all classified fields:

```json
[
  {
    "field": "string — field name",
    "category": "DataCategory value",
    "sensitivity": "Sensitivity value",
    "bdsgElevated": true | false
  }
]
```

`bdsgElevated` is `true` when the BDSG §26 employee data rule elevated the field's effective
sensitivity above its declared sensitivity. Always `false` for `framework: gdpr`.

Fields are sorted alphabetically by `field` name for determinism.

---

## 6. Determinism Requirements

Privacy records MUST be stable across compilations given the same source and lock file:

1. **Field order:** All object fields appear in alphabetical key order.
2. **Array order:** All array values are sorted (strings alphabetically, objects by first field).
3. **Null vs absent:** Fields that are absent in the source appear as `null` in the manifest
   (not omitted), except for `privacyRecords` itself which is omitted entirely when not needed.
4. **Timestamps:** The `timestamp` field at the top-level manifest is NOT included in privacy
   record hashes — only deterministic fields are hashed.
5. **expansionHash:** Computed as `sha256(rod_type + ":" + sorted_cfg_json)` where
   `sorted_cfg_json` is the JSON serialization of the privacy rod's config with all keys sorted.

---

## 7. Multiple `private-data` Rods

A panel may contain multiple `private-data` rods (e.g., one for intake, one for export).
Each produces a separate entry in `privacyRecords`. Entries are ordered by rod declaration
order in the panel source.

```json
{
  "privacyRecords": [
    { "rodId": "pd_intake", "framework": "gdpr", "article30": { "purpose": "intake", ... }, ... },
    { "rodId": "pd_export", "framework": "gdpr", "article30": { "purpose": "export", ... }, ... }
  ]
}
```

---

## 8. Querying Privacy Records

The `privacyRecords` field is designed for structural querying by compliance tooling:

```bash
# List all processing purposes in a compiled panel
jq '.privacyRecords[].article30.purpose' mf.strux.json

# Check if any rod processes special category data
jq '.privacyRecords[].article30.specialCategories | select(length > 0)' mf.strux.json

# Get all retention periods
jq '.privacyRecords[].article30.retention' mf.strux.json
```

This is the structural certification output that benchmarks can query — no heuristic scanning
needed.
