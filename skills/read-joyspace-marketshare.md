---
name: read-joyspace-marketshare
description: Use when a task requires weekly market-share (市占率) data from the joyspace sheet https://joyspace.jd.com/sheets/V4prlF3VZv16pXY3GTNN — e.g. judging "three consecutive weeks of share decline", a YTD trend, or any specified week window (month / H1 / custom range) for brands or categories. Triggers on any read of this market-share workbook, so its fixed layout is used instead of re-probing the sheet structure.
---

# Read joyspace market-share workbook (minimal-read map)

The workbook V4prlF3VZv16pXY3GTNN has a stable layout: each channel occupies a vertical block of rows; JD block is at the top; brands are columns; weeks are rows. Row/column COUNTS DO DRIFT over time (new brand → new column appended after the last brand column; new week → new row appended below the previous last week). The map below gives coordinates by RULE, not by frozen numbers — recompute the two anchors each run.

## Week ↔ calendar conversion (verified = ISO week system)

- **Wn's Monday = 2025-12-29 + (n−1)×7 days**; a week runs Monday..Sunday. (2025-12-29 is ISO-2026-W1 Monday; 2026-08-31 = W36 Monday — both verified.)
- Today → theoretical latest week n = ((today − 2025-12-29).days // 7) + 1 (use Monday of the current ISO week if mid-week).
- Use for: annotating report week windows with calendar ranges (e.g. "W36 = 2026-08-31~09-06"), translating user dates ("9月第一周" → W36) into week numbers, and predicting the latest filled row before reading.

## Sheets

| sheet | purpose | read? |
|---|---|---|
| 总表 | brand-level weekly market share | YES (JD block only) |
| 类目 | category-level weekly market share | only if per-category context is needed |
| 自营 / POP / 市占可添加品牌名单 / 类目底表 / 1-30自营·市占·GMV (hidden) | unrelated | NO |

Get sheetIds once via `sheets.getEtWorksheets` if not known: 总表 and 类目 were index 0 and 3 (sheetId 22 / 21) as of 2026-09; verify by sheet name, not index, if the workbook is restructured.

## 总表 (brand) block layout — JD block

- Row 0: channel name 「JD」. Rows 1-3 (对TMB / 对DY / 周环比): channel-level aggregates — NOT used for brand judgment, skip.
- Row 4: **brand header row** — brand names start at col 2, one per column, contiguous to the last brand column (last col grows as brands are added). Duplicate brand names may exist (e.g. KODAK twice); take the first occurrence.
- Row 5: blank.
- Row 6 onward: weekly data. col 0 = 「2026年第N周」, col 1 = 「WN」, col 2..last = that week's share per brand (decimal 0-1; originalCellValue and cellText %-string agree).
- **Anchor rule: week Wn sits at row 5+n** (W1 = row 6). The latest week is the last non-empty row in the JD block — JD block ends before the next channel block (a row whose col 0 is a channel/section title like 「A-TM(大贸+国内现货)」, not a 「2026年第N周」 row).

## 类目 (category) sheet layout

Same shape: row 4 = category header (col 2 onward: 数码相框, 闪光灯/手柄, 相机配件, 手持稳定器, 三脚架/云台, 相机包, 滤镜, 读卡器, 相机清洁/贴膜, 镜头附件, 拍照配件, 数码支架, 创意配件 — grows the same way), row 6+ weekly data, Wn = row 5+n. Category names are close to but not always identical with source-CSV category names (e.g. 闪光灯/手柄 vs 闪光灯手柄) — match with tolerance.

## Read procedure — parameterized week window

The task specifies a week window: default = latest 3 weeks; other shapes (a month, H1, YTD, custom range) are all the same formula with different parameters.

1. (Skip if sheetIds known) `sheets.getEtWorksheets` — find 总表 / 类目 by name.
2. Read 总表 row 4, col 2..~110 → brand header list (extend the range if the read comes back full-width — the last brand column drifts).
3. **Compute-then-verify anchor (saves a probe call):** convert today to the theoretical latest week n via the calendar formula; read col 0-1 at row 5+n only — expect 「Wn」. Mismatch (sheet not yet updated / lagging) → read a small window of rows around it (e.g. rows 5+n−4 .. 5+n+2) and take the last actual 「Wk」 row; only if that also fails, scan col 0 until the next channel title.
4. **Read the week window as ONE rectangle:** rows (5+W_first)..(5+W_last), col 2..last, single `getEtRangeData` call. This covers latest-3-weeks (one 3-row rectangle), a month, H1, or YTD (W1..W_last) — a CONTIGUOUS window is itself the minimal read; the old "never bridge" rule applies only to NON-adjacent ranges (e.g. header row + week rows = 2 calls).
   - **Verified safe:** a YTD full rectangle (36 weeks × 108 brands = 3,888 cells, ~1.2MB response) returned complete with no truncation (2026-09-15). One transient timeout on first attempt — retry once, this matches the known flaky-timeout behavior of the API regardless of range size.
   - Fallback ONLY on error/short response: split into ≤1,500-cell blocks and re-read; the response is persisted to a tool-results file either way — parse it with a local script, never in-context.
   - Parse into a local cache (e.g. `_ms_ytd_<N>w.json` for YTD): brand → {W1..W_last}. Any later week-window request slices this cache locally with ZERO extra calls.
5. Consecutive-decline judgment (for 3-week windows): W(n-2) > W(n-1) > Wn (strict). For longer windows, judge per the task's definition (e.g. first-vs-last, or every-week strict decline) — ask once if the task is ambiguous.
6. If per-category context is needed, read 类目 with the SAME window (2 calls): (a) row 4, col 2..last → category headers; (b) rows (5+W_first)..(5+W_last), col 2..last. **REUSE the week anchor confirmed on 总表** — do not re-probe unless the category read comes back inconsistent.

## Rules

- Never read rows 1-3 aggregates, non-JD channel blocks, or hidden sheets. Reading W1..W(n-3) history is ALLOWED when the task's window demands it (YTD/H1) — it is forbidden only as idle probing beyond the requested window.
- Non-adjacent ranges (header row vs week rows) must be separate calls; a contiguous week window is one rectangle. When in doubt, prefer fewer, larger contiguous calls over bridges across unrelated rows.
- Brand names in this table may differ in case/parentheses from source data (Insta360 vs 影石Insta360, LLANO vs llano) — normalized matching is handled upstream (see skill `profile-brand-category-csv`); unresolvable near-matches go to the user.
- Empty cells in the weekly grid are business-true (brand not yet listed that week, e.g. ARZOPA/Divoom empty W1-W19 of 2026) — treat as "no data", never as zero share.
- If coordinates drift beyond the anchor rules (row 4 no longer a brand header, week rows no longer at 5+n), re-probe the block structure once, then UPDATE this skill file to the new rule.
