# Read/Write Reference

## Contents

1. When to use this
2. Worksheet targeting rules
3. Worksheet metadata and dimensions
4. Read endpoints
5. How to choose a write API
6. Row and column operations
7. Worksheet management
8. Post-write verification

## 1. When to use this

Read this document when the task involves reading sheets, sampling data, reading headers, updating cells, replacing full tables, updating by key, appending rows, inserting or deleting rows and columns, or creating and renaming worksheets.

## 2. Worksheet targeting rules

This is the most important operational rule.

- Prefer `worksheet_name`
- Some endpoints only respect `uri?gid=<index>`
- If you pass neither, the backend often defaults to the first worksheet

Typical rules:

- `read_sheet` / `update_range` / `clear_range` / `update_data_keep_headers`
  Prefer `worksheet_name`
- `read_headers` / `append_rows` / `update_range_by_lookup`
  Commonly use `uri?gid=<index>`

If the user says “update the second sheet” or “append to Summary”, identify the sheet first, then execute the write.

## 3. Worksheet metadata and dimensions

Playground split worksheet discovery into two explicit read-only APIs. Both require viewer access and are available on the compatibility path used by this skill:

```text
POST /api/v1/excel/worksheet/metadata
POST /api/v1/excel/worksheet/dimensions
```

The canonical V2 paths are also available:

```text
POST /api/v1/excel_v2/worksheet/metadata
POST /api/v1/excel_v2/worksheet/dimensions
```

### `worksheet/metadata`

Use this first when you need to identify worksheets before reading or writing.

What it is:

- A registry-first worksheet metadata endpoint.
- It returns logical worksheet identity without blocking on full row/column scans.
- It replaces the old habit of calling `list_worksheets` just to learn sheet names, gids, order, URLs, source routing, or engines.

Typical request:

```json
{
  "uri": "https://www.maybe.ai/docs/spreadsheets/d/<document_id>"
}
```

Useful response fields:

- Workbook fields: `document_id`, `spreadsheet_id`, `spreadsheet_url`, `spreadsheet_title`, `worksheet_count`
- Worksheet identity: `title`, `name`, `worksheet_name`, `sheet_name`, `gid`, `sheet_id`, `index`, `worksheet_url`
- Engine routing: `data_engine`, `style_engine`, `sql_engine`, `formula_engine`
- Source routing: `source.document_id`, `source.gid`, `source.engine`, `source.logical_gid`, `source.logical_name`
- Metadata marker: `source_info.operation: "worksheet_metadata"`, `source_info.metadata_only: true`, `source_info.metadata_source: "registry"`

Do not expect this endpoint to return full `row_count`, `dimensions`, or `tables`. It may include a cheap `column_count` when PG metadata or a valid cache is available, but row counts and used ranges belong to `worksheet/dimensions`.

Use it when:

- choosing the correct worksheet by name or gid
- verifying sheet order and worksheet URLs
- checking whether each sheet is routed to `pg` or `excelize`
- confirming the final engine after import
- preparing a write request that must not default to the first sheet

### `worksheet/dimensions`

Use this only when dimensions actually matter.

What it is:

- A worksheet dimension lookup endpoint.
- It calls the engine-specific `worksheet_dimensions` upstream and returns exact row/column metadata when available.
- It does not include the old identity probe behavior; `source_info.identity_probe_included` should be `false`.

Typical all-sheet request:

```json
{
  "uri": "https://www.maybe.ai/docs/spreadsheets/d/<document_id>"
}
```

Target a single worksheet in large workbooks:

```json
{
  "uri": "https://www.maybe.ai/docs/spreadsheets/d/<document_id>",
  "gid": 7
}
```

or:

```json
{
  "uri": "https://www.maybe.ai/docs/spreadsheets/d/<document_id>",
  "worksheet_name": "Orders"
}
```

Useful response fields:

- `row_count`, `column_count`
- `dimensions.rows`, `dimensions.columns`, `dimensions.ref`
- `tables` when the upstream can identify table ranges
- `row_count_exact`, `column_count_exact`
- `dimension_source`, `column_count_source`
- `dimension_stale`, `dimension_refreshing`
- `source_info.dimension_source`: `upstream_enriched`, `registry_cache`, `upstream_and_registry_cache`, `partial`, or `registry_only`
- `source_info.dimension_errors` when some dimensions could not be fetched

For Excelize-backed worksheets, `force_refresh` and `use_cache` can control dimension cache behavior:

```json
{
  "uri": "https://www.maybe.ai/docs/spreadsheets/d/<document_id>",
  "worksheet_name": "Orders",
  "force_refresh": true,
  "use_cache": false
}
```

These cache controls are not forwarded to PG-backed worksheets.

Use it when:

