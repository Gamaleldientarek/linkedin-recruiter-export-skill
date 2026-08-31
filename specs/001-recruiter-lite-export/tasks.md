# Tasks: LinkedIn Recruiter Lite Export

**Input**: Design documents from `/specs/001-recruiter-lite-export/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md (all present)

**Tests**: Included ONLY for the offline workbook builder (plan.md mandates fixture-based pytest for it). The browser phase is validated live per quickstart.md — no automated tests possible against LinkedIn's UI.

**Organization**: Tasks grouped by user story. US1 = candidate roster (P1, MVP), US2 = message threads (P2), US4 = resume (P2), US3 = notes & tags (P3). Tasks marked **(live)** need the user present with Chrome logged into Recruiter Lite; all browser steps run in the main session only.

## Format: `[ID] [P?] [Story] Description`

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Skill skeleton and test scaffolding

- [ ] T001 Create skill directory tree: `.claude/skills/linkedin-recruiter-export/` with empty `references/` and `scripts/`, plus `tests/fixtures/sample-project/` at repo root
- [ ] T002 [P] Create `tests/fixtures/sample-project/` fixture data per contracts/raw-data-schemas.md: `candidates.jsonl` (4 candidates incl. one Arabic-named, one with null fields), `messages.jsonl` (threads for 2 candidates incl. one `scheduled`), `notes.jsonl` (notes + tags for 2 candidates), `state.json` (status `complete`) — one candidate must have zero messages and zero notes
- [ ] T003 [P] Write `references/safety-rules.md` in `.claude/skills/linkedin-recruiter-export/references/`: checkpoint detection signals (URL `/checkpoint/`, verification/CAPTCHA/unusual-activity page text), absolute stop-and-handoff rule, pacing spec (randomized 2–5 s between navigations, 8–12 s rest every ~10 candidates), logged-out detection, per research.md R7

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The workbook builder and the skill's core frame — every story depends on these

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 Implement `scripts/build_workbook.py` in `.claude/skills/linkedin-recruiter-export/scripts/` per contracts/workbook-format.md: CLI (`<raw-dir> [--out]`), JSONL read with dedupe-on-read rules from contracts/raw-data-schemas.md, three sheets with exact columns, frozen bold headers, wrapping, hyperlinked profile URLs, message/note counts on Candidates sheet, never-overwrite `-HHMMSS` suffix, named non-zero-exit errors (missing files, invalid candidate, unknown profile_url in messages/notes)
- [ ] T005 Write `tests/test_build_workbook.py` covering: sheet/column layout matches contract, dedupe keeps last occurrence, zero-message candidate absent from Messages sheet, Arabic text round-trips, empty project → headers only, overwrite protection creates suffixed file, unknown-profile_url message fails with named error — run `uv run --with openpyxl,pytest pytest tests/` until green (fixtures from T002)
- [ ] T006 Write `SKILL.md` core frame in `.claude/skills/linkedin-recruiter-export/`: frontmatter (name `linkedin-recruiter-export`, trigger-rich description per contracts/skill-interface.md), hard rules section (no credentials, checkpoint stop, no CSS selectors, pacing, local-only, main-session-only browser work), precondition checks (ToolSearch-load Chrome MCP tools, tab context, logged-in verification at `linkedin.com/talent/home`), state.json read/write procedure (atomic rewrite, schema from data-model.md), run-mode decision table (new / resume / complete→ask fresh-vs-incremental per FR-015), and end-of-run summary format (FR-013)

**Checkpoint**: `uv run --with openpyxl scripts/build_workbook.py tests/fixtures/sample-project --out /tmp/sample.xlsx` succeeds (quickstart Scenario 1); SKILL.md frame reviewed

---

## Phase 3: User Story 1 — Export a project's candidate list (P1) 🎯 MVP

**Goal**: Pick a project, walk the full pipeline, produce a workbook whose Candidates sheet matches Recruiter Lite exactly

**Independent Test**: quickstart Scenario 2 steps 1–3 against a small real project; verify SC-001 (every candidate exactly once, working links)

- [ ] T007 [US1] **(live)** Recon run: with the user's Chrome, navigate Recruiter Lite (`/talent/home` → Projects → a project pipeline), capture accessibility-tree shape via `read_page`/`get_page_text` for: projects list, pipeline candidate rows (name, headline, company, location, stage, date added), pagination/lazy-load behavior, and how candidate profile hrefs appear — record findings as semantic waypoints (no class names) in `references/extraction-guide.md` sections "Projects list" and "Pipeline roster"
- [ ] T008 [US1] Write the project-selection procedure into `SKILL.md`: list projects by name from the Projects page, match optional argument, ask the user to pick otherwise (contracts/skill-interface.md interaction point 1)
- [ ] T009 [US1] Write the roster-extraction loop into `SKILL.md` + `references/extraction-guide.md`: scroll/paginate → extract candidate fields → dedupe by profile URL → append `candidates.jsonl` → mark roster progress in `state.json`; completion condition = full pass with no new unique candidates and no next-page control (FR-008, spec pagination edge case); blank-not-guessed rule for missing fields; loud stop naming missing structure (FR-012)
- [ ] T010 [US1] **(live)** Validate US1 end-to-end on a small real project: run the skill, then `build_workbook.py`, verify Candidates sheet against the on-screen pipeline (SC-001) and fix extraction-guide waypoints until it passes

**Checkpoint**: MVP — a real project's roster exports correctly to a workbook

---

## Phase 4: User Story 2 — Full message history per candidate (P2)

**Goal**: Every candidate's complete InMail thread in the Messages sheet, inbox-state independent

**Independent Test**: quickstart Scenario 2 message spot-check — 3 threads match message-for-message with correct directions (SC-002)

- [ ] T011 [US2] **(live)** Recon a candidate conversation view from within the project (thread container, per-message sender/date/text, scheduled-message appearance, no-thread state) and record semantic waypoints in `references/extraction-guide.md` section "Message thread"
- [ ] T012 [US2] Write the per-candidate message procedure into `SKILL.md` + `references/extraction-guide.md`: open thread from candidate, extract each message (seq, date, direction from sender identity, subject, full text; `scheduled` label per FR-004), append `messages.jsonl`, mark `messages: done|none` in `state.json`, zero rows for no-thread candidates (US2 acceptance 2), pacing between candidates per safety-rules
- [ ] T013 [US2] **(live)** Validate US2 on the test project: full run, spot-check 3 known threads incl. one archived conversation (inbox-state independence) against the workbook Messages sheet (SC-002)

**Checkpoint**: Roster + complete message history export together

---

## Phase 5: User Story 4 — Resume an interrupted run (P2)

**Goal**: Interruptions (tab closed, checkpoint, Esc) never cost completed work

**Independent Test**: quickstart Scenario 3 — interrupt at ~50%, re-invoke, only pending candidates processed, final coverage equals uninterrupted run (SC-004)

- [ ] T014 [US4] Write resume behavior into `SKILL.md`: on invoke with existing `state.json` status ≠ `complete` → report progress counts, continue only `pending` work (FR-007); checkpoint stop path records `stopped_checkpoint` + reason, waits for explicit user resume (FR-010, safety-rules); completed-project path asks fresh (archive raw dir to `raw/<slug>-archived-<timestamp>/`) vs incremental (only roster members absent from state) per FR-015
- [ ] T015 [US4] **(live)** Validate quickstart Scenario 3 (interrupt/resume) and Scenario 4 (fresh-vs-incremental on the completed project); confirm no duplicate rows beyond dedupe rules and old workbook untouched

**Checkpoint**: Long runs are interruption-safe

---

## Phase 6: User Story 3 — Notes and tags per candidate (P3)

**Goal**: Every note and tag attributed to the right candidate in the Notes sheet

**Independent Test**: quickstart Scenario 2 notes check — known notes/tags appear correctly attributed

- [ ] T016 [US3] **(live)** Recon the notes/tags surface on a candidate detail view and record semantic waypoints in `references/extraction-guide.md` section "Notes & tags"
- [ ] T017 [US3] Write the notes/tags procedure into `SKILL.md` + `references/extraction-guide.md`: extract note text/author/date and tag labels (author/date only where visible, FR-005), append `notes.jsonl`, mark `notes: done|none` in `state.json`, zero rows for candidates without notes (US3 acceptance 2)
- [ ] T018 [US3] **(live)** Validate notes/tags on the test project against the workbook Notes sheet

**Checkpoint**: All four stories functional

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T019 [P] Verify end-of-run summary output (FR-013) and empty-project behavior (quickstart Scenario 6) — adjust SKILL.md wording if the summary misses counts or skip reasons
- [ ] T020 [P] Write repo `README.md`: what the skill does, prerequisites, invocation, safety posture, workbook format pointer to contracts — no candidate data examples. Include an **Installation** section: (a) Claude Code — clone repo and copy `.claude/skills/linkedin-recruiter-export/` into `~/.claude/skills/` (global) or a project's `.claude/skills/` (per-project), enable Claude-in-Chrome, restart session; (b) ChatGPT — honest note that Claude skills don't run on ChatGPT natively, with a short "adapt manually" pointer (SKILL.md + references are plain-Markdown procedure text usable as instructions for any agent that controls the user's browser)
- [ ] T021 Self-review the finished skill against `references/safety-rules.md` and contracts/skill-interface.md hard guarantees (SC-005 rule inspection from quickstart Scenario 5); fix any drift
- [ ] T022 Run full quickstart validation pass (Scenarios 1–4, 6) and record results in `specs/001-recruiter-lite-export/quickstart.md` as a dated validation log appendix
- [ ] T023 Commit and push all work to `origin/main` (github.com/Gamaleldientarek/linkedin-recruiter-export-skill), confirming `exports/` stays ignored

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none — start immediately; T002 and T003 parallel after T001
- **Foundational (Phase 2)**: needs Phase 1 (T004/T005 need T002 fixtures; T006 needs T003 safety rules). BLOCKS all stories
- **US1 (Phase 3)**: needs Phase 2. T007 → T008/T009 → T010
- **US2 (Phase 4)**: needs US1 roster loop (iterates the roster). T011 → T012 → T013
- **US4 (Phase 5)**: needs state.json usage from US1/US2 to be meaningful. T014 → T015
- **US3 (Phase 6)**: needs US1 candidate iteration. T016 → T017 → T018
- **Polish (Phase 7)**: needs all desired stories; T019/T020 parallel; T021 → T022 → T023

### Story order note

US2 and US4 are both P2; US2 runs first because US4's resume test is only meaningful once runs are long (roster + threads). US3 (P3) last. Live tasks (T007, T010, T011, T013, T015, T016, T018) require the user present and are naturally serial — the browser is a single shared resource, so cross-story parallelism does not apply to live work.

### Parallel Opportunities

- T002 ∥ T003 (fixtures vs safety rules)
- T004+T005 ∥ T006 (Python builder vs SKILL.md frame — different files)
- T019 ∥ T020 in Polish

## Implementation Strategy

**MVP first**: Phases 1–3 deliver a working roster export (US1) — already a usable deliverable. Validate with the real small project, then layer US2 (threads), US4 (resume), US3 (notes) incrementally, each with its own live checkpoint. Commit after each phase checkpoint; push at T023 (or per-phase if preferred).
