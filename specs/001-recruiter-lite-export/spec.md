# Feature Specification: LinkedIn Recruiter Lite Export

**Feature Branch**: `001-recruiter-lite-export`

**Created**: 2026-08-31

**Status**: Draft

**Input**: User description: "Build a Claude Code skill named linkedin-recruiter-export that exports data from the user's own LinkedIn Recruiter Lite account by driving their logged-in Chrome browser through the Claude-in-Chrome browser tools (Recruiter Lite has no native CSV export). Per run the user picks ONE Recruiter Lite project; the skill exports candidates (name, LinkedIn profile URL, headline, company, location, pipeline stage, date added), full InMail/message threads, and notes & tags, into one Excel workbook with Candidates / Messages / Notes sheets, built from incrementally-saved raw data so interrupted runs resume. Must not rely on hardcoded LinkedIn CSS selectors, must pace navigation like a human, and must stop and hand control to the user on any security checkpoint page."

## Clarifications

### Session 2026-08-31

- Q: When you re-run the skill on a project that was already fully exported before, what should happen? → A: Ask each time — the user chooses between a fresh full re-export and an incremental update at the start of every re-run.
- Q: If an export file for the same project and same date already exists, what should the new run do? → A: Add a time suffix — existing export files are never overwritten.
- Q: Where should the exported workbooks (candidate PII) be stored? → A: In the project's `exports/` folder (Google-Drive-synced, consistent with how AZMX client work is already stored).
- Q: Does the message export cover all inbox states (Inbox / Awaiting Reply / Scheduled / Archived)? → A: Yes — messages are captured per candidate directly from each conversation thread, not by walking inbox folders, so inbox filter state is irrelevant; archived conversations are included, and scheduled (not-yet-sent) messages are captured and labeled `scheduled` where visible.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Export a project's candidate list (Priority: P1)

Jimmy (recruiting for AZMX) opens Claude Code, invokes the export skill, sees a list of his Recruiter Lite projects, picks one, and receives an Excel workbook containing every candidate in that project's pipeline — full name, LinkedIn profile URL, headline/current title, company, location, pipeline stage, and date added.

**Why this priority**: The candidate roster with profile links is the core data Recruiter Lite holds hostage (no native export). On its own it is already a usable deliverable for HR reporting and outreach tracking.

**Independent Test**: Run the skill against a small project (e.g., 10–20 candidates) and compare the workbook's Candidates sheet against what is visible in the Recruiter Lite pipeline: every candidate present exactly once, every field matching, every profile link opening the right person.

**Acceptance Scenarios**:

1. **Given** the user is logged into Recruiter Lite in their own Chrome browser, **When** they invoke the skill, **Then** they are shown their projects by name and asked to choose one.
2. **Given** a chosen project with candidates spread across multiple pages, **When** the export runs, **Then** all pages are traversed and every visible candidate appears exactly once in the Candidates sheet.
3. **Given** a completed run, **Then** a single workbook exists at `exports/<project-name>-<date>.xlsx` with the Candidates sheet populated and each row carrying a working LinkedIn profile URL.

---

### User Story 2 - Export full message history per candidate (Priority: P2)

For the same chosen project, the workbook's Messages sheet contains the complete InMail/message thread for each candidate: one row per message with the candidate's name, profile URL, message date, direction (sent or received), and full message text.

**Why this priority**: Message history shows who was contacted, what was said, and who replied — essential for handover, auditing outreach, and avoiding duplicate contact. It depends on the candidate list existing (P1).

**Independent Test**: Pick three candidates with known conversations; verify their threads in the Messages sheet match the conversations visible in Recruiter Lite message-for-message, in order, with correct direction labels.

**Acceptance Scenarios**:

1. **Given** a candidate with a multi-message thread, **When** the export runs, **Then** every message in the thread appears as its own row with date, direction, and full text.
2. **Given** a candidate who was never messaged, **When** the export runs, **Then** the candidate still appears in the Candidates sheet and the Messages sheet simply has no rows for them (no error, no placeholder noise).

---