- deciding whether a sheet is too large for a full `read_sheet`
- selecting a bounded `range_address`
- estimating import or analysis size
- finding the used range before writing a report block
- checking table hints before SQL or bulk updates
- verifying row/column dimensions after row, column, import, or worksheet operations

### Relationship to `list_worksheets`

`list_worksheets` remains available and is fine for legacy scripts or simple flows:

```text
POST /api/v1/excel/list_worksheets
```

Use the split APIs for newer agent workflows:

- Need names, gids, URLs, sources, or engines: call `worksheet/metadata`.
- Need row counts, column counts, used ranges, tables, or exactness flags: call `worksheet/dimensions`.
- Need both: call `worksheet/metadata` first, then targeted `worksheet/dimensions` for the sheet(s) that matter.

## 4. Read endpoints

### Read a full sheet or a range

```text
POST /api/v1/excel/read_sheet
```

Common parameters:

- `worksheet_name`
- `range_address`
- `value_render_option`
- `filter_tokens`
- `auto_filter`

Use it to:

- inspect data
- sample and verify
- read chart or formatting metadata

### Read headers

```text
POST /api/v1/excel/read_headers
```

Use it to:

- get the schema quickly
- confirm column names before writing SQL

### List worksheets and versions

```text
POST /api/v1/excel/list_worksheets
POST /api/v1/excel/list_worksheets_version
POST /api/v1/excel/list_versions
POST /api/v1/excel/read_version
```

Prefer `worksheet/metadata` over `list_worksheets` when you only need sheet identity or engine routing. Prefer `worksheet/dimensions` when you need row/column counts or used ranges.

## 5. How to choose a write API

### `update_data_keep_headers`

Best when:

- headers are already correct
- you need to replace the entire data region
- you want to preserve column order
- you want to preserve formula columns

Advantages:

- input can be list-of-dict
- safer for agents

### `update_range_by_lookup`

Best when:

- syncing business records by key
- updating existing rows
- appending missing rows automatically

Common keys:

- `Order ID`
- `SKU`
- `ID`

### `append_rows`

Best when:

- you want a blind append of object rows
- the target sheet and headers are already known

### `update_range`

Best when:

- you need to update an exact A1 range
- the target is non-tabular
- you are making a small manual cell edit

Value handling:

- `update_range` defaults to `RAW`; numeric-looking strings such as `"5.53%"` and `"9,007,000"` stay strings.
- Use `value_input_option=USER_ENTERED` only when you want Excel-like parsing of formulas, dates, numbers, and percentages.
- Read the response `message` after writes:
  - `parse_result=NOT_REQUESTED` means `RAW` kept numeric-looking strings as text; inspect `preserved_values`.
  - `parse_result=PASS` means `USER_ENTERED` parsed the submitted numeric-looking strings; inspect `parsed_values`.
  - `parse_result=PARTIAL` means values in `parsed_values` parsed, while values in `preserved_text_values` may stay text unless the target cells are numeric-formatted.

### `clear_range`

Best when:

- you need to clear a specific range
- you want a local reset before a write

## 6. Row and column operations

Related endpoints:

```text
POST /api/v1/excel/insert_rows
POST /api/v1/excel/delete_rows
POST /api/v1/excel/move_row
POST /api/v1/excel/move_rows
POST /api/v1/excel/undo_delete_rows
POST /api/v1/excel/insert_columns
POST /api/v1/excel/delete_columns
POST /api/v1/excel/move_column
POST /api/v1/excel/move_columns
POST /api/v1/excel/undo_delete_columns
POST /api/v1/excel/add_header_columns
POST /api/v1/excel/set_columns_width
POST /api/v1/excel/set_rows_height
```

Notes:

- row numbers are 1-based
- columns typically use Excel letters such as `A` and `B`

## 7. Worksheet management

Related endpoints:

```text
POST /api/v1/excel/write_new_worksheet
POST /api/v1/excel/delete_worksheet
POST /api/v1/excel/rename_worksheet
POST /api/v1/excel/move_worksheet
POST /api/v1/excel/copy_worksheet
```

Guidance:

- When creating a new report sheet, write data first and style it separately
- Before deleting a worksheet, confirm the `gid` or sheet name to avoid deleting the wrong sheet

## 8. Post-write verification

Do at least one of the following:

- `read_sheet`
- `read_headers`
- `worksheet/metadata` or `list_worksheets`
- `worksheet/dimensions` when row/column counts should change

Strongly recommended after:

- `update_data_keep_headers`
- `update_range_by_lookup`
- writes to non-first worksheets
- `write_new_worksheet`

Related scripts:

- `scripts/02-read-data.sh`
- `scripts/03-write-data.sh`
- `scripts/04-rows-columns.sh`
- `scripts/05-worksheets.sh`
