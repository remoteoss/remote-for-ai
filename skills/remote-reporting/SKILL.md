---
name: remote-reporting
description: Create and download reports via tools for employers. Use when asked about reports, reporting library, report builder, payroll reports, exporting report data, CSV/XLS/XLSX downloads, filtered report runs, generated report files, or required report filters.
license: MIT
---

# Reporting via tools

## Critical Rules

**Required filters come from report building blocks.** Before creating a report run, call `get_report_building_blocks` for the selected report and set every required filter it identifies.

**Preview before running.** Before creating a report run, call `preview_report` with the selected report template, filters, grouping, and sample size. Use the preview to verify the report shape and adjust fields, grouping, or filters before calling `create_report_run`.

**Always render the preview as a full markdown table for the user.** Do not summarise it, do not truncate columns, do not show a few sample rows — show every row returned by `preview_report` with every column from the template. If the table is wide, render it wide; the user needs to see the actual shape before approving the run. Follow the preview table with a single short line summarising what they're looking at (period, filters, row count) so they can decide whether to run it.

**Vocabulary:** A **report** is a `data_export_template`. A **report run** is a `data_export`.

**Never expose internal mechanics to the user.** The user should never need to know about base64 decoding, data URIs, tool scoping limits, response truncation, or pagination quirks. Handle all of that silently. If something fails, retry with the documented workaround before surfacing anything to the user. The only time to surface a mechanics-level problem is when the user must take action in the UI (e.g. paste a slug from the URL).

---

## Employer Context

### Start with report discovery

- `list_reports` — **Start here for reporting questions.** Use `category` when the user names a domain, for example `category=payroll` for payroll reports. Prefer `published_to=reporting_library` unless the user asks for customer-created reports.

### Need a report template slug (get from `list_reports`)

- `get_report` — Fetch one report template before creating a run when the user names a specific report.
- `get_report_building_blocks` — Inspect fields, variables, and required filters for the template. **Call this before every `preview_report` and `create_report_run`.**
- `get_report_filter_schema` — Inspect allowed filters and field names when building filtered report runs.
- `get_report_filter_values` — Resolve valid values for a filter field before passing filters to a report run.
- `preview_report` — Preview a sampled result before creating a report run. **Call this before every `create_report_run` unless the user explicitly asks to skip preview.**

### Need values for required filters

- `legal_entity_slug` — Resolve from `employer_resolve_legal_entity_by_name` when the user gives a legal entity name. Use `list_employer_legal_entities` when listing or narrowing entities by country, status, payroll status, active services, or partial query.
- `payroll_run_slug` / payroll slug — Resolve from `list_unified_payroll_runs`. Filter by legal entity, country, payroll period, or status when the user gives that context. Use `get_payroll_run` only after choosing a specific payroll run from the list.
- Other required filters — First try `get_report_filter_values` for the required field. If it returns values, choose from those. If it requires another domain object, resolve that object with the relevant list/get tool before creating the report run.

### Need a report run slug (get from `create_report_run` or `list_report_runs`)

- `get_report_run` — Inspect a report run, including validation errors if file generation fails.
- `poll_report_run_download` — Start or poll async file generation. Poll until `status` is `file_creation_completed`.

### Need a file_slug (get from `poll_report_run_download`)

- `download_file` — Download the generated report file after `poll_report_run_download` returns `file_slug`.

### Workflows

**"List reports"**

1. Call `list_reports`.
2. Add `category` when the user names a domain, for example `payroll`, `billing`, or another available category.
3. If the user wants Remote-published reports, include `published_to=reporting_library`.
4. Present template names, descriptions, and slugs only when useful for follow-up tool calls.

**"List payroll reports"**

1. Call `list_reports` with `category=payroll`.
2. Prefer `published_to=reporting_library` unless the user asks for customer-created reports.
3. Present template names, descriptions, and slugs only when useful for follow-up tool calls.

**"Create/run a report for X"**

1. `list_reports` to resolve the template. Use `category=payroll` when the user asks for payroll. If the report the user named does not exist, tell them clearly and show what is available.
2. `get_report_building_blocks` for the chosen template and identify required filters.
3. **Check that data exists for the user's requested period before doing anything else.** If the user named a month, quarter, or year, call `list_unified_payroll_runs` (or the relevant domain-list tool) for that window first. If it returns zero rows, do not run an empty report — surface the nearest periods that *do* have data and offer them as alternatives (recommend the closest match first).
4. Resolve any required domain-object filters:
   - legal entity filters: `employer_resolve_legal_entity_by_name` or `list_employer_legal_entities`
   - payroll run filters: `list_unified_payroll_runs`, then `get_payroll_run` if needed
5. `get_report_filter_schema` to map required filters to valid filter field names and operators.
6. For each required or user-requested filter that needs controlled values, call `get_report_filter_values`.
7. **For variance / period-over-period / comparison templates**: the `required_filters` from `get_report_building_blocks` only flags the toggle (e.g. `calculate_variance`). You must *also* set the two period brackets (`variance_period_1_start/end` + `variance_period_2_start/end`) **or** the two run slugs (`variance_payroll_run_1` + `variance_payroll_run_2`). Without them, the report is empty. The user will not name these — infer them from their period (e.g. "March variance" → period 2 = March, period 1 = February).
8. Call `preview_report` with the template `slug`, optional sample size, grouping choices, and all required or user-requested filters.
9. **Render the preview as a full markdown table** (every row, every column from the template). Add a one-line caption: period(s), filters, row count.
10. Inspect the preview for expected columns, grouping, and filter effects. If the preview does not match the user's request, adjust the fields, grouping, or filters and call `preview_report` again.
11. Call `create_report_run` with the template `slug`, report name, optional format, grouping choices, and all required filters.
12. Use the returned report run slug as the report run slug.

**"Download/export this report"**

1. Resolve the report run slug from `create_report_run`, `list_report_runs`, or `get_report_run`.
2. Call `poll_report_run_download`.
3. If `status` is `file_creation_in_progress`, wait briefly and call `poll_report_run_download` again with the same slug and format.
4. When `status` is `file_creation_completed`, call `download_file` with `file_slug`.
5. **`download_file` returns a data URI, not raw file bytes.** Response shape: `{"data":{"content":"data:text/csv;base64,<base64>"}}`. Strip the `data:<mime>;base64,` prefix and base64-decode the rest to get usable CSV / XLSX bytes. Save the file somewhere the user can access (e.g. `~/Downloads/<sensible-name>.csv`) and tell them the path. Never paste raw base64 or the data URI back at the user.
6. If `status` is `file_creation_failed`, call `get_report_run` and explain the validation errors from the report run.
7. After saving, give the user a short summary of what's in the file: row count, top movers / largest line items if applicable, and any anomaly that's worth flagging (e.g. a column that drops to zero across the company in one period — could indicate a data issue rather than a real change).

---

## Gotchas

- `poll_report_run_download` is asynchronous for MCP: it returns JSON status, not the file bytes.
- `file_slug` is only available after `status` is `file_creation_completed`.
- Do not create a report run until required filters from `get_report_building_blocks` are set and `preview_report` has shown the expected shape.
- If a required filter points to a domain object, resolve the object slug first. Do not guess slugs from names or dates.
- Do not invent filter field names. Use building blocks and `get_report_filter_schema`, then resolve values with `get_report_filter_values` when available.
- Use `report = data_export_template` and `report run = data_export` in explanations so users understand whether you are talking about the reusable template or one generated file.

---

## Known issues & graceful handling

These are real, observed failure modes of the underlying tools. Handle them transparently — the user should never have to know they exist unless action on their side is genuinely required.

### Required filters from `get_report` / `get_report_building_blocks` are incomplete for variance / comparison reports

**Symptom.** `data_source_config.required_filters` only flags a single toggle filter (e.g. `calculate_variance: true`), but running the report with just that toggle produces empty or meaningless output.

**Cause.** The toggle gates the variance fields; the actual period boundaries are *not* marked required even though the report needs them.

**Handling.**

- For any template whose name or description mentions variance, comparison, period-over-period, year-over-year, etc., always also set **both** period brackets explicitly: `variance_period_1_start`, `variance_period_1_end`, `variance_period_2_start`, `variance_period_2_end` — or the equivalent run-slug pair `variance_payroll_run_1` + `variance_payroll_run_2`.
- Infer the two periods from the user's intent. *"March variance"* → period 1 = February, period 2 = March. *"Compare Q4 to Q3"* → period 1 = Q3, period 2 = Q4.
- If unsure which interpretation the user wants (prior month vs. prior year vs. budget-vs-actual), ask before running — but make the question concrete: *"Compare against the previous month, or the same month last year?"*

### No data exists for the requested period

**Symptom.** User asks for a month or quarter; `list_unified_payroll_runs` for that window returns 0.

**Handling.**

- Check this **before** previewing or running. A zero-data report wastes a roundtrip and confuses the user.
- Pull the actual data coverage (sort `list_unified_payroll_runs` by period and tally by month).
- Tell the user the requested period has no data, show the available coverage as a small table, and recommend the closest sensible alternative period — don't make them guess. Example: *"No March 2026 data — the most recent month with data is January 2026. Want Jan-vs-Dec instead?"*

### Large tool responses get truncated / saved to disk

**Symptom.** Tools like `list_unified_payroll_runs`, `download_file`, or unfiltered `list_*` calls can return responses big enough that the harness saves them to a file instead of returning them inline.

**Handling.**

- Use the `fields` parameter on list endpoints to project only what you need (e.g. `["payroll_runs", "total_count"]` plus a few inner fields). This is the cheapest mitigation.
- For paginated endpoints, page through with `page_size: 50` rather than asking for everything at once.
- When a tool result lands in a file, parse it with `python3 -c "..."` via Bash. Do not paste large file contents into a Read tool — slice by character range or load via `json.load`.
- The user does not care that this happened. Skip narration; just produce the answer.

### `download_file` returns a data URI

**Symptom.** Response is `{"data":{"content":"data:text/csv;base64,<...>"}}`, not raw CSV.

**Handling.**

```python
import json, base64
raw = open(tool_output_path).read()
content = json.loads(raw)['data']['content']
prefix = 'data:text/csv;base64,'   # or data:application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;base64, for xlsx
csv = base64.b64decode(content.split(',', 1)[1]).decode('utf-8')
open('/Users/<user>/Downloads/<name>.csv', 'w').write(csv)
```

Save to `~/Downloads/<sensible-name>.<ext>` and tell the user the path. Never paste the data URI or raw base64 in the response.

---

## Presentation rules

- **Preview**: always a full markdown table. Every row from `preview_report`, every column from the template. One short caption underneath (period, filter summary, row count). No "first N shown" truncation.
- **Downloaded report summary**: after saving the file, give the user (a) the file path, (b) row count, (c) a short table of the most interesting findings — e.g. top 5-10 line items by absolute variance, or totals by category. Flag obvious anomalies (a column that drops to zero everywhere, suspiciously round numbers, missing entities).
- **Slugs**: never print a slug at the user unless they explicitly asked for one or will need it (e.g. *"the report run is saved as `<slug>` if you want to revisit it in the UI"*). Slugs are agent-internal.
- **Internal mechanics**: never narrate base64 decoding, data URIs, tool truncation, retries, or pagination. The user sees a clean answer; the work happens silently.
- **When the user must act**: be one-line and specific — not a paragraph explaining why.

---

## How to Resolve Slugs

- **data_export_template slug** — From `list_reports` or `get_report`.
- **data_export slug** — From `create_report_run`, `list_report_runs`, or `get_report_run`.
- **required filters** — From `get_report_building_blocks` for the selected template.
- **filter field_name** — From `get_report_building_blocks` and `get_report_filter_schema` (`variable_path` or field name shown by the schema).
- **legal_entity_slug** — From `employer_resolve_legal_entity_by_name` when the user gives a name; otherwise from `list_employer_legal_entities`.
- **payroll_run_slug / payroll slug** — From `list_unified_payroll_runs`; use legal entity, country, period, or status filters to narrow the list.
- **file_slug** — From `poll_report_run_download` once `status` is `file_creation_completed`.
