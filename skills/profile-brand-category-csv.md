---
name: profile-brand-category-csv
description: Use when a user uploads a brand/category business CSV or Excel (.xlsx) for brand-performance analysis (e.g. GMV YoY screening, negative-growth classification, brand+category unit analysis) and the column layout or data conventions may vary between files. Handles xlsx multi-sheet (data sheet found by content signature, not position) and CSV encoding dispatch. Triggers on any first-pass data exploration of such a source file before classification questions are asked.
---

# Profile brand-category CSV/XLSX (one-shot data profiling)

Run this on ANY newly uploaded brand/category business data file BEFORE doing classification, asking the user questions, or reading external config tables. Goal: turn open-ended data exploration into one deterministic script run that surfaces every口径 (data-convention) question at once.

## Why

Fragmented exploration (checking duplicates, then nulls, then edge values, one doubt per tool call) wastes 8+ rounds before all questions are assembled. One profiling pass produces the full question list in a single run.

## Procedure

1. Write a local profiling script (Python, `_profile.py`) against the file. **Input dispatch by extension** (verified 2026-09-15 on two real exports — 运营 and 采控 permission scopes of the same dataset, both parsed identically and cross-validated 295/295 units against the CSV export of the same data):
   - `.csv`: detect encoding first (`utf-8-sig`, fallback `gbk`), then `csv.reader`.
   - `.xlsx`: `openpyxl.load_workbook(path, data_only=True)` — **NORMAL mode, NOT `read_only=True`**: this export family stores no cached dimension, and read_only mode returns a phantom 1×1 sheet (looks like an empty file). Numeric cells arrive as native floats — skip string parsing for them.
   - **Multi-sheet content-signature detection** (the first sheet is often a 基础信息/筛选条件 page, but position is NOT guaranteed): for each sheet read row 1; the data sheet is the one whose first row contains ALL of `主品牌` + `三级类目` + `Net GMV`. Exactly one hit → use it, zero questions asked. Zero hits → stop and ask the user. ≥2 hits → list candidates and ask (one batch). Meta sheets (筛选条件 etc.) never carry the signature and are skipped automatically.
   - Script preamble for ALL dispatch paths (GBK-console mojibake immunity, verified): `sys.stdout.reconfigure(encoding='utf-8')` at entry; write Chinese findings to the report FILE, print only ASCII status lines.
   - Never assume column positions — map by the header contract below.
   **Header contract (this export family): header NAMES are invariant; only column ORDER and the presence of per-metric aux columns vary.**
   - Identity columns (exact names, transparent to attribution): `主品牌`, `主品牌ID`, `三级类目`, `三级类目ID`.
   - Metric anchors (exact names): `Net GMV`, `综合毛利率（不含返利）`, `综合毛利额（不含返利）`, `销售毛利率`, `销售毛利额`, `广告毛利率`, `广告毛利额`; each may carry a `<metric>_同比` companion column. Required for profiling: `Net GMV` + `Net GMV_同比` (the others are context).
   - Aux columns `占比`/`贡献` (0, 1, or 2 per metric, either order): attribute each to the nearest metric anchor on its LEFT, skipping identity columns. The `占比` owned by `Net GMV` is the sales-share column for the report.
   - Any column outside these sets is UNEXPECTED (e.g. newly added, non-required content): treat it as an attribution barrier — aux columns to its right become orphans and must be flagged, never silently attributed to a known metric. List unexpected columns in the Structure section.
   - Structural anomalies to flag: orphan aux (no metric on its left); same aux type attributed twice to one metric; required columns missing.
2. Run it and produce `_profile_report.txt` with these sections (skip a section with a note if the column it needs is absent — do NOT fabricate):
   1. **Structure**: row/col count, headers, sort order; count of (brand+category) combos, unique combos.
   2. **Field completeness**: rows with empty YoY (grouped by whether GMV==0), missing/non-numeric GMV rows.
   3. **Edge-value distribution** (the source of口径 questions): YoY == -1 exactly (candidate anomaly zone); YoY < -1 (negative GMV — refunds? ask how to treat); GMV == 0 with YoY != -1 (incl. empty YoY and YoY == +1 — opportunity-zone口径 confirmation); rows with GMV < 0, listed in full.
   4. **Duplicate combos**: same brand+category appearing in multiple rows (different brand IDs = different shop entities), with per-row values shown.
   5. **Alias suspects**: normalize names (strip parentheses content, casefold) and match against the reference lists the task will use (key-brand list / market-share table); list near-miss-but-not-identical pairs.
   6. **Category inventory**: all category names with total GMV and computed YoY, for cross-checking against external market-share category names (note near-matches like 闪光灯/手柄 vs 闪光灯手柄).
   7. **Number formats**: YoY/ratio as decimal vs percent string; money units; thousands separators.
   8. **Cache validation** (only if user opted to reuse prior artifacts): spot-check 2-3 values of cached key-brand list / market-share data against fresh reads.
3. Convert sections 3/4/5 findings into ONE AskUserQuestion batch (max 4 questions). Do not ask anything the source file itself answers.

## Rules

- All questions to the user come AFTER the full profile run, in one batch.
- Report raw numbers exactly as found; never round or merge duplicate rows silently — ask.
- If header names themselves change (beyond the contract above), stop and ask the user for the column mapping before proceeding — order changes alone must NEVER trigger this.
- Do NOT convert xlsx to csv as a preprocessing step — direct openpyxl read is one step, lossless (multi-sheet preserved), and avoids an encoding probe; conversion adds a call and can drop sheets.
- Keep the script; it is reusable for the same task family. The header-contract mapping in step 1 makes column reordering and missing/extra aux columns zero-touch, and the input dispatch makes csv/xlsx zero-touch.

## Cross-reference

- Market-share table layout: see skill `read-joyspace-marketshare`.
- Task-level conventions (new-doc publishing, cache reuse): usually set by the task prompt or user memory.
