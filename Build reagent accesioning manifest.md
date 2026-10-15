---
name: build-reagent-accessioning-manifest
description: Build and validate reagent accessioning .xlsx manifests from Veeva RQ/RQC/RQS data, tracker data, photos, or barcode files. Use for RQ-, RQC-, RQS-manifest, barcode-upload, accessioning, or 25B-dedup requests; do not create Veeva RQC records here.
metadata:
  short-description: Build and validate accessioning manifests
---

# Build reagent accessioning manifest

This is the standalone manifest skill adapted from the Claude workflow in `quintinromano-code/reagent-qualification-skills`.

## Codex tools and boundary

- Veeva reads: `mcp__codex_apps__veeva_veeva_get_record` for a known display ID and `mcp__codex_apps__veeva_veeva_vql_query` for read-only SELECT queries.
- Google Drive/Sheets: use `google_drive_search`, `google_drive_get_spreadsheet_range`, `google_drive_fetch`, `google_drive_export_file`, and `google_drive_upload_file`. Upload validated manifests to the user's shared Drive upload folder `10Gsp1uSPXPFjzuV-ignSYgweMxy9_gv1` unless the user explicitly asks not to upload.
- Local workbooks: use the bundled workspace dependencies and `openpyxl`; return the finished workbook as an absolute-path Markdown link.
- PDFs/photos: inspect every relevant page with the available PDF/image tools.

This skill never creates or updates Veeva records. If RQCs do not exist and the user wants them created, hand off to `$create-rqc-records`.

## Non-negotiable safeguards

1. Route first. Never assume a prior RQ/RQS, scope, manifest type, or barcode mapping carries over.
2. Re-read Veeva/source data immediately before every workbook write.
3. Never silently repair barcodes, resolve photo/data mismatches, or guess unlabeled barcode order.
4. A real serial produces one row; `Manifest` produces `quantity__c` rows with blank serials.
5. `Qualified` in the filename always pairs with `QC=FALSE`; initial qualification uses `QC=TRUE` and no `Qualified` label.
6. Columns A and C must be text (`number_format='@'`); column D must be a genuine `datetime.date` with `number_format='MM/DD/YYYY'`. Reapply after every reopen/rewrite.
7. After the workbook passes validation, upload the created manifest to the shared Drive upload folder `10Gsp1uSPXPFjzuV-ignSYgweMxy9_gv1`, unless the user explicitly asks not to upload it. Do not upload a failed or unverified workbook. Report the uploaded Drive link.
8. Keep the workbook machine-readable and visually plain: use a single simple worksheet, visible gridlines, no merged cells, no colored header fills, no decorative borders, no charts, and no conditional formatting unless the user explicitly requests them. A plain bold header row and basic column widths are sufficient.

## Route

- `RQ-*`: Path A qualified reagent manifest.
- Bare `RQC-*`: direct-entry manifest from RQC data.
- RQS/shipment plus manifest-only language: query for RQCs. Existing RQCs continue here; no RQCs requires a decision or handoff to RQC creation.
- Barcode file for a manifest already built in the current or prior session: Phase 2, after identifying the workbook.
- Shipment-only case with RQCs: ask whether this is initial qualification or qualified reagent before building.

## Path A — qualified manifest

### 1. Resolve the RQ/RQCs

For an RQ, confirm it with `veeva_get_record` and/or:

```vql
SELECT id, name__v, batch_id__c, comments__c, status__v, state__v, created_date__v
FROM reagent_qualification__c
WHERE name__v = 'RQ-XXXXXX'
```

Find associated RQCs using the part serial in `comments__c`, the RQ creation-date window, and a trailing component-name wildcard. If `shipment__c` is known, query by shipment directly. VQL has no `IN (...)`; query individually or use a shared field.

For shipment-only entry:

```vql
SELECT id, name__v, kit_part_number__c, kit_component__c,
       component_lot_number__c, expiration_date__c,
       part_serial_number_tc__c, quantity__c, shipment__c
FROM reagent_qual_component__c
WHERE shipment__c = '<shipment_internal_id>'
```

Exclude case-insensitively: `1.5B Sequencing Kit`, `10B Sequencing Kit`, and `SPRQ Sequencing Plate`. If everything is excluded, report that no accessioned reagent remains.

### 2. Determine shipment date and filename

```vql
SELECT id, name__v, date_shipment_received__c
FROM reagent_shipment__c
WHERE id = '<shipment__c>'
```

Use `RQS-######_<Reagent Label>_recDDMonYYYY.xlsx`. If shipment received date is null, use RQ creation date and disclose the fallback.

Labels:

