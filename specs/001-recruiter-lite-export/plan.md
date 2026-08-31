# Implementation Plan: LinkedIn Recruiter Lite Export

**Branch**: `001-recruiter-lite-export` | **Date**: 2026-08-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-recruiter-lite-export/spec.md`

## Summary

A Claude Code skill (`linkedin-recruiter-export`) that drives the user's own logged-in Chrome session through the Claude-in-Chrome MCP tools to export one chosen Recruiter Lite project per run: candidates (name, profile URL, headline, company, location, stage, date added), full InMail threads, and notes & tags. The browser phase appends raw JSONL incrementally to `exports/raw/<project-slug>/` (resumable); a Python script (`openpyxl`) then builds one Excel workbook `exports/<project-slug>-<date>.xlsx` with Candidates / Messages / Notes sheets. Hard rules: no hardcoded LinkedIn CSS class selectors, human-paced navigation, immediate stop-and-handoff on any security checkpoint.

## Technical Context

**Language/Version**: Skill procedure = Markdown (Claude Code SKILL.md, executed by the main session). Workbook builder = Python 3.11+ run via `uv run --with openpyxl`.

**Primary Dependencies**: Claude-in-Chrome MCP tools (`tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `computer`, `read_page`, `get_page_text`, `find`, `javascript_tool`, `read_network_requests`); `openpyxl` for the workbook. No other runtime dependencies.

**Storage**: Local files only. Raw layer: JSONL + `state.json` under `exports/raw/<project-slug>/`. Deliverable: `.xlsx` under `exports/`. Both inside this (Drive-synced) project folder per FR-016.

**Testing**: `scripts/build_workbook.py` tested offline against fixture JSONL (`tests/fixtures/`) — run with `uv run --with openpyxl,pytest pytest`. Browser phase validated by the live quickstart run against a small real project (no way to unit-test LinkedIn's UI).

**Target Platform**: macOS, Claude Code main session (only the main session has the Chrome MCP tools — the skill must never delegate browser steps to subagents), Chrome with Claude-in-Chrome extension permitted on linkedin.com.

**Project Type**: Claude Code skill (procedure document + helper script) — not an app or service.

**Performance Goals**: 50-candidate project with threads in < 45 min unattended (SC-003). Pacing floor matters more than speed: 2–5 s randomized delay between navigations, ≤ ~1 page load per 3 s sustained.

**Constraints**: No credential handling; no security-checkpoint automation (stop + handoff); no hardcoded obfuscated CSS selectors (accessibility tree, visible text, href patterns only); everything stays local; never overwrite an existing workbook (time suffix).

**Scale/Scope**: Projects of tens to low hundreds of candidates; single user; one project per run.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is the unfilled Spec Kit template — no ratified project principles exist. No gates to enforce. General defaults applied instead: simplest structure that works (one skill + one script), no speculative abstraction, test what is testable offline. **PASS** (pre-Phase-0 and re-checked post-Phase-1).

## Project Structure

### Documentation (this feature)

```text
specs/001-recruiter-lite-export/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── skill-interface.md
│   ├── raw-data-schemas.md
│   └── workbook-format.md
└── tasks.md             # Phase 2 output (/speckit-tasks — not created here)
```

### Source Code (repository root)

```text
.claude/skills/linkedin-recruiter-export/
├── SKILL.md                     # The skill: frontmatter + operating procedure
├── references/
│   ├── extraction-guide.md      # Per-page extraction recipes (projects list, pipeline, thread, notes)
│   └── safety-rules.md          # Checkpoint detection, pacing, stop conditions
└── scripts/
    └── build_workbook.py        # JSONL raw layer → xlsx workbook (openpyxl)

exports/                         # Created at runtime by the skill
├── raw/<project-slug>/          # state.json + *.jsonl (resume layer)
└── <project-slug>-<date>.xlsx   # Deliverables

tests/
└── fixtures/                    # Sample JSONL + expected workbook shape for build_workbook.py
```

**Structure Decision**: Single skill directory under `.claude/skills/` (Claude Code convention), one helper script, runtime `exports/` tree at repo root. No src/ hierarchy — the "application" is a procedure the model executes plus one build script.

## Complexity Tracking

No constitution violations — table not needed.
