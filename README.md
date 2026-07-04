# CreateRQC Skill

A [Claude / Cowork](https://claude.ai) **skill** that creates **RQC (Reagent - Component)** records in **Veeva QualityOne** from a reagent inventory tracker spreadsheet.

Given a **Kit Content Product Name** and a **shipment received date**, the skill:

1. Finds the matching row(s) in the reagent inventory tracker sheet.
2. Maps the row's fields to the Veeva `reagent_qual_component__c` object.
3. Resolves the shipment (RQS) reference to its Veeva record ID.
4. Converts the expiration date (MM/DD/YYYY → YYYY-MM-DD).
5. Previews the record and **waits for explicit confirmation** before writing.
6. Creates one RQC per matching row and reports the new `RQC-####` IDs.

## Setup

This is a generalized copy with internal identifiers removed. Before use, edit `SKILL.md` and replace:

- `<SHEET_FILE_ID>` — the Google Drive file ID of your tracker sheet.
- Object/field API names if your Veeva configuration differs.

## Requirements

- A Veeva connector exposing create-record and VQL query tools.
- A Google Workspace / Sheets connector for reading the tracker.

## Safety

The skill only writes to Veeva after showing a preview and receiving explicit user confirmation, and it refuses to create records with missing required fields.

## License

MIT
