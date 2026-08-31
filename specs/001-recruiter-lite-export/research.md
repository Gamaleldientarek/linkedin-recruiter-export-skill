# Phase 0 Research: LinkedIn Recruiter Lite Export

All Technical Context unknowns resolved. Each decision below: Decision / Rationale / Alternatives considered.

## R1. Browser automation surface

**Decision**: Claude-in-Chrome MCP tools driven by the **main Claude Code session** (never subagents). Tool roles: `tabs_context_mcp` → session start; `tabs_create_mcp` → dedicated work tab; `navigate` → URL moves; `read_page` (accessibility tree) → structured extraction of lists/threads; `get_page_text` → bulk text capture; `find` → semantic element location; `javascript_tool` → harvest profile hrefs by URL pattern and scroll containers; `computer` → clicks/scrolls where semantics fail; `read_network_requests` → diagnostic only.

**Rationale**: The extension runs inside the user's real logged-in Chrome — no credentials touched (FR-001), no separate automation fingerprint (Playwright/Puppeteer would need a fresh login and trips LinkedIn's automation detection far harder). Only the main session holds these MCP tools.

**Alternatives considered**: Playwright/Puppeteer (separate browser profile, login handling, much higher detection risk — rejected); LinkedIn official APIs (Recruiter System Connect etc. are enterprise-contract only, not available to Lite — rejected); manual copy-paste (the problem being solved — rejected).

## R2. Extraction strategy without CSS selectors (FR-011)

**Decision**: Three stable signal layers, in priority order:
1. **Href patterns** — candidate identity from anchor URLs (`/talent/profile/…` or `/in/…`); the public profile URL is the dedupe key.
2. **Accessibility tree** (`read_page`) — roles, labels, and text structure survive LinkedIn's class-name churn because they feed screen readers.
3. **Visible text + landmarks** (`get_page_text`, `find` with semantic descriptions) — for fields (headline, location, stage) and thread content.

Obfuscated class names (e.g. `artdeco-…`, hashed CSS modules) are never referenced. When an expected structure can't be located, fail loudly naming what's missing (FR-012).

**Rationale**: LinkedIn ships obfuscated, frequently-rotated class names; accessibility semantics and URL shapes are the only stable contract.

**Alternatives considered**: Hardcoded CSS/XPath selectors (breaks silently on every UI push — rejected); intercepting Voyager API JSON via network capture (richer data but brittle, undocumented, and behaviorally closer to scraping infrastructure than reading the visible page — rejected for v1, noted as diagnostic aid only).

## R3. Recruiter Lite entry points

**Decision**: Start at `https://www.linkedin.com/talent/home`; projects list at the Projects navigation item; each project exposes its pipeline (stages such as Uncontacted / Contacted / Replied) as candidate lists; each candidate row opens a profile slide-in/detail with tabs or sections for Messages and Notes. Exact in-page geometry is confirmed live during the first recon run (Task phase includes a recon task) — the skill's extraction guide describes *what to look for semantically*, not pixel positions.

**Rationale**: Lite's UI differs from full Recruiter and changes; the skill documents semantic waypoints and verifies them at run time rather than baking in a frozen map.

**Alternatives considered**: Fully pre-mapping the UI in the skill (goes stale — rejected); no guide at all, improvising each run (wastes tokens and risks inconsistent output — rejected).

## R4. Message capture semantics

**Decision**: Per-candidate thread capture from the candidate's conversation view — independent of inbox filter tabs (Inbox / Awaiting Reply / Scheduled / Archived all surface the same underlying threads). Direction inferred per message from sender identity (user's own name → `sent`; candidate name → `received`); scheduled-but-unsent messages captured with status `scheduled` when visible. Candidates with no thread produce zero message rows, no error.

**Rationale**: Iterating the project roster guarantees coverage aligned to the chosen project (SC-002) and makes inbox state irrelevant; folder-walking the inbox would miss nothing extra for project candidates but would add non-project conversations out of scope.

**Alternatives considered**: Walking inbox folders (scope creep, duplicates — rejected).

## R5. Raw data layer & resume model

**Decision**: `exports/raw/<project-slug>/` containing `state.json` (run metadata + per-candidate progress map keyed by profile URL) and three append-only JSONL files (`candidates.jsonl`, `messages.jsonl`, `notes.jsonl`). Every completed candidate updates `state.json` (candidate marked `done`) after their rows are appended. Resume = load state, skip `done` candidates. A run is `complete` when the roster pass found no new candidates and all are `done`. Re-run of a `complete` project asks fresh vs incremental (FR-015): fresh archives the old raw dir (rename with timestamp) and starts clean; incremental keeps state and only processes roster members not in it.

**Rationale**: JSONL append is crash-safe (no partial-file corruption of earlier records); profile URL is a natural idempotency key (FR-008); state separate from data keeps resume logic trivial.

**Alternatives considered**: SQLite (heavier, opaque in a Drive-synced folder — rejected); single JSON rewritten each save (corruption risk on interrupt — rejected).

## R6. Workbook builder

**Decision**: `scripts/build_workbook.py`, Python 3.11+, executed as `uv run --with openpyxl scripts/build_workbook.py <raw-dir> <output-xlsx>`. Reads the three JSONL files, writes Candidates / Messages / Notes sheets with a frozen header row, auto-width columns, and UTF-8-safe text (Arabic preserved — SC-006). Refuses to overwrite: if the target exists it appends `-HHMMSS` (FR-006). Exits non-zero with a clear message on malformed input.

**Rationale**: openpyxl is the boring, reliable choice; `uv run --with` removes environment setup entirely.

**Alternatives considered**: pandas + xlsxwriter (heavier dependency for three flat sheets — rejected); having the model write xlsx via a generic tool each run (non-deterministic, untestable — rejected).

## R7. Pacing & safety (FR-009, FR-010)

**Decision**: Randomized 2–5 s pause between navigations, longer 8–12 s pause every ~10 candidates. Before every extraction step, a checkpoint scan: URL containing `/checkpoint/`, or page text matching verification/CAPTCHA/unusual-activity phrasing → immediately stop, tell the user what page is showing, end all automation until the user says to resume. Log the stop in `state.json` so the resumed run knows where it was. Never retried automatically, never interacted with.

**Rationale**: This is the user's own account; pacing keeps footprint at human scale, and the checkpoint rule is an absolute per spec — the skill treats it as non-negotiable.

**Alternatives considered**: None — spec mandates this behavior.

## R8. Skill packaging

**Decision**: `.claude/skills/linkedin-recruiter-export/` with `SKILL.md` (frontmatter `name`, `description`; body = phased operating procedure with hard rules up top), `references/extraction-guide.md` (per-page semantic recipes, loaded when extracting), `references/safety-rules.md` (checkpoint/pacing rules, loaded always), `scripts/build_workbook.py`. Description written so the skill triggers on "export recruiter project", "linkedin recruiter export", etc.

**Rationale**: Standard Claude Code skill layout; splitting references keeps SKILL.md lean while the long extraction detail loads only when needed.

**Alternatives considered**: Single monolithic SKILL.md (bloats every invocation — rejected); a plugin (overkill for one project-local workflow — rejected).
