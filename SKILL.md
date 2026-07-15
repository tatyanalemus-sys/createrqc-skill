# Create Veeva RQC Records from a Reagent Inventory Tracker

Create one or more **RQC records** (Veeva object `reagent_qual_component__c`, "Reagent - Component", ID prefix `RQC-`) from a **reagent inventory tracker** Google Sheet. There are **two source sheets** (see below); the tracker is the source of truth for RQC field values.

## When to use this skill
Trigger when the user says things like:
- "Create RQCs for RQS-000677"
- "Add RQC records for [shipment name]"
- "Log reagents in Veeva for this shipment"
- "Create RQCs for every reagent on RQS-XXXXXX"

The simplest invocation is just: **"Create RQCs for RQS-XXXXXX"** — Claude finds all matching rows in the tracker and creates one RQC per line.

## Key Veeva objects
- **RQC records** → `reagent_qual_component__c` (Reagent - Component)
- **RQS records** → `reagent_shipment__c` (Reagent - Shipment) — each RQC must link to a parent shipment via `shipment__c` (internal Veeva ID, e.g. `V5K0000000D9001`)
- **RQ records** → `reagent_qualification__c` (Reagent - Qualification) — separate object, not created by this skill

## Source sheets (two options)
Reagent lines live in one of two trackers. If the user names one, use it. If not, ask which — or, when matching by RQS #, product name, or TC#, search both and use whichever contains the matching row (and tell the user which sheet it came from).

### Sheet 1 — Exome+ Reagent Inventory Tracker
- fileId `1_FBszw8VM22zqEOBOqea-4TzxqiHIhKmwS-wmOue5PY`, tab **"Exome_Materials Received"**. 24 columns.
- **0-indexed columns:** A(0) Kit Content Product Name · B(1) Box Part # · C(2) Box Name · D(3) Kit Content Part Number · E(4) Box Lot · F(5) Item Lot (Component Lot#) · G(6) Expiration Date · H(7) Quantity · I(8) Received Date · J(9) Part Serial Number (TC#) · K(10) Max Runs · L(11) Available Samples per Shipment · M(12) Method · N(13) RQS Record · O(14) RQC Record · P(15) RQ Record · … X(23) RQ Supervisor Approval Date

