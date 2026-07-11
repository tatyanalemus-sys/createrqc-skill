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
Search Drive for "Exome+ Reagent Inventory Tracker" (file ID: `1_FBszw8VM22zqEOBOqea-4TzxqiHIhKmwS-wmOue5PY`).

### 2. Get the RQS internal Veeva ID
```vql
SELECT id, name__v, date_shipment_received__c FROM reagent_shipment__c WHERE name__v = '<RQS-XXXXXX>'
```
Use the returned `id` as the `shipment__c` value on every RQC.

### 3. Export sheet and filter rows by RQS name
Use `sheets_export_csv` on the file, decode the base64 content, and filter for rows where the "RQS Record" column (column N) matches the given RQS name. These are all the reagent lines to create RQCs for.

The file is large — save the tool result to disk and read it with bash/python.

### 4. Create RQC records
Call `veeva_create_record` with `object_name: reagent_qual_component__c` for each matching row. Run multiple creates in parallel (2–3 at a time).

### 5. Verify
After creation, confirm all records exist:
```vql
SELECT id, name__v, box_name__c, component_lot_number__c, expiration_date__c 
FROM reagent_qual_component__c 
WHERE shipment__c = '<shipment_internal_id>'
ORDER BY name__v ASC
```

### 6. Report back
Return a table of newly created RQC numbers (RQC-XXXXXX) mapped to their plate/reagent names, and remind the user to update the RQC Record column (column O) in the Google Sheet.

## Notes
- Expiration dates in the sheet are M/D/YYYY — convert to YYYY-MM-DD for Veeva.
- `name__v` (the RQC-* number) is auto-assigned by Veeva; do not set it manually.
- The sheet also has separate RQS and RQ columns — those are different objects and not created by this skill.
- Box lot number is typically the same for all reagents in a single shipment.
