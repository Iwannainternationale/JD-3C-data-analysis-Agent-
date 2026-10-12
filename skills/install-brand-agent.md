---
name: install-brand-agent
description: One-shot installer for the brand-tier-analysis agent (品牌分层分析 agent). Use when a user says "按照安装文档执行" / "安装品牌分析agent" / "setup brand analysis agent" in an environment where the agent is not yet installed.
---

# Install brand-tier-analysis agent

Installs the 4 skills + main prompt of the brand-tier-analysis agent into the CURRENT user's environment. Self-checks environment and permissions first; stops and gives manual instructions for anything that cannot be automated.

## Source of truth

The team repo (ask the user for the URL if unknown): `.claude/skills/` holds the 4 skills, `prompts/brand-tier-analysis-agent.md` holds the main prompt. If no repo URL is available but this skill itself already sits inside the repo, install from the repo's own `.claude/skills/`.

## Procedure

1. **Environment self-check** (report each item pass/fail):
   - `python --version` available (profiling/report scripts need it)
   - `webcli --help` runs (publish skill needs `webcli joyspace` subcommand)
   - joyspace MCP server connected (tools like `aitable.getSchema` / `sheets.getEtWorksheets` visible)
   - SSO login state valid (run `check_sso_token` if available; otherwise instruct the user to log in)
   - Any failure here → do NOT abort silently; continue what is installable, and list manual steps for the rest.
2. **Document permission check** (the skills hardcode two joyspace docs — read access is REQUIRED):
   - 重点品牌名录: aitable `B1Nn3iBEdr4zwfdcP00H` → call `aitable.getSchema` with this page_id
   - 市占工作簿: sheet `V4prlF3VZv16pXY3GTNN` → call `sheets.getEtWorksheets` with this page_id
   - Permission denied / not found → stop and tell the user to request access from the doc owner, do not guess or proceed.
3. **Install skills** (user-level install so they work in any working directory):
   - Copy the 4 skill directories (`profile-brand-category-csv`, `read-keybrand-list`, `read-joyspace-marketshare`, `publish-joyspace-report`) into the user's global skills dir:
     - Windows: `C:/Users/<user>/.claude/skills/`
     - macOS/Linux: `~/.claude/skills/`
   - Do NOT overwrite an existing skill of the same name without telling the user; report version conflict if found.
   - Also copy `prompts/brand-tier-analysis-agent.md` next to the skills (suggest `~/.claude/brand-tier-analysis-agent.md`) so the user can paste it into a new session.
4. **Verify installation**: confirm each of the 4 `SKILL.md` files exists at the target path; print the list.
5. **Report** a checklist: installed items / verified items / failed or manual-action items (MCP config and SSO login can only be checked, never performed, by this installer — say so explicitly).

## Rules

- Never fabricate a successful check; every pass/fail must come from an actual command/tool result.
- If any step fails twice with the same approach, stop and report — do not try creative workarounds.
- This installer does NOT modify the team repo; it only copies FROM it.
