---
name: idempiere-language-tool
description: Manage iDempiere UI languages and auto-populate translation (_trl) records with the Language Maintenance process, and resolve translation gaps that break Copy Tab Fields and other _trl operations
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-language-tool.md
  category: system-administration
  scope: idempiere
---

# iDempiere Language Tool

The purpose of this tool is to manage iDempiere UI languages and auto-populate their translation (`_trl`) records.

This is important because dictionary records (elements, fields, tabs, windows, menus) and several stock processes create a `_trl` row for every active *system* language. When a system language is missing translations, those operations fail — most visibly **Copy Tab Fields** (`AD_Tab_Copy`), which aborts with a `NOT NULL` violation on `ad_field_trl.name`.

## TOC

- [Language Model](#language-model)
- [Auto-Populate Translations](#auto-populate-translations)
- [Failure: Copy Tab Fields Aborts on a Missing Translation](#failure-copy-tab-fields-aborts-on-a-missing-translation)
- [Gotcha: Orphaned Org Reference Fails the COMMIT](#gotcha-orphaned-org-reference-fails-the-commit)
- [OEIG Languages](#oeig-languages)

## Language Model

- The **base language** (`IsBaseLanguage='Y'`, e.g. `en_US`) holds the untranslated values and needs no `_trl` rows.
- A language becomes translatable only when it is `IsActive='Y'` **and** `IsSystemLanguage='Y'`.
- `_trl` rows are created only for active, non-base system languages.

Find the languages that will receive translations:

```sql
SELECT ad_language_id, ad_language, name
FROM   ad_language
WHERE  isactive = 'Y' AND issystemlanguage = 'Y' AND isbaselanguage = 'N';
```

## Auto-Populate Translations

Run the **Language Maintenance** process (`AD_Language_Maintain`, process id `179`) with the `AD_Language` row as the record and a `MaintenanceMode` parameter:

| Mode | Value | Effect |
|------|-------|--------|
| Add Missing Translations | `A` | Insert the `_trl` rows a language is missing, across every translation table |
| Delete Translation | `D` | Remove the language's `_trl` rows and clear `IsSystemLanguage` |
| Re-Create Translation | `R` | Delete then Add |

Run mode `A` **after** creating dictionary records so new elements/fields/tabs gain their translations. Mode `A` requires the language to be active and a system language.

Invoke it via REST like any process (authenticate, grant process access, POST) — see the Programmatic Method in the idempiere-cache-reset tool; only the parameters differ:

```json
{"model-name": "ad_language", "record-id": <AD_Language_ID>, "MaintenanceMode": "A"}
```

The summary reads `Deleted=0 - Inserted=<n>`; re-running is idempotent (`Inserted=0` once complete).

## Failure: Copy Tab Fields Aborts on a Missing Translation

`AD_Tab_Copy` saves each copied field, which creates a `_trl` row per system language. If the copied columns' elements lack that language's translations, the new `ad_field_trl.name` is NULL and the whole copy rolls back:

```
null value in column "name" of relation "ad_field_trl" violates not-null constraint
```

Resolution: run Add Missing Translations (mode `A`) for the language **before** copying tabs. (See the idempiere-window tool for `AD_Tab_Copy` and the Zoom to Record pattern.)

## Gotcha: Orphaned Org Reference Fails the COMMIT

Add Missing Translations maintains every translation table in a single transaction. A base row with a dangling `AD_Org_ID` — for example config left behind by an incomplete client/org cleanup — fails a deferred foreign key (such as `adorg_aassetgrouptrl`) at COMMIT and rolls back the entire run, including the valid inserts. The error names the constraint, which identifies the base table; find and fix (reparent or delete) the orphaned rows, then re-run:

```sql
-- adorg_aassetgrouptrl -> base table a_asset_group; list rows whose org is gone
SELECT * FROM a_asset_group b
WHERE  NOT EXISTS (SELECT 1 FROM ad_org o WHERE o.ad_org_id = b.ad_org_id);
```

## OEIG Languages

Our seed ships `es_CO` (Spanish) as the only non-base system language, and we keep it — Spanish may be needed. After the go-live deploy creates the OEIG dictionary records, the `populate_language_translations` deploy script runs Language Maintenance (mode `A`) for every active non-base system language so tab copies and other `_trl` operations succeed. The GardenWorld client-11 cleanup must run first (it removes the org-11 config that would otherwise trip the COMMIT gotcha above).

Tags: #tool #idempiere #language #translation #system-admin