### User Story 3 - Export notes and tags per candidate (Priority: P3)

The workbook's Notes sheet contains every note and tag attached to each candidate in the project: candidate name, profile URL, note text, note date/author where visible, and tags.

**Why this priority**: Notes and tags carry the recruiter's own judgments (screening outcomes, ratings, reminders). Valuable, but smaller in volume and less critical than the roster and message history.

**Independent Test**: Pick candidates with known notes/tags; verify each note and tag appears in the Notes sheet attributed to the right candidate.

**Acceptance Scenarios**:

1. **Given** a candidate with two notes and one tag, **When** the export runs, **Then** the Notes sheet holds those entries attributed to that candidate.
2. **Given** a candidate with no notes or tags, **Then** no rows are produced for them and no error occurs.

---

### User Story 4 - Resume an interrupted run (Priority: P2)

If a run stops partway — browser tab closed, session interrupted, LinkedIn security prompt — re-invoking the skill for the same project continues from where it left off instead of starting over.

**Why this priority**: A full-thread export visits one conversation per candidate at human pace; a 100-candidate project is a long run. Without resume, any interruption wastes the entire session and doubles the account's page-visit footprint.

**Independent Test**: Deliberately stop a run after ~half the candidates are processed, re-invoke the skill, and confirm the already-processed candidates are not revisited and the final workbook is complete.

**Acceptance Scenarios**:

1. **Given** a run interrupted after N candidates, **When** the skill is re-invoked for the same project, **Then** it reports how many candidates are already done and continues with the remainder only.
2. **Given** a resumed run that completes, **Then** the final workbook is identical in coverage to what an uninterrupted run would have produced (every candidate exactly once).

---

### Edge Cases

- **Security checkpoint / CAPTCHA / unusual-activity page**: the run stops immediately, tells the user exactly what happened and which page is showing, and hands the browser back to them. It never attempts to solve, bypass, or retry through a verification page. After the user resolves it manually, they may resume.
- **Not logged in / session expired**: detected up front; the user is asked to log in themselves before anything else happens.
- **Empty project**: produces a workbook with headers and zero data rows, plus a clear message — not an error.
- **Duplicate candidate encountered across pages** (pagination overlap, lazy-load re-renders): deduplicated by LinkedIn profile URL; a candidate never appears twice.
- **Candidate with a hidden/restricted profile or missing fields**: exported with whatever fields are visible; missing fields left blank rather than guessed.
- **Arabic or other non-Latin names and message text**: preserved exactly in the workbook (correct encoding, no mojibake).
- **Pagination end detection**: the run ends when a full pass yields no new unique candidates and no next-page control exists — never by assuming a fixed page count.
- **LinkedIn UI changed since the skill was written**: when an expected element (project list, pipeline, thread) cannot be found, the run stops with a clear "UI changed / cannot find X" message rather than exporting wrong or partial data silently.
- **Browser tab closed mid-run**: treated as an interruption; already-saved data is kept and the run is resumable.
- **Re-run of an already-completed project**: the user is asked to choose fresh full re-export or incremental update; the previous workbook is never overwritten (new file gets a time suffix).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The skill MUST operate only on the user's own logged-in LinkedIn Recruiter Lite session in their own browser; it MUST NOT collect or handle the user's credentials.
- **FR-002**: The skill MUST list the account's Recruiter Lite projects by name and let the user choose exactly one project per run.
- **FR-003**: For the chosen project, the skill MUST capture every candidate in the pipeline with: full name, LinkedIn profile URL, headline/current title, current company, location, pipeline stage, and date added — leaving blank any field not visible.
- **FR-004**: The skill MUST capture the complete message/InMail thread for each candidate as individual messages, each with date, direction (sent/received), and full text. Capture is per candidate conversation, independent of inbox filter state — threads in Inbox, Awaiting Reply, or Archived are all included, and scheduled (not-yet-sent) messages are captured labeled `scheduled` where visible.
- **FR-005**: The skill MUST capture all notes (text, and date/author where visible) and tags attached to each candidate in the project.
- **FR-006**: The skill MUST produce a single Excel workbook per run named after the project and date, with three sheets — Candidates, Messages, Notes — where Messages and Notes rows are linkable to Candidates rows via the profile URL. If a workbook for the same project and date already exists, the new file MUST get a time suffix; existing exports are never overwritten.
- **FR-007**: The skill MUST save extracted data incrementally during the run, and a re-run of an **unfinished** export MUST resume from saved progress without revisiting completed candidates.
- **FR-015**: When the chosen project already has a **completed** export, the skill MUST ask the user at the start of the run whether to perform a fresh full re-export or an incremental update (visiting only candidates not present in the last completed export), and behave accordingly.
- **FR-016**: Exported workbooks and raw progress data MUST be stored in the project's `exports/` folder (which syncs with the user's Google Drive).
- **FR-008**: Candidates MUST be deduplicated by LinkedIn profile URL across pages and passes.
- **FR-009**: Navigation MUST be paced with deliberate human-like delays between page loads and actions.
- **FR-010**: On encountering any security verification page (checkpoint, CAPTCHA, unusual-activity warning), the skill MUST stop immediately, inform the user, and hand control back — it MUST NOT attempt to bypass or automate through it.
- **FR-011**: Extraction MUST NOT depend on LinkedIn's obfuscated/unstable styling identifiers; it MUST read the page through stable means (visible text, document structure, link targets) so that cosmetic UI changes do not silently corrupt output.
- **FR-012**: When a required page element cannot be found, the skill MUST fail loudly with a message naming what it could not find, rather than continuing with wrong or empty data.
- **FR-013**: At the end of each run, the skill MUST report a summary: project name, candidates exported, messages exported, notes/tags exported, output file path, and any candidates skipped with reasons.
- **FR-014**: All exported files MUST stay on the user's machine; nothing is sent to any external service.

### Key Entities

- **Project**: a Recruiter Lite project (name, URL/identifier, candidate count); the unit of one export run.
- **Candidate**: a person in a project's pipeline; keyed by LinkedIn profile URL; carries name, headline, company, location, stage, date added; parent of Messages and Notes.
- **Message**: one message in a candidate's InMail thread; date, direction (sent/received), text; belongs to exactly one Candidate.
- **Note / Tag**: recruiter-authored note text (with date/author where visible) or tag label; belongs to exactly one Candidate.
- **Export Run**: one invocation for one project on one date; owns its progress state (which candidates are done) and its output workbook.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of candidates visible in the chosen project's pipeline appear in the export, each exactly once, with a working profile link.
- **SC-002**: For candidates with conversations, the exported thread matches the on-screen conversation message-for-message (spot-check of at least 3 candidates per run shows zero missing or misattributed messages).
- **SC-003**: A 50-candidate project with message history exports end-to-end in under 45 minutes of unattended time; the user's manual involvement is limited to picking the project and (rarely) resolving a security prompt.
- **SC-004**: An interrupted run, when resumed, completes without re-processing already-finished candidates, and the final workbook coverage equals an uninterrupted run's.
- **SC-005**: Zero automated interactions with security verification pages across all runs — every checkpoint results in an immediate stop and handoff.
- **SC-006**: The workbook opens correctly in Excel with Arabic and English text intact, and a non-technical HR colleague can filter candidates by stage without any rework.

## Assumptions

- The user has an active LinkedIn Recruiter Lite subscription and is already logged in via Chrome with the Claude-in-Chrome extension installed and permitted on linkedin.com.
- Only what is visible to the user's own account is exported; the skill never accesses data the user could not see manually. This is the user's own recruiting pipeline, exported for legitimate internal HR use at AZMX.
- One project per run is the operating model; multiple projects mean multiple runs.
- "Date added" and note author/date are exported only where Recruiter Lite displays them; Lite hides some fields that full Recruiter shows.
- Export volume is small-business scale (projects of tens to low hundreds of candidates), not bulk scraping scale.
- The output workbook is the deliverable; no database, no sync, no scheduling in v1.
- Resume state is per project and kept locally alongside the raw export data.
