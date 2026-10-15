---
name: create-rqc-records
description: Create reagent qualification component (RQC) records in Veeva from the Helix reagent inventory trackers. Use for requests to create RQCs, log reagents, qualify an RQS shipment, or reconcile tracker rows with Veeva; do not use for manifest-only requests.
metadata:
  short-description: Create and verify Veeva RQC records
---

# Create RQC records

This is the standalone RQC-creation skill adapted from the Claude workflow in `quintinromano-code/reagent-qualification-skills`.

## Codex tools

- Veeva reads: `mcp__codex_apps__veeva_veeva_get_record` for a known display ID and `mcp__codex_apps__veeva_veeva_vql_query` for read-only SELECT queries.
- Veeva writes: `mcp__codex_apps__veeva_veeva_create_record`, one record at a time. Use `veeva_get_object_fields` if field requirements are uncertain.
- Google Sheets/Drive: use `google_drive_search`, `google_drive_get_spreadsheet_range`, `google_drive_get_spreadsheet_cells`, `google_drive_batch_update_spreadsheet`, `google_drive_fetch`, and `google_drive_export_file` as appropriate. Use a complete export or workbook download when exact columns matter.
- Photos/PDFs: inspect every relevant page with the available PDF/image tools.

## Safety and scope

1. This skill creates production Veeva records. Show an exact preview and obtain explicit confirmation immediately before any create call.
2. Re-read the tracker and Veeva source data immediately before the create call.
3. After Veeva creation and verification, write each newly created RQC display ID into the matching source tracker row's RQC column. Restrict the write to the exact verified rows, preserve all other cells, and re-read the written cells to confirm the values.
4. Never trust the tracker’s RQC column as proof that a record exists; query Veeva.
5. Never guess scope, photo-to-reagent matching, barcode mappings, or mismatches. Ask and stop when unresolved.
6. The selected batch may shrink during validation. Once RQCs are created, it is fixed for any later manifest.
7. Do not silently create a manifest. Confirm whether one is wanted under the rules below.

## Route

Use this skill for:

- an RQS/shipment plus “create RQCs,” “log reagents,” or equivalent;
- a tracker row or shipment that needs RQC records;
- photos attached for an RQC-creation request;
- a bare tracker/shipment request with no manifest language.

Do not use this skill for an explicit `RQ-*` qualified-manifest request or a manifest-only request with existing RQCs; use the standalone accessioning-manifest skill instead. If a shipment-only manifest request has no RQCs, tell the user and ask whether they want to create RQCs first.

## Tracker sources

Search both trackers when an RQS is named; one shipment can appear in both.

- Exome+ file ID `1_FBszw8VM22zqEOBOqea-4TzxqiHIhKmwS-wmOue5PY`, tab `Exome_Materials Received`: A product, B box part, C box name, D kit part, E box lot, F item lot, G expiration, H quantity, I received date, J TC#/item serial, N RQS, O RQC, P RQ.
- Confirmatory file ID `1e0O0O4np4itZdNvFunvX-yfdq01p9OkyKWon8Aa-K9g`, tab `WGS_Materials Received`: A reagent, B box part, C kit content number, D description/box name, E box lot, F item lot, G expiration, H count, I received date, J RQS, K RQC, L RQ, M QC status, N date passed, O in calendar, P comment.

The Confirmatory sheet has no TC# column. Only set `part_serial_number_tc__c` when verified elsewhere.

Match trimmed names case-insensitively and dates as `MM/DD/YYYY`. Skip blank rows; stop on blank required cells. If matching by product/date reveals an RQS, resolve it and rerun the search across both trackers for the complete shipment before asking scope.

## Workflow

### 1. Resolve the shipment

For a known RQS display ID:

```vql
SELECT id, name__v
FROM reagent_shipment__c
WHERE name__v = '<RQS-XXXXXX>'
```

If not found, stop and ask. Use the internal `id` as `shipment__c`.

### 2. Confirm scope and photos

Ask one combined question every run:

> RQS-000681 has 6 tracker reagents: TAP, THB, SNP2, ExV2, GAP5, and CNV2. Proceed with all, or only some? Do you have box/tube photos to cross-check before I prepare the RQC preview?

Do not ask again if the user already answered for this run. Hold early attachments until the final scope is known.

### 3. Check existing RQCs

Query Veeva by product, TC#, shipment, or verified lot/shipment combination. For example:

```vql
SELECT id, name__v, kit_component__c, box_lot_number__c,
       component_lot_number__c, expiration_date__c, quantity__c,
       part_serial_number_tc__c, shipment__c
FROM reagent_qual_component__c
WHERE kit_component__c = '<product name>'
```

If a matching record exists, report its `RQC-*` and remove that reagent from the create batch. Never duplicate it.

### 4. Cross-check photos

Compare product, kit/REF, item/component lot, box part/lot, expiration, and serial. When box and tube serials differ, use the tube/item serial. Check every PDF page; labels and barcodes can be on different pages.

Known exceptions: Pre-Load Buffer uses the tube item serial; Helix UDI plates may have the serial on a tray/box-edge sticker; 25B cartridges, buffer cartridges, and lyo inserts use the box as item serial. A barcode missing from a photo is inconclusive. Show both values and stop on any mismatch; do not resolve it yourself.

### 5. Check recent consistency

Read the newest 20 RQCs and flag wording, lot-format, spelling, or unexpected-blank drift:

```vql
SELECT id, name__v, kit_component__c, box_part_number__c, box_name__c,
       kit_part_number__c, box_lot_number__c, component_lot_number__c,
       expiration_date__c, quantity__c, part_serial_number_tc__c, shipment__c
FROM reagent_qual_component__c
ORDER BY created_date__v DESC
```

Confirmatory records may legitimately lack a TC#.

### 6. Preview and create

Show every candidate row with exact values, source sheet, scope, existing-record findings, photo findings, and anomalies. Get explicit confirmation. Then create one `reagent_qual_component__c` record at a time with:

```text
kit_component__c
box_part_number__c
box_name__c
kit_part_number__c
box_lot_number__c
component_lot_number__c
expiration_date__c: YYYY-MM-DD
quantity__c
part_serial_number_tc__c: only when verified
shipment__c: resolved internal ID
```

Do not set `date_shipment_received__c`, `name__v`, or lifecycle/status fields; Veeva derives them.

### 7. Verify and report

Re-query the shipment:

```vql
SELECT id, name__v, box_name__c, component_lot_number__c, expiration_date__c
FROM reagent_qual_component__c
WHERE shipment__c = '<shipment_internal_id>'
ORDER BY name__v ASC
```

After the re-query confirms the newly created records, update the RQC column in the source tracker for the exact rows used to create them. Match by the verified shipment, product, lot, and received-date data; never guess a row. Batch the cell update, then re-read the written cells and confirm each tracker value equals the Veeva display ID.

Report each new RQC mapped to reagent and lot, the tracker rows updated, existing-RQC exclusions, tracker anomalies, and any unresolved issue.

## Optional manifest handoff

Confirmatory-sourced batches do not receive a manifest automatically. For Exome+ or direct RQC batches, ask whether a manifest is wanted. If requested, route to the accessioning-manifest workflow and ask initial qualification versus qualified reagent unless already answered.

## Shared manifest rules — version 2

These rules are intentionally duplicated in the accessioning-manifest skill so each skill can be shared independently.

- `QC=TRUE` and no `Qualified` filename label means initial qualification: one row and one barcode per reagent.
- `QC=FALSE` and `Qualified` in the filename means qualified reagent.
- Validate every barcode at collection, after parsing, immediately before each workbook write, and during 25B rewrites. Default pattern is `XX#######-XXXX`; exceptions are ADA+digits, lowercase `x` in ExV2, and Buffer Cartridge `BC` + variable digits + numeric suffix. No spaces.
- If Box and Item Serial are both present, use Item Serial. Never repair or reorder silently.
- In `.xlsx`, columns A/C are text with `number_format='@'`; column D is a `datetime.date` with `number_format='MM/DD/YYYY'`. Reapply and verify after every reopen/rewrite.
- Keep manifests machine-readable and visually plain: visible gridlines, plain white sheet, no merged cells, no colored header fills, no decorative borders, no charts, and no conditional formatting unless explicitly requested. A plain bold header row and basic column widths are sufficient.
- After a manifest is built and validated, upload the created workbook to the user's shared Drive upload folder `10Gsp1uSPXPFjzuV-ignSYgweMxy9_gv1` unless the user explicitly asks not to upload it. Report the uploaded Google Drive link. The separate 25B deduplication folder remains `1P-1a91iDcEplWxlpJdmnC1LDA8heYyuQ`.
