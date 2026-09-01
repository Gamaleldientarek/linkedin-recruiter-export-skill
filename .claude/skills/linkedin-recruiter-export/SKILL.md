---
name: linkedin-recruiter-export
description: Export LinkedIn Recruiter Lite data to Excel using the user's own logged-in Chrome via Claude-in-Chrome. Use when the user asks to export a Recruiter or Recruiter Lite project, export their InMail inbox or message threads, export LinkedIn recruiter data, pull their recruiter pipeline, export candidates/InMails/notes from LinkedIn, or turn Recruiter data into a spreadsheet/workbook. Two sources: a user-chosen project pipeline, or the InMail inbox (all four folders) when no project exists. Every candidate row always includes their LinkedIn profile link. Exports candidates, full InMail threads, and notes & tags into exports/<slug>-<date>.xlsx. Resumable, human-paced, stops hard on any security checkpoint.
---

# LinkedIn Recruiter Lite Export

Drive the user's own logged-in Chrome (Claude-in-Chrome MCP tools) to export Recruiter Lite data into an Excel workbook. Two sources, one per run:

- **Project mode** — ONE user-chosen project pipeline (roster + per-candidate threads + notes).
- **Inbox mode** — the InMail inbox itself (all four folders), for seats with no projects or when the user asks to export their InMails/messages.

Raw data lands incrementally in `exports/raw/<slug>/` (JSONL + `state.json`), then `scripts/build_workbook.py` builds `exports/<slug>-<YYYY-MM-DD>.xlsx` with Candidates / Messages / Notes sheets.

**Read `references/safety-rules.md` FIRST. Its rules override everything below.**

## Hard rules (non-negotiable)

1. **Security checkpoint / CAPTCHA / verification page → full stop**, record `stopped_checkpoint` in state, hand control to the user. Never interact with such a page in any way. Details: safety-rules §1.
2. **Never handle credentials.** Logged out → the user logs in themselves (safety-rules §2).
3. **No hardcoded CSS class selectors.** LinkedIn's class names are obfuscated and rotate. Extract via: profile-URL href patterns → accessibility tree (`read_page`) → visible text (`get_page_text`, `find`). If an expected structure is missing, stop loudly naming it (safety-rules §5).
4. **Human pacing**: randomized 2–5 s between navigations; 8–12 s rest every ~10 candidates (safety-rules §3).
5. **Main session only.** Browser tools are not available to subagents — never delegate browser steps.
6. **Local only.** Data goes to `exports/` and nowhere else. Missing fields stay `null`, never guessed.
7. **Every candidate row must carry their LinkedIn profile link.** The profile URL (from the thread-header or roster-row href, `?trk` query stripped) is both the dedupe key and a required export column. If a candidate's profile link cannot be found, stop and name the candidate rather than exporting a row without it.
8. **Read-only.** Never click Edit, Delete, Add note, Add reminder, call, or reply controls — the export must leave the account exactly as it found it.

## Phase 0 — Preconditions

1. Load Chrome tools if deferred, in ONE ToolSearch call: `tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `computer`, `read_page`, `get_page_text`, `find`, `javascript_tool`. Unavailable → stop; tell the user to enable the Claude-in-Chrome extension and permit linkedin.com.
2. `tabs_context_mcp`, then `tabs_create_mcp` → dedicated work tab. Never reuse tab IDs from previous sessions.
3. Navigate to `https://www.linkedin.com/talent/home`. Verify a logged-in Recruiter surface: "Recruiter Lite" top nav with "Create a project" / "Reports" links and the seat name in the sidebar, no sign-in form (there is NO "Projects" nav link — that's normal, see extraction-guide). Run the checkpoint scan (safety-rules §1). Not logged in → stop, ask the user to log in, wait.

## Phase 1 — Source selection & run mode

1. **Pick the source.** If the user asked for their InMails/inbox/messages → **inbox mode**. Otherwise navigate directly to `https://www.linkedin.com/talent/projects` (there is no nav link — see extraction-guide "Projects list") and read the `Projects (N)` heading:
   - N ≥ 1 → **project mode**: extract project names (+ URLs) via `read_page`; a user-given name that uniquely matches wins, otherwise present the list and ask the user to pick ONE.
   - N = 0 → tell the user this seat has no projects and offer **inbox mode**; proceed only on their yes.
2. Slugify: project name for project mode; the literal `inbox` for inbox mode → `exports/raw/<slug>/`.
4. **Run-mode decision** (read `state.json` if it exists):

| State | Action |
|---|---|
| No state.json | New run: `mode: fresh`, create dirs + fresh state |
| `status` ≠ `complete` | **Resume**: report counts (done vs pending), continue pending work only. If prior status was `stopped_checkpoint`, confirm the user resolved it before resuming |
| `status` = `complete` | **Ask the user**: fresh (rename raw dir → `raw/<slug>-archived-<YYYYMMDD-HHMMSS>/`, start clean) or incremental (keep state; only candidates not already in it become pending) |

## Phase 2 — Roster (candidates)

**Inbox mode** replaces Phases 2–3 with the inbox walk below; Phase 4 notes and Phase 5 workbook apply unchanged.

### Inbox mode — walk the four folders

Goal: every conversation partner, exactly once, keyed by profile URL, with their full thread.

1. Walk the folders in order — `main`, `awaitingreply`, `scheduled`, `archived` (`/talent/inbox/0/<folder>`, waypoints in extraction-guide "Inbox"). An empty folder (e.g. "Your message threads will appear here…") is legitimate — record zero and move on.
2. In each folder, list thread rows: candidate name, thread permalink (`/talent/inbox/0/<folder>/id/<id>`), date, status label (e.g. `Pending`). Add unseen threads to state as pending; the same candidate in two folders dedupes by profile URL once the thread is opened.
3. Open each pending thread (paced). From the thread header extract the candidate: name, **profile URL** (the `/talent/profile/<id>` href, `?trk` stripped — required, hard rule 7), headline, location, industry, connection degree. Stage column = folder name (`Inbox` / `Awaiting Reply` / `Scheduled` / `Archived`); date added = thread date. Append to `candidates.jsonl`.
4. In the same visit extract every message per extraction-guide "Messages within a thread": seq, date-time, direction (sender = seat name → `sent`, else `received`; `scheduled` in the Scheduled folder), subject where present, full body text. Verify count against the "N messages" activity button. Append to `messages.jsonl`, mark thread `done` in state.
5. Completion: all four folders walked, every thread `done`/`none`.

### Project mode — pipeline roster

Goal: every candidate in the pipeline, exactly once, keyed by profile URL.

1. Open the project's pipeline/candidates view. Extract visible candidate rows per extraction-guide "Pipeline roster": name, profile URL (from hrefs — canonical `/in/…` preferred, talent URL kept separately), headline, company, location, stage, date added. Blank → `null`.
2. Dedupe against URLs already seen this pass and in state. New candidates: append line to `candidates.jsonl`, add to `state.json` as `{roster: done, messages: pending, notes: pending}`.
3. Advance: scroll to load more / next page control. **Completion condition**: a full pass adds no new unique candidates AND no next-page control exists. Never assume a fixed page count.
4. Pace every navigation (hard rule 4). Atomic state writes: temp file + rename.
5. Record `project.candidate_count_seen`. Empty project → skip to Phase 5 (workbook with headers only, clear message).

## Phase 3 — Messages (per candidate, project mode only)

Goal: complete thread per candidate, inbox-state independent (Inbox / Awaiting Reply / Archived all included — in project mode threads are opened from the candidate, never by walking inbox folders). Inbox mode already captured messages in its folder walk — skip this phase.

For each candidate with `messages: pending`:

1. Open the candidate's detail view from the project, then their conversation/messages area (extraction-guide "Message thread").
2. No thread → set `messages: none`, move on (zero rows is correct, not an error).
3. Extract every message in display order: `seq` (1-based), date, direction — `sent` if the sender is the account owner, `received` if the candidate, `scheduled` for written-but-unsent messages where shown — subject (usually first InMail only), full text. Scroll up within the thread until the earliest message is loaded.
4. Append one line per message to `messages.jsonl`, then set `messages: done`.
5. Pace between candidates; long rest every ~10 (hard rule 4).

## Phase 4 — Notes & tags (per candidate)

For each candidate with `notes: pending`: open the notes/tags area — the candidate detail view in project mode, or the thread view (where "Add note" sits) in inbox mode (extraction-guide "Notes & tags"). Extract each note (text, author + date where visible) as `kind: note` and each tag label as `kind: tag`; append to `notes.jsonl`; set `notes: done` (or `none`). Same pacing. This phase can run in the same per-candidate visit as Phase 3 to halve page loads — extract messages then notes before moving on.

## Phase 5 — Workbook & summary

1. Mark run complete in state: `status: complete`, `completed_at` timestamp — only when every candidate is `done`/`none` in all fields.
2. Build: `uv run --with openpyxl .claude/skills/linkedin-recruiter-export/scripts/build_workbook.py exports/raw/<slug> --out exports/<slug>-<YYYY-MM-DD>.xlsx`. The script never overwrites (auto `-HHMMSS` suffix) and fails loudly on malformed data — report its error verbatim if it fails.
3. Report to the user: project name, candidates exported, message rows, note/tag rows, output file path, and any candidates skipped with reasons.

## Interruption & errors

- Any stop (checkpoint, error, tab closed, user interrupt): data already appended is safe; the next invocation resumes via Phase 1's run-mode table.
- Missing expected structure anywhere: safety-rules §5 — stop, name it, never guess.