- Single component: component + ` Qualified`.
- Complete Twist Probes (THB, TAP, ExV2, CNV2, GAP5, SNP2): `Twist Probes Qualified`.
- Complete Wash Buffer (WBS, WB1, WB2, WB3, SAB): `Wash Buffer Kit Qualified`.
- Complete 25B (Lyo Insert, Buffer Cartridge, Pre Load Buffer, Reagent Cartridge): `25B Sequencing Reagents Qualified`.
- Complete Helix UDI set (UP01–UP16): `Helix UDI 10nt Primer Plates Qualified`.
- Partial/unrecognized mix: component names separated by ` / ` + ` Qualified`.
- Multiple complete families: family names separated by ` / ` + ` Qualified` once.

### 3. Build and verify the workbook

Header:

```text
Kit Content Part Number | Kit Content Product Name | Item Lot Number | Expires On | Part Serial Number | Lot Status | QC | Notes
```

Use `openpyxl`. For every row:

- A and C: strings with `number_format='@'`.
- D: `datetime.date`, never a date-looking string, with `number_format='MM/DD/YYYY'`.
- F: `Active`.
- H: empty string.
- Qualified manifest: G=`FALSE`.

Keep the presentation basic for downstream system ingestion: leave gridlines visible, use a plain white sheet, avoid merged cells and decorative styling, and do not add tables, charts, conditional formatting, or hidden rows/columns unless explicitly requested. The header may be bold but should not use a colored fill.

Verify after saving and after every later rewrite that A/C are text, D is date-typed, and no unrelated columns changed. In Codex, save to the task workspace and return an absolute-path Markdown link.

## Initial qualification manifests

Use this mode for a direct RQC batch or shipment-only case already confirmed as initial qualification.

- One row per reagent regardless of `quantity__c`.
- G=`TRUE`.
- One barcode per reagent.
- No `Qualified` in filename.
- Do not re-ask manifest type if the shipment-only decision already answered it.

For Exome+ or direct RQC batches, do not build automatically unless the request clearly asks for one or the user confirms. Confirmatory-sourced batches are skipped automatically unless explicitly requested.

## Barcode validation and Phase 2

Validate every serial at collection, after file parsing, immediately before each workbook write, and during any 25B rewrite.

Default pattern:

```text
XX#######-XXXX
```

Two capital letters, seven digits, hyphen, capital-letter suffix; digits may appear in the suffix; no spaces. Confirmed exceptions are ADA+digits, lowercase `x` in an ExV2 suffix, and Buffer Cartridge `BC` + variable digits + numeric suffix such as `BC1100059171-1`.

Known 25B suffixes: `-LI#` lyo insert, `-TT#` Pre-Load Buffer, `-25B#` reagent cartridge, `BC...-1` buffer cartridge.

When Box and Item Serial columns both exist, always use Item Serial. Group supplied barcodes by reagent, reconcile supplied count with blank rows, and verify lot/part data. A mismatch stops the write. Multiple unlabeled barcodes require explicit mapping confirmation.

For a later barcode upload:

1. Identify or request the manifest workbook.
2. Read every barcode-file sheet with `openpyxl` or `csv.reader`.
3. Fill only column E after validation and reconciliation.
4. Reapply A/C text and D date formatting on every touched row.
5. Reload and verify serials, formatting, and unchanged non-serial columns. Report remaining blanks by reagent.

## Phase 3 — 25B deduplication

Run only for 25B qualified manifests (`QC=FALSE`) after all barcodes are filled.

- Shared dedup folder: `1P-1a91iDcEplWxlpJdmnC1LDA8heYyuQ`.
- Older private folder `1xBq0_DWabo6Wi9Gfa_Fq6CyIXlnohpnr` is reference-only.
- Search the shared folder for prior 25B manifests and inspect prior `.xlsx` rows.
- Remove current rows whose barcode appears in a prior manifest with `QC=TRUE`; these are single-use.
- Compact remaining rows, reapply A/C and D formats, revalidate carried-forward barcodes, and reload to verify.
- If no prior file is found, say only that the shared-folder search was clean; an unshared file may still exist.
- After deduplication and final validation, upload the current file to the shared Drive upload folder `10Gsp1uSPXPFjzuV-ignSYgweMxy9_gv1`, unless the user explicitly asks not to upload it. The deduplication folder and upload folder are separate.

## Reporting requirements

Always provide a report, not just a file link. Include filename, row counts by reagent, prefilled versus supplied barcodes, serial column chosen, validation result, mismatches, formatting checks, 25B rows removed and folder searched, and any unresolved blocker. State the next user action when one is required.

## Shared rules — version 2

These rules are intentionally duplicated in the RQC skill so each skill can be shared independently. Keep the version marker synchronized when either copy changes.
