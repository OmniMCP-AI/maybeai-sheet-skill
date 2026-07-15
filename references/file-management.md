# File Management Reference

## Contents

1. When to use this
2. Basic conventions
3. Engine selection
4. Core endpoints
5. Sharing and permissions
6. Recommended flows

## 1. When to use this

Read this document when the task involves uploading, importing, searching, copying, renaming, deleting, sharing, or exporting MaybeAI spreadsheets.

## 2. Basic conventions

- Base URL: `https://play-be.omnimcp.ai`
- Auth header: `Authorization: Bearer <MAYBEAI_API_TOKEN>`
- Most follow-up requests use:

```text
https://www.maybe.ai/docs/spreadsheets/d/<document_id>
```

- After upload succeeds, record:
  - `document_id`
  - `uri`

## 3. Engine selection

MaybeAI Sheet uses Playground as the product-level router. It can create workbooks in different engines:

- `excelize`: workbook-style runtime for Excel layout, styles, formulas, merged cells, and workbook semantics.
- `postgres` / `pg`: SheetTable runtime for table-like data, SQL, large row counts, large cell counts, append/upsert, and PG-native reads/writes.

Best practice:

- Inspect unfamiliar files before import with `POST /api/v1/excel/import/plan` and multipart `engine=auto`.
- Use `engine=postgres` for a worksheet only when it is one flat table, has more than 5,000 rows, and every data column has one datatype except the header and missing values. The older high-cell-count preference still applies when the data is one homogeneous table.
- If the task depends on Excel-specific workbook fidelity, use the Excelize upload path and keep the dataset size modest.
- Use Excelize for reports, summaries, dashboards, formulas, merged cells, styled workbooks, or worksheets with multiple separated tables. For example, `L1_广州瑞鹏_详细` in the LLM cost analysis workbook has two tables in one worksheet and should use Excelize.
- A small single table such as `L1_客户集中度_帕累托` can use PG or Excelize; auto may choose Excelize because the sheet is not large.
- Explicit `engine=postgres` is strict. If the worksheet is not PG-compatible, expect `PG_IMPORT_UNSUPPORTED_LAYOUT`. Use `engine=auto` when fallback to Excelize is acceptable.
- Avoid sending large table data through row-object JSON writes or `/api/v1/excel/upload`; that path can fail when the server expands the workbook into large in-memory payloads.
- After importing a file, check response `worksheet_engines[].selected_engine`, `worksheet_engines[].final_engine`, and any `fallback_reason`; then call `worksheet/metadata` and confirm `engine: "pg"` / `data_engine: "pg"` for PG sheets and `excelize` for workbook-layout sheets. Use `list_worksheets` only when you need legacy compatibility.

## 4. Core endpoints

### Import a large table-like file into SheetTable/PG

```text
POST /api/v1/excel/import
```

Use this for large table-like `.xlsx` files: more than 10,000 rows in any worksheet, more than 100,000 populated cells across the workbook, or data shaped primarily as records.

```bash
curl -sS -X POST "$BASE_URL/api/v1/excel/import" \
  -H "Authorization: Bearer $MAYBEAI_API_TOKEN" \
  -F "engine=postgres" \
  -F "file=@/absolute/path/to/file.xlsx"
```

For unknown or mixed workbooks, inspect first and import with auto:

```bash
curl -sS -X POST "$BASE_URL/api/v1/excel/import/plan" \
  -H "Authorization: Bearer $MAYBEAI_API_TOKEN" \
  -F "engine=auto" \
  -F "file=@/absolute/path/to/file.xlsx"

curl -sS -X POST "$BASE_URL/api/v1/excel/import" \
  -H "Authorization: Bearer $MAYBEAI_API_TOKEN" \
  -F "engine=auto" \
  -F "file=@/absolute/path/to/file.xlsx"
```

Expected success shape:

```json
{
  "success": true,
  "documentId": "<document_id>",
  "fileUri": "https://www.maybe.ai/docs/spreadsheets/d/<document_id>",
  "sheets": ["Sheet1"]
}
```

Verification:

```bash
curl -sS -X POST "$BASE_URL/api/v1/excel/worksheet/metadata" \
  -H "Authorization: Bearer $MAYBEAI_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"uri":"https://www.maybe.ai/docs/spreadsheets/d/<document_id>"}'
```

Confirm the response reports the expected final engine for each worksheet.

### Upload a file

```text
POST /api/v1/excel/upload
```

Notes:

- `multipart/form-data`
- `file` is required
- `user_id` is optional and exists only as a compatibility field
- Use this for Excelize/workbook-style imports where layout, styles, formulas, merged cells, or workbook fidelity matter more than table scale

Script: `scripts/01-file-management.sh`

### Import from URL

```text
POST /api/v1/excel/import_by_url
```

Use this when you already have a public downloadable `.xlsx` URL.

### List or search files

```text
POST /api/v1/excel/list_files
POST /api/v1/excel/search_files
```

Use search when you need to find historical files by keyword.

### Rename, delete, or copy

```text
POST /api/v1/excel/rename_file
POST /api/v1/excel/delete_file
POST /api/v1/excel/copy_excel
```

These endpoints all operate on the same `uri`.

### Export

```text
GET /api/v1/excel/export/{document_id}
POST /api/v1/excel/download
```

Guidance:

- Prefer `export` when you want the `.xlsx` file directly
- Use `download` when you already have a `uri`

## 5. Sharing and permissions

### Visibility

```text
POST /api/v1/share/sheet/visibility
```

### Share with a specific user

```text
POST /api/v1/share/sheet/update-permission
```

### Inspect permissions

```text
POST /api/v1/share/sheet/list
POST /api/v1/share/sheet/permission
```

## 6. Recommended flows

### Bring a new file into the system

1. Inspect approximate row count and workbook intent
2. For unfamiliar files, call `/api/v1/excel/import/plan` with `engine=auto`
3. If table-like and rows > 5,000 with homogeneous columns, use `/api/v1/excel/import` with `engine=postgres`
4. If mixed or workbook-layout, use `/api/v1/excel/import` with `engine=auto` or Excelize
5. Record `document_id` and `uri`
6. `worksheet/metadata`
7. `worksheet/dimensions` when row/column counts or used ranges matter
8. `read_headers` or a small `read_sheet`

### Bring a large table-like file into the system

1. `POST /api/v1/excel/import` with multipart `engine=postgres`
2. Record `document_id` and `uri`
3. `worksheet/metadata`
4. Confirm `engine: "pg"` or worksheet `data_engine: "pg"`
5. `worksheet/dimensions` when row/column counts or used ranges matter
6. `read_headers` or a small `read_sheet`

### Export before delivery

1. Finish all writes
2. Read back the key ranges with `read_sheet`
3. `export` the final file

### Reuse a historical file

1. `search_files`
2. `copy_excel`
3. Edit the copy instead of the original
