# Extraction Guide — semantic waypoints per page

How to find and read each Recruiter Lite surface WITHOUT CSS class selectors. Signals in priority order: **href patterns → accessibility tree (`read_page`) → visible text (`get_page_text` / `find`)**. `javascript_tool` may harvest hrefs by URL pattern and scroll containers — it must never query obfuscated class names.

> **Status**: waypoints below are UNVERIFIED until the first live recon run (tasks T007/T011/T016). During recon, correct anything that differs, remove this banner from the affected section, and note the verification date. If a waypoint fails during a normal run, stop per safety-rules §5 — do not improvise silently.

## Projects list *(unverified — verify in T007)*

- Entry: `https://www.linkedin.com/talent/home`, then the "Projects" item in the Recruiter navigation.
- Each project appears as a link whose href contains `/talent/hire/` (project ID inside). Harvest name + href pairs from anchors matching that pattern.
- Paginate/scroll the list the same way as the roster loop if the account has many projects.
- Record for each: display name, URL. These feed Phase 1 selection.

## Pipeline roster *(unverified — verify in T007)*

- A project's candidate view lists candidates grouped or filterable by pipeline stage (e.g. Uncontacted / Contacted / Replied). If stages are tabs/filters, iterate every stage so no candidate is missed; the stage label being viewed is the candidate's `stage`.
- Candidate row anatomy (via `read_page` on the list):
  - **Identity**: the row's main link → candidate name (link text) + href. Hrefs containing `/talent/profile/` are talent URLs; a `/in/` href (sometimes behind a "public profile" affordance) is the canonical `profile_url`. If only the talent URL is visible at row level, capture it and pull the `/in/` URL from the candidate detail view during Phase 3/4 visit; until then the talent URL serves as the dedupe key and `profile_url` is backfilled when found.
  - **Fields**: headline/current title, company, location appear as the row's secondary text lines; "date added" often near the stage control. Map by meaning, not position; anything absent → `null`.
- **Pagination/lazy-load loop**: extract visible rows → scroll the list container (or click a next-page control found by its accessible name, e.g. "Next") → wait for load (paced) → extract again → dedupe. End when a full pass yields no new unique candidates and no next control exists.

## Message thread *(unverified — verify in T011)*

- From the candidate's detail view (opened from the project), the messages/InMail area shows the conversation with this candidate. Open it from the candidate — never via the general inbox (inbox filter tabs are irrelevant to capture).
- Thread anatomy (via `read_page` on the conversation panel):
  - Messages appear in chronological order; each carries sender name, timestamp, body text.
  - **Direction**: sender = account owner's name → `sent`; sender = candidate → `received`; an unsent scheduled message (badge/label mentioning "scheduled") → `scheduled`.
  - **Subject**: InMail subject line typically heads the first message only.
- Scroll UP inside the thread container until the earliest message is present before extracting (older messages lazy-load).
- No conversation exists → `messages: none`. Distinguish "no thread" (fine) from "thread area failed to load" (stop, safety-rules §5).

## Notes & tags *(unverified — verify in T016)*

- On the candidate detail view, notes live in a Notes section/tab (recruiter-authored text entries, each with author and date where shown); tags are short labels on the profile header or a tags control.
- Extract every note: text, author (`null` if not shown), date (`null` if not shown) → `kind: note`.
- Extract every tag label → `kind: tag`.
- Neither present → `notes: none`, zero rows, no error.

## General rules

- Extract into memory per page, then append JSONL lines immediately — never batch a whole project in memory.
- Arabic and RTL text: copy exactly as read; no normalization.
- Every navigation and every stage/tab switch counts for pacing (safety-rules §3).
- Checkpoint scan (safety-rules §1) after EVERY navigation, before extraction.
