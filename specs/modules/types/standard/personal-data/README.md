# Standard Personal Data Types

Pre-classified `@type` definitions for common personal data patterns. These types ship with core
and are available to any `.strux` project without explicit import.

## Purpose

Each type carries built-in `FieldClassification` (category + sensitivity) per field. When used
with `PrivateData<T>` or a `private-data` rod, the rod automatically knows:
- Which fields to pseudonymize
- Which fields trigger encryption
- What categories to record in the Art. 30 manifest entry

## Standard Types

| Type | File | Composes |
|---|---|---|
| `PersonName` | [person-name.strux](person-name.strux) | — |
| `PersonalContact` | [personal-contact.strux](personal-contact.strux) | — |
| `PostalAddress` | [postal-address.strux](postal-address.strux) | — |
| `UserIdentity` | [user-identity.strux](user-identity.strux) | PersonName + PersonalContact |
| `EmployeeRecord` | [employee-record.strux](employee-record.strux) | UserIdentity |
| `FinancialAccount` | [financial-account.strux](financial-account.strux) | — |

## Sealed Types

Standard types are **sealed** — they cannot be redefined or extended in-place. Authors can
compose them into custom types:

```
// ✗ Not allowed — redefining a standard type:
@type PersonalContact { email: string, phone: string, twitter: string }  // compile error

// ✓ Allowed — composing a standard type:
@type ExtendedContact { base: PersonalContact, linkedin: Optional<string> }
```

Custom fields added to a composition require explicit `FieldClassification` in the `private-data`
rod's `cfg.fields` if they contain personal data.

## Classification Propagation

Field classifications propagate through nested types. When `UserIdentity` is used, the rod sees
all classifications from `PersonName` and `PersonalContact` as well as `UserIdentity`'s own
fields (`date_of_birth`, `national_id`).

See [specs/core/type-system.md §8](../../../core/type-system.md) for the full classification rules.
