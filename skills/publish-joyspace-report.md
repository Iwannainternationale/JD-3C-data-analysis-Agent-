---
name: publish-joyspace-report
description: Use when an analysis result must be delivered as a joyspace online document — the task says "记录到/发布到/生成 joyspace 文档", or a finished local Markdown report needs to go online, or headings/tables in a report must render as native document blocks rather than plain text.
---

# Publish report to a NEW joyspace document

Deliver completed analysis reports by creating a NEW joyspace online document in ONE shot. Do NOT write into an existing document (no appending, no segment-by-segment edits) — segmented `edit` appends lose content under async-save races and degrade headings/tables to plain-text paragraphs.

## Procedure

1. **Compose locally, completely.** Generate the full report as one local Markdown file (`_report_final.md`). It must already contain the final heading hierarchy (`#`/`##`/`###`) and Markdown tables. Use f-strings / `.format()` in generator scripts — never `%`-formatting with Chinese text (percent-escape bugs).
2. **Publish once** with the o2 webcli upload path (works where the joyspace MCP lacks a body-update action):
   ```bash
   PYTHONIOENCODING=utf-8 webcli joyspace upload <local.md> --title "<报告标题>" [-f yaml]
   ```
   - `PYTHONIOENCODING=utf-8` is REQUIRED on Windows (default GBK console breaks the CLI).
   - The first `# H1` in the file becomes the doc title and is removed from the body; pass `--title` to control it explicitly.
   - Space/folder placement: omit → personal space; `--space <id|url>` (+ `--folder-id`) → target location.
3. **Verify by reading back the FULL body via ONE canonical path** — `webcli joyspace view` (any format) returns only summary fields (title/author/read_count) and can NEVER verify the body; do not probe with it:
   ```bash
   PYTHONIOENCODING=utf-8 webcli joyspace export <returned-doc-id> --output <local_readback.md>
   ```
   (Equivalent: joyspace MCP `joyspace.get_normal_markdown_content` with page_id.) Compare the export against the local file: every module heading present; per-module record counts match local stats; spot-check one table row.
   **Empty/over-short readback ≠ immediate failure**: the CLI reports success when the doc shell is created, but the body may still be ingesting. Wait ~10s and re-read ONCE; a second empty readback is a confirmed empty-body failure.
4. **First confirmed empty-body failure → re-upload the UNCHANGED file once** (transient cover; most single failures pass this), verify per step 3. Success → deliver.
5. **Two consecutive confirmed empty-body failures → Desktop-fallback protocol.** This is a KNOWN intermittent webcli upload-service fault, content-independent (observed 2026-09-14: same file failed 4 consecutive uploads then succeeded unchanged; a 61KB file succeeded in the same window; failed docs show `updated_at == created_at` — body injection never happened). Do NOT bisect the content, do NOT probe-upload to diagnose — the uploads themselves are what is failing.
   1. **Desktop copy FIRST (guaranteed deliverable):** copy the local MD to the user's Desktop with a sanitized timestamped name (`<标题去掉 \/:*?"<>| 等非法字符>_<YYYYMMDD_HHMM>.md`). Resolve Desktop via `powershell -Command "[Environment]::GetFolderPath('Desktop')"` (handles OneDrive-redirected desktops), fallback `$USERPROFILE/Desktop`. Tell the user the path immediately.
   2. **Continue retries in the background (up to 3, spaced ≥60s):** loop `upload → sleep ~12s → export → compare`, delete each failed doc immediately, sleep 60-120s between attempts. Run the loop as a background task so the conversation is not blocked; notify the user when it lands or exhausts. Do NOT hammer — dense retries may prolong a fault window.
   3. **Success** → deliver the URL (mention the desktop copy as archive). **All retries exhausted** → stop, deliver the desktop file as the deliverable, and offer options: retry later / alternate path (joyspace MCP `create_doc_routing`).
   4. **Optional diagnosis while waiting:** on a failed doc, `get_page_basic_info` — `updated_at == created_at` confirms the injection-race signature; re-run one upload with `-v --trace retain-on-failure` to capture automation internals, and consider reporting the trace to the webcli/o2 owners.
6. **Content-mismatch failure** (body present but modules/counts wrong) is a DIFFERENT failure: fix the LOCAL file, re-upload once; second failure → stop and ask the user. No desktop fallback needed — the local file is already the deliverable source.
7. **Deliver** the URL (or desktop path per step 5) to the user.

## Rules

- One report = one upload call per attempt; additional attempts only via steps 4-5. Never silently fall back to segmented appends — if the CLI rejects the file for size, ask the user about splitting into linked docs.
- Empty-body failure is service-side and content-independent: never bisect content, never content-probe to diagnose it.
- Writing into an existing document is allowed ONLY for a short pointer (≤ a few hundred characters: summary + link to the new report doc), and only when the task explicitly requires a trace in that document.
- Delete every failed/probe document immediately after its diagnostic purpose ends; leave no residue.