### Sheet 2 — Confirmatory Reagent Inventory Tracking
- fileId `1e0O0O4np4itZdNvFunvX-yfdq01p9OkyKWon8Aa-K9g`, tab **"WGS_Materials Received"**. 16 columns. **Different layout — do NOT reuse Sheet 1 column positions.**
- **0-indexed columns:** A(0) Reagent · B(1) Box Part # · C(2) Kit Content Number · D(3) Description · E(4) Box Lot · F(5) Item Lot · G(6) Expiration Date · H(7) Count · I(8) Received Date · J(9) RQS Record · K(10) RQC Record · L(11) RQ Record · M(12) Veeva QC Status · N(13) Date Passed · O(14) In Calender · P(15) Comment
- **This sheet has NO Part Serial Number (TC#) column.** Do not set `part_serial_number_tc__c` from this sheet unless the value appears elsewhere (e.g. inside the Item Lot string); if the user needs a TC#, ask.

## Inputs to collect from the user
Rows can be matched two ways:
1. **Kit Content Product Name / Reagent** (Sheet 1 col A, Sheet 2 col A) **+ Received Date** (both: col I). Match name exactly (case-insensitive, trimmed); compare dates as calendar dates, not strings. All sheet dates are **MM/DD/YYYY**.
2. **RQS record** (e.g. `RQS-000688`) — Sheet 1 col N, Sheet 2 col J. If the user gives an RQS number, match rows directly on the RQS column.

If inputs are missing/ambiguous, or you don't know which sheet, ask before proceeding.

## Required fields — mapping depends on which sheet
| RQC field (API) | Type | Sheet 1 (Exome+) col | Sheet 2 (Confirmatory) col |
|---|---|---|---|
| `kit_component__c` | String | A — Kit Content Product Name | A — Reagent |
| `box_part_number__c` | String | B — Box Part # | B — Box Part # |
| `box_name__c` | String | C — Box Name | D — Description |
| `kit_part_number__c` | String | D — Kit Content Part Number | C — Kit Content Number |
| `box_lot_number__c` | String | E — Box Lot | E — Box Lot |
| `component_lot_number__c` | String | F — Item Lot (Component Lot#) | F — Item Lot |
| `expiration_date__c` | Date | G — Expiration Date → `YYYY-MM-DD` | G — Expiration Date → `YYYY-MM-DD` |
| `quantity__c` | Number | H — Quantity | H — Count |
| `part_serial_number_tc__c` | String | J — Part Serial Number (TC#) | *(not in this sheet)* |
| `shipment__c` | Object (ID) | Resolve from N — RQS Record | Resolve from J — RQS Record |

Do **not** set `date_shipment_received__c` (not editable; auto-derived from the linked shipment), `name__v` (Veeva assigns the RQC-#### ID), or `status__v`/lifecycle fields (defaulted).

## Reading a sheet
Read with `sheets_export_csv`. The `content` field is **base64-encoded**, and the result is large — it is saved to a file rather than returned inline. Decode the base64, then parse. Cells contain embedded newlines, so use a real CSV parser with a raised field-size limit.
- **Path gotcha:** the saved tool-result file is under the `.claude/projects/.../tool-results/` tree, reachable from bash (not the raw macOS temp path). Locate with `find /sessions/*/mnt/.claude -name "mcp-Google_Workspace-sheets_export_csv*"`. Example:
  ```python
  import json, csv, io, sys, base64
  csv.field_size_limit(sys.maxsize)
  d = json.load(open(path_to_saved_result))
  raw = base64.b64decode(d['content']).decode('utf-8', errors='replace')
  rows = list(csv.reader(io.StringIO(raw)))   # rows[0] is the header
  ```
  (`sheets_read` truncates and merges columns — do NOT rely on it for exact field values.)
- Both sheets have year-separator / non-data rows and blank cells — skip them.

## Step-by-step workflow

### 1. Confirm inputs and sheet
Confirm the match inputs (Product Name/Reagent + Received Date, OR an RQS number) **and which sheet** (or search both).

### 2. Get the RQS internal Veeva ID
```vql
SELECT id, name__v FROM reagent_shipment__c WHERE name__v = '<RQS-XXXXXX>'
```
Use the returned `id` as `shipment__c` on every RQC. If the RQS cell is blank or the RQS can't be found, STOP and ask — never create an RQC without a valid shipment link.

### 3. Export sheet and filter rows
Export/decode/parse the correct sheet and find **all matching rows** using that sheet's column positions (RQS column, or Reagent + Received Date). One-RQC-per-matching-row. If **zero** rows match, report and stop; confirm spelling/date/sheet with the user. Verify required source cells are non-empty; if any required cell is blank, STOP and ask rather than submitting a partial record.

### 4. Check Veeva for existing RQCs BEFORE creating (critical)
**Do not trust the tracker's RQC Record column (Sheet 1 col O, Sheet 2 col K) to decide whether an RQC already exists — it is often left blank even when the record exists in the vault.** Always confirm against Veeva first:
```vql
SELECT id, name__v, kit_component__c, box_lot_number__c, component_lot_number__c, expiration_date__c, quantity__c, part_serial_number_tc__c, shipment__c
FROM reagent_qual_component__c WHERE kit_component__c = '<product name/reagent>'
```
(Or filter by `part_serial_number_tc__c = '<TC#>'` when available.) If a record already exists with the same **TC#** — or same component lot + shipment when TC# is absent — **STOP and report the existing RQC-XXXXXX instead of creating a duplicate.** Only proceed for rows with no Veeva match.

> Example: on RQS-000688 the tracker's RQC Record column was blank, but RQC-002138 already existed for TC0001050-ETOH linked to that shipment. Creating from the sheet alone would have made a duplicate.

### 5. Consistency check against the last 20 RQCs
Before building the preview, pull the most recently created RQCs and compare how the tracker data maps onto them, to catch drift in how fields are populated:
```vql
SELECT id, name__v, kit_component__c, box_part_number__c, box_name__c, kit_part_number__c, box_lot_number__c, component_lot_number__c, expiration_date__c, quantity__c, part_serial_number_tc__c, shipment__c
FROM reagent_qual_component__c ORDER BY created_date__v DESC
```
Take the first 20 rows (`limit: 20`). **Flag any discrepancy in how data is pulled from the tracker**, e.g.: `component_lot_number__c` format (lot alone `250743` vs lot+TC `210854 (TC0001027-ETOH)`); `box_name__c`/Description wording; whether `part_serial_number_tc__c` is its own field vs embedded in the lot; a field left blank on the candidate that is consistently populated on recent records for the same product (or vice-versa); part-number/product-name spelling mismatches. Summarize mismatches and **ask the user how to proceed before continuing** — do not silently normalize. If consistent, note that and continue. (The two sheets use different conventions — Confirmatory has no TC# field, so a missing TC# there is expected, not a discrepancy.)

### 6. Preview and confirm
Build a preview table of every row to be created and the exact field values (resolved shipment ID + RQS name, expiration as YYYY-MM-DD, and which sheet the data came from), plus any consistency findings from step 5. **Show it to the user and get explicit confirmation before creating anything** — this writes to the production Veeva QualityOne vault.

### 7. Create RQC records
On confirmation, call `veeva_create_record` with `object_name: reagent_qual_component__c` for each non-duplicate row. Create records **one at a time** and collect the returned RQC IDs.

### 8. Verify
```vql
SELECT id, name__v, box_name__c, component_lot_number__c, expiration_date__c
FROM reagent_qual_component__c
WHERE shipment__c = '<shipment_internal_id>'
ORDER BY name__v ASC
```

### 9. Report back
Return a table of newly created RQC numbers (RQC-XXXXXX) mapped to their reagent names/component lots. Offer to write them back into the RQC Record column of the correct sheet (Sheet 1 col O / Sheet 2 col K) — ask before editing the sheet.

## Notes / guardrails
- **Always preview and confirm before creating; never bulk-create silently.** This writes to production.
- **Use the correct column mapping for the sheet you're reading** — the two trackers have different layouts. Confirmatory (WGS) has no TC# column.
- **A blank RQC Record column does not mean the RQC is absent** — always confirm against Veeva by TC# / component lot + shipment.
- **Always run the last-20-RQC consistency check** and surface drift before creating; ask, don't normalize.
- Expiration dates in the sheet are M/D/YYYY — convert to YYYY-MM-DD for Veeva. Flag obviously malformed years (e.g. 2208) instead of submitting them.
- `name__v` (the RQC-* number) is auto-assigned by Veeva; do not set it manually.
- `veeva_get_record` does not accept the `RQC-` prefix — look RQCs up via `veeva_vql_query` on `reagent_qual_component__c`.
- Box lot number is typically the same for all reagents in a single shipment.
- Reference record for expected shape: RQC-002123 (`reagent_qual_component__c`), linked to shipment RQS-000677.