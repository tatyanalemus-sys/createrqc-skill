# Create Veeva RQC Records from Exome+ Reagent Tracker

## When to use this skill
Trigger when the user says things like:
- "Create RQCs for RQS-000677"
- "Add RQC records for [shipment name]"
- "Log reagents in Veeva for this shipment"
- "Create RQCs for every reagent on RQS-XXXXXX"

The simplest invocation is just: **"Create RQCs for RQS-XXXXXX"** — Claude will find all matching rows in the tracker automatically and create one RQC per line.

## Key Veeva objects
- **RQC records** → `reagent_qual_component__c` (Reagent - Component)
- **RQS records** → `reagent_shipment__c` (Reagent - Shipment) — each RQC must link to a parent shipment via `shipment__c` (internal Veeva ID, e.g. `V5K0000000D9001`)
- **RQ records** → `reagent_qualification__c` (Reagent - Qualification) — separate object, not created by this skill

## Required fields for reagent_qual_component__c
| Field | Label | Source in sheet |
|-------|-------|----------------|
| `kit_component__c` | Kit Content Product Name | Column A |
| `box_part_number__c` | Box Part Number | Column B |
| `box_name__c` | Box Name | Column C |
| `kit_part_number__c` | Kit Content Part Number | Column D |
| `box_lot_number__c` | Box Lot Number | Column E |
| `component_lot_number__c` | Component Lot Number | Column F |
| `expiration_date__c` | Expiration Date | Column G (convert to YYYY-MM-DD) |
| `quantity__c` | Quantity | Column H |
| `shipment__c` | Shipment (internal ID) | Look up from RQS name |
| `part_serial_number_tc__c` | Part Serial Number (TC#) | Column J |

## Step-by-step workflow

### 1. Find the Google Sheet
Search Drive for "Exome+ Reagent Inventory Tracker" (file ID: `1_FBszw8VM22zqEOBOqea-4TzxqiHIhKmwS-wmOue5PY`), tab "Exome_Materials Received".

### 2. Get the RQS internal Veeva ID
```vql
SELECT id, name__v, date_shipment_received__c FROM reagent_shipment__c WHERE name__v = '<RQS-XXXXXX>'
```
Use the returned `id` as the `shipment__c` value on every RQC. If the RQS can't be found, STOP and ask the user — never create an RQC without a valid shipment link.

### 3. Export sheet and filter rows by RQS name
Use `sheets_export_csv` on the file, decode the base64 content, and filter for rows where the "RQS Record" column (column N) matches the given RQS name. These are all the reagent lines to create RQCs for.

The file is large — the tool result is saved to disk. Locate it from bash with `find /sessions/*/mnt/.claude -name "mcp-Google_Workspace-sheets_export_csv*"` (the raw macOS temp path is not reachable), then decode/parse with python (`base64.b64decode`, `csv` with a raised field-size limit). Skip year-separator rows like "2022 Received Reagents".

If **zero** rows match, report that and stop. Verify all required source cells are non-empty for each matched row; if any required cell is blank, STOP and ask rather than submitting a partial record.

### 4. Check Veeva for existing RQCs BEFORE creating (critical)
**Do not trust the tracker's RQC Record column (column O) to decide whether an RQC already exists — it is often left blank even when the record exists in the vault.** Always confirm against Veeva first:
```vql
SELECT id, name__v, kit_component__c, box_lot_number__c, component_lot_number__c, expiration_date__c, quantity__c, part_serial_number_tc__c, shipment__c
FROM reagent_qual_component__c WHERE kit_component__c = '<product name>'
```
(Or filter by `part_serial_number_tc__c = '<TC#>'`.) If a record already exists with the same **TC# (`part_serial_number_tc__c`)** — or the same component lot + shipment when TC# is absent — **STOP and report the existing RQC-XXXXXX instead of creating a duplicate.** Only proceed for rows with no Veeva match.

> Example: on RQS-000688 the tracker's column O was blank, but RQC-002138 already existed for TC0001050-ETOH linked to that shipment. Creating from the sheet alone would have made a duplicate.

### 5. Preview and confirm
Build a preview table of every row to be created and the exact field values (including the resolved shipment ID + RQS name, and expiration converted to YYYY-MM-DD). **Show it to the user and get explicit confirmation before creating anything** — this writes to the production Veeva QualityOne vault.

### 6. Create RQC records
On confirmation, call `veeva_create_record` with `object_name: reagent_qual_component__c` for each non-duplicate row. Create records **one at a time** and collect the returned RQC IDs. Do not set `name__v` (auto-assigned) or `date_shipment_received__c` (auto-derived, not editable).

### 7. Verify
After creation, confirm all records exist:
```vql
SELECT id, name__v, box_name__c, component_lot_number__c, expiration_date__c
FROM reagent_qual_component__c
WHERE shipment__c = '<shipment_internal_id>'
ORDER BY name__v ASC
```

### 8. Report back
Return a table of newly created RQC numbers (RQC-XXXXXX) mapped to their reagent names/component lots. Offer to write them back into the RQC Record column (column O) of the Google Sheet — ask before editing the sheet.

## Notes
- **Always preview and confirm before creating; never bulk-create silently.** This writes to production.
- **A blank column O does not mean the RQC is absent** — always confirm against Veeva by TC# / component lot + shipment.
- Expiration dates in the sheet are M/D/YYYY — convert to YYYY-MM-DD for Veeva. Flag obviously malformed years (e.g. 2208) instead of submitting them.
- `name__v` (the RQC-* number) is auto-assigned by Veeva; do not set it manually.
- `veeva_get_record` does not accept the `RQC-` prefix — look RQCs up via `veeva_vql_query` on `reagent_qual_component__c`.
- The sheet also has separate RQS and RQ columns — those are different objects and not created by this skill.
- Box lot number is typically the same for all reagents in a single shipment.
- Reference record for expected shape: RQC-002123 (`reagent_qual_component__c`), linked to shipment RQS-000677.