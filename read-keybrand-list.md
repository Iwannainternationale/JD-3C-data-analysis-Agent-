---
name: read-keybrand-list
description: Use when a task needs the key-brand list (重点品牌名录) as a partition basis — the joyspace aitable at https://joyspace.jd.com/table/B1Nn3iBEdr4zwfdcP00H. Triggers on any read of this brand list table, so its pagination quirks are handled without trial-and-error.
---

# Read key-brand list (joyspace aitable)

The table B1Nn3iBEdr4zwfdcP00H ("重点品牌list") is a joyspace aitable (多维表格, pageType 21) holding the authoritative key-brand roster. Brand count drifts upward over time (was 107 as of 2026-09); always read ALL records, never reuse a remembered count.

## Read procedure

1. `aitable.getSchema` with page_id = B1Nn3iBEdr4zwfdcP00H → get the data sheet id and recordsCount (sheet name 品牌LIST, sheetId was "5"; field 品牌名称, type MultiLineText). Use the returned recordsCount to size pagination.
2. Paginate with `aitable.listRecordsByPage`:
   - **Pagination parameter: `page_num` + `page_size` WORKS; `start` / `offset` DO NOT** (they are silently ignored — the API returns the first page again). Do not burn calls retrying start/offset.
   - Recommended: `page_size` = 20 (proven safe; larger values also return up to 100) and walk `page_num` = 1..⌈N/20⌉; or try a single large `page_size` (e.g. = recordsCount) and verify the returned record count equals recordsCount, falling back to page_num walk if short.
3. Each record's brand name is in `fields` JSON → `品牌名称.text`. Collect in order; dedupe if the roster ever repeats a name.
4. Verify completeness: total collected == recordsCount from schema. If a page returns empty while more records are expected, the page_size window shifted — re-read that page.

## Rules

- The roster is the ONLY partition basis for key/secondary brand classification (per task prompt). Do not add or remove brands by judgment; near-miss names go to the alias-matching step (skill `profile-brand-category-csv`) and then to the user.
- If the table grows past ~1000 records and page_num walking gets slow, cache the full list to a local JSON (`_keybrand_<N>.json`) with the recordsCount stamped, and on later runs spot-check freshness (recordsCount from schema vs stamped count) before reuse.
- Table access failure (no permission / deleted / sheet missing) → stop and ask the user; do not guess the roster.
