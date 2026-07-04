---
name: CreateRQC
description: Create RQC (Reagent Component) records in Veeva from a reagent inventory tracker sheet, matched by Kit Content Product Name + shipment/received date.
---

# CreateRQC — Create Reagent Component (RQC) records in Veeva from the tracker sheet

Use this skill to create one or more **RQC records** (Veeva object `reagent_qual_component__c`, label "Reagent - Component", ID prefix `RQC-`) using data from the **Reagent Inventory Tracker** Google Sheet. The tracker sheet is the source of truth for RQC field values.

> **Note:** This is a shared/public copy. Replace the placeholders below with your own values before use:
> - `<SHEET_FILE_ID>` — the Google Drive file ID of your reagent inventory tracker.
> - `<REFERENCE_RQC_ID>` / `<REFERENCE_RQS_ID>` — an example record for your vault (optional).

## Inputs to collect from the user
1. **Kit Content Product Name** — matches sheet column A (e.g. `SNP Backbone Probes 2 (SNP2)`). Match exactly, case-insensitive, trimmed.
2. **Shipment / Received Date** — matches sheet column I "Received Date" (e.g. `6/9/2026`). Compare as calendar dates, not strings. All sheet dates are **MM/DD/YYYY**.

If either input is missing, ask before proceeding.

## Source data
- Sheet: **Reagent Inventory Tracker**, fileId `<SHEET_FILE_ID>`, tab **"Materials Received"**.
- Read it with a Google Sheets CSV export tool. **Important:** some CSV tools return **base64-encoded** content. Decode it first, then parse. Cells contain embedded newlines, so use a real CSV parser and raise the field-size limit. Example:
  ```python
  import json, csv, io, sys, base64
  csv.field_size_limit(sys.maxsize)
  d = json.load(open(path_to_saved_result))
  raw = base64.b64decode(d['content']).decode('utf-8', errors='replace')
  rows = list(csv.reader(io.StringIO(raw)))   # rows[0] is the header
  ```
  (A plain "read" tool may truncate and merge columns — do NOT rely on it for exact field values.)
- The sheet may have year separator rows like "2022 Received Reagents" and blank cells — skip non-data rows.

### Sheet columns (0-indexed positions)
A(0) Kit Content Product Name · B(1) Box Part # · C(2) Box Name · D(3) Kit Content Part Number · E(4) Box Lot · F(5) Item Lot (Component Lot#) · G(6) Expiration Date · H(7) Quantity · I(8) Received Date · J(9) Part Serial Number (TC#) · K(10) Max Runs · L(11) Available Samples per Shipment · M(12) Method · N(13) RQS Record · O(14) RQC Record · P(15) RQ Record · … X(23) RQ Supervisor Approval Date

## Target object: `reagent_qual_component__c` (RQC-*)
Create via the Veeva "create record" tool with `object_name: "reagent_qual_component__c"`.

### Field mapping (all required fields must be present)
| RQC field (API) | Type | Source column |
|---|---|---|
| kit_component__c | String | A — Kit Content Product Name |
| box_part_number__c | String | B — Box Part # |
| box_name__c | String | C — Box Name |
| kit_part_number__c | String | D — Kit Content Part Number |
| box_lot_number__c | String | E — Box Lot |
| component_lot_number__c | String | F — Item Lot (Component Lot#) |
| expiration_date__c | Date | G — Expiration Date → convert MM/DD/YYYY to `YYYY-MM-DD` |
| quantity__c | Number | H — Quantity (numeric) |
| part_serial_number_tc__c | String | J — Part Serial Number (TC#) |
| shipment__c | Object (ID) | Resolve from N — RQS Record (see below) |

Do **not** set `date_shipment_received__c` — it is not editable and is auto-derived from the linked shipment. Do not set `name__v` — Veeva assigns the RQC-#### ID automatically. Do not set `status__v`/lifecycle fields — the object defaults them.

### Resolving shipment__c
`shipment__c` must be the Veeva **ID** of the shipment record, which is the RQS record named in column N. Resolve it:
```
SELECT id, name__v FROM reagent_shipment__c WHERE name__v = '<value from column N, e.g. RQS-000677>'
```
Use the returned `id` as `shipment__c`. If column N is blank or the RQS can't be found in Veeva, STOP and ask the user rather than creating a record without a valid shipment link.

### Date conversion
Sheet dates are always **MM/DD/YYYY** (e.g. `11/16/2027` = November 16, 2027). Convert Expiration Date to ISO `YYYY-MM-DD` (→ `2027-11-16`) for `expiration_date__c`. Watch for obviously malformed years; if a date looks wrong, flag it to the user instead of silently submitting.

## Procedure
1. Confirm both inputs (Product Name + Received Date).
2. Export/decode/parse the sheet and find **all rows** where column A matches the Product Name AND column I matches the Received Date (date-equality). This is a **one-RQC-per-matching-row** operation.
3. If **zero** rows match: report that and stop. (The exact product name and received date must match the sheet; confirm spelling/date/tab with the user if nothing is found.)
4. If matching rows already have an RQC Record value in column O, warn the user (they may already exist) and ask whether to proceed.
5. Verify all required fields are non-empty for each matched row. **If any required source cell is blank, STOP and ask** — do not submit partial records.
6. Build a preview table of the rows found and the exact field values that will be submitted (including the resolved shipment ID + RQS name, and expiration converted to YYYY-MM-DD). **Show it to the user and get explicit confirmation before creating anything** — this writes to the production quality vault.
7. On confirmation, for each matching row: resolve `shipment__c`, then create the record. Create records one at a time and collect the returned RQC IDs.
8. Report the created RQC IDs mapped to their component lots. Offer to write them back into column O of the sheet (ask first before editing the sheet).

## Guardrails
- This creates live records in a production quality vault. **Always preview and confirm before creating.** Never bulk-create silently.
- The tracker sheet is the source of truth — take RQC field values directly from the matched row.
- Verify all required fields are non-empty for each row before submitting; if any required source cell is blank, stop and ask.
- If multiple rows match, create one RQC each (do not merge), but confirm the full set with the user first.
