---
name: linkedin-recruiter-export
description: Export a LinkedIn Recruiter Lite project to Excel using the user's own logged-in Chrome via Claude-in-Chrome. Use when the user asks to export a Recruiter or Recruiter Lite project, export LinkedIn recruiter data, pull their recruiter pipeline, export candidates/InMails/notes from LinkedIn, or turn a Recruiter project into a spreadsheet/workbook. Exports candidates (name, profile URL, headline, company, location, stage, date added), full InMail threads, and notes & tags for ONE user-chosen project per run into exports/<project>-<date>.xlsx. Resumable, human-paced, stops hard on any security checkpoint.
---

# LinkedIn Recruiter Lite Export

Drive the user's own logged-in Chrome (Claude-in-Chrome MCP tools) to export ONE Recruiter Lite project per run into an Excel workbook. Raw data lands incrementally in `exports/raw/<project-slug>/` (JSONL + `state.json`), then `scripts/build_workbook.py` builds `exports/<project-slug>-<YYYY-MM-DD>.xlsx` with Candidates / Messages / Notes sheets.

**Read `references/safety-rules.md` FIRST. Its rules override everything below.**

## Hard rules (non-negotiable)

1. **Security checkpoint / CAPTCHA / verification page → full stop**, record `stopped_checkpoint` in state, hand control to the user. Never interact with such a page in any way. Details: safety-rules §1.
2. **Never handle credentials.** Logged out → the user logs in themselves (safety-rules §2).
3. **No hardcoded CSS class selectors.** LinkedIn's class names are obfuscated and rotate. Extract via: profile-URL href patterns → accessibility tree (`read_page`) → visible text (`get_page_text`, `find`). If an expected structure is missing, stop loudly naming it (safety-rules §5).
4. **Human pacing**: randomized 2–5 s between navigations; 8–12 s rest every ~10 candidates (safety-rules §3).
5. **Main session only.** Browser tools are not available to subagents — never delegate browser steps.
6. **Local only.** Data goes to `exports/` and nowhere else. Missing fields stay `null`, never guessed.

## Phase 0 — Preconditions

1. Load Chrome tools if deferred, in ONE ToolSearch call: `tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `computer`, `read_page`, `get_page_text`, `find`, `javascript_tool`. Unavailable → stop; tell the user to enable the Claude-in-Chrome extension and permit linkedin.com.
2. `tabs_context_mcp`, then `tabs_create_mcp` → dedicated work tab. Never reuse tab IDs from previous sessions.
3. Navigate to `https://www.linkedin.com/talent/home`. Verify a logged-in Recruiter surface (Projects navigation visible, no sign-in form). Run the checkpoint scan (safety-rules §1). Not logged in → stop, ask the user to log in, wait.

## Phase 1 — Project selection & run mode

1. Open the Projects area from Recruiter navigation. Extract project names (+ their URLs) from the projects list via `read_page`; recipes in `references/extraction-guide.md`.
2. If the user gave a project name that uniquely matches → use it. Otherwise present the list and ask the user to pick ONE.
3. Slugify the name (lowercase, hyphens, filesystem-safe) → `exports/raw/<slug>/`.
4. **Run-mode decision** (read `state.json` if it exists):

| State | Action |
|---|---|
| No state.json | New run: `mode: fresh`, create dirs + fresh state |
| `status` ≠ `complete` | **Resume**: report counts (done vs pending), continue pending work only. If prior status was `stopped_checkpoint`, confirm the user resolved it before resuming |
| `status` = `complete` | **Ask the user**: fresh (rename raw dir → `raw/<slug>-archived-<YYYYMMDD-HHMMSS>/`, start clean) or incremental (keep state; only candidates not already in it become pending) |

## Phase 2 — Roster (candidates)

Goal: every candidate in the pipeline, exactly once, keyed by profile URL.

1. Open the project's pipeline/candidates view. Extract visible candidate rows per extraction-guide "Pipeline roster": name, profile URL (from hrefs — canonical `/in/…` preferred, talent URL kept separately), headline, company, location, stage, date added. Blank → `null`.
2. Dedupe against URLs already seen this pass and in state. New candidates: append line to `candidates.jsonl`, add to `state.json` as `{roster: done, messages: pending, notes: pending}`.
3. Advance: scroll to load more / next page control. **Completion condition**: a full pass adds no new unique candidates AND no next-page control exists. Never assume a fixed page count.
4. Pace every navigation (hard rule 4). Atomic state writes: temp file + rename.
5. Record `project.candidate_count_seen`. Empty project → skip to Phase 5 (workbook with headers only, clear message).

## Phase 3 — Messages (per candidate)

Goal: complete thread per candidate, inbox-state independent (Inbox / Awaiting Reply / Archived all included — threads are opened from the candidate, never by walking inbox folders).

For each candidate with `messages: pending`:

1. Open the candidate's detail view from the project, then their conversation/messages area (extraction-guide "Message thread").
2. No thread → set `messages: none`, move on (zero rows is correct, not an error).
3. Extract every message in display order: `seq` (1-based), date, direction — `sent` if the sender is the account owner, `received` if the candidate, `scheduled` for written-but-unsent messages where shown — subject (usually first InMail only), full text. Scroll up within the thread until the earliest message is loaded.
4. Append one line per message to `messages.jsonl`, then set `messages: done`.
5. Pace between candidates; long rest every ~10 (hard rule 4).

## Phase 4 — Notes & tags (per candidate)

For each candidate with `notes: pending`: open the notes/tags area on their detail view (extraction-guide "Notes & tags"). Extract each note (text, author + date where visible) as `kind: note` and each tag label as `kind: tag`; append to `notes.jsonl`; set `notes: done` (or `none`). Same pacing. This phase can run in the same per-candidate visit as Phase 3 to halve page loads — extract messages then notes before moving on.

## Phase 5 — Workbook & summary

1. Mark run complete in state: `status: complete`, `completed_at` timestamp — only when every candidate is `done`/`none` in all fields.
2. Build: `uv run --with openpyxl .claude/skills/linkedin-recruiter-export/scripts/build_workbook.py exports/raw/<slug> --out exports/<slug>-<YYYY-MM-DD>.xlsx`. The script never overwrites (auto `-HHMMSS` suffix) and fails loudly on malformed data — report its error verbatim if it fails.
3. Report to the user: project name, candidates exported, message rows, note/tag rows, output file path, and any candidates skipped with reasons.

## Interruption & errors

- Any stop (checkpoint, error, tab closed, user interrupt): data already appended is safe; the next invocation resumes via Phase 1's run-mode table.
- Missing expected structure anywhere: safety-rules §5 — stop, name it, never guess.
