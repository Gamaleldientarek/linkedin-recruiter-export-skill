# Quickstart: Validating the LinkedIn Recruiter Lite Export

Runnable scenarios proving the feature end-to-end. Contracts: [skill-interface](./contracts/skill-interface.md), [raw-data-schemas](./contracts/raw-data-schemas.md), [workbook-format](./contracts/workbook-format.md).

## Prerequisites

- Chrome open with the Claude-in-Chrome extension installed and permitted on `linkedin.com`
- Logged into LinkedIn with the Recruiter Lite account
- `uv` installed (`command -v uv`)
- A small real Recruiter Lite project (≈5–20 candidates) to use as the test subject

## Scenario 1 — Offline: workbook builder (no browser)

```bash
uv run --with openpyxl scripts/build_workbook.py tests/fixtures/sample-project --out /tmp/sample.xlsx
```

**Expected**: exit 0; prints output path + per-sheet row counts matching the fixture; opening the file shows 3 sheets per the workbook contract, Arabic fixture strings intact, zero-message candidate absent from Messages sheet. Re-running the same command does NOT overwrite — a `-HHMMSS` file appears.

```bash
uv run --with openpyxl,pytest pytest tests/
```

**Expected**: all tests pass (schema validation, dedupe-on-read, overwrite protection, unknown-profile_url failure).

## Scenario 2 — Live: small project, full export (US1+US2+US3)

In a Claude Code session in this project:

1. Invoke `/linkedin-recruiter-export`.
2. Confirm the skill lists your projects by name and asks you to pick — pick the small test project.
3. Let it run unattended. Watch that page moves are visibly paced (seconds apart, not machine-gun).

**Expected**: final summary reports project name, candidate/message/note counts, and the workbook path `exports/<slug>-<date>.xlsx`. Verify against Recruiter Lite manually:
- every pipeline candidate present exactly once, profile links open the right people (SC-001)
- 3 spot-checked threads match message-for-message with correct directions (SC-002)
- notes/tags attributed to the right candidates
- an "Archived"-state conversation still appears in Messages (inbox-state independence)

## Scenario 3 — Live: interrupt and resume (US4)

1. Start Scenario 2 on the test project; when roughly half the candidates are done, close the work tab (or hit Esc to interrupt).
2. Re-invoke `/linkedin-recruiter-export`, pick the same project.

**Expected**: skill reports N candidates already done and continues with the rest only (`state.json` shows `done` entries untouched, no duplicate JSONL bloat beyond the dedupe rules). Final workbook coverage equals Scenario 2's (SC-004).

## Scenario 4 — Live: completed-project re-run (FR-015)

Re-invoke on the project completed in Scenario 2/3.

**Expected**: skill asks **fresh vs incremental**. Fresh → old raw dir archived (`raw/<slug>-archived-…/`), full re-walk, new workbook with time suffix, old workbook untouched. Incremental → only candidates added to the project since last run are visited.

## Scenario 5 — Safety: checkpoint handoff (FR-010 / SC-005)

Cannot be forced deliberately — validated by rule inspection plus behavior if it ever occurs naturally.

**Expected on occurrence**: the run stops immediately, names the page it saw, performs zero interactions with it, records `stopped_checkpoint` in `state.json`, and waits for the user; after manual resolution, resume continues per Scenario 3.

**Rule inspection**: `references/safety-rules.md` must define the checkpoint detection signals and the absolute stop rule; `SKILL.md` must place it in the always-loaded hard rules.

## Scenario 6 — Empty project edge case

Run against an empty project (create one if needed).

**Expected**: no error; workbook with headers only; summary says 0 candidates.
