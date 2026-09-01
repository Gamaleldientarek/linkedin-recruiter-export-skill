# Handoff — LinkedIn Recruiter Lite Export Skill

**Last updated:** 2026-09-01 · **Branch:** `main` (in sync with origin) ·
**Repo (public):** github.com/Gamaleldientarek/linkedin-recruiter-export-skill

Read this first when picking the project back up. It says where things stand,
what changed on 2026-09-01, and exactly what to do next.

## What this project is

A Claude Code skill (`.claude/skills/linkedin-recruiter-export/`) that drives
Jimmy's own logged-in Chrome (Claude-in-Chrome MCP) to export LinkedIn
Recruiter Lite data — candidates, full InMail threads, notes/tags — into an
Excel workbook via `scripts/build_workbook.py`. Spec-driven: all design docs
live in `specs/001-recruiter-lite-export/` (spec, plan, tasks, contracts,
quickstart with a dated validation log).

## Current state — done and verified

- **Offline layer complete**: workbook builder + 10 passing pytest tests
  (`uv run --with openpyxl,pytest pytest tests/`), fixtures, SKILL.md,
  safety-rules.md, README.
- **Skill has two modes** (one per run):
  - **Project mode** — the original spec: pick a project, walk its pipeline.
    *Never live-tested — the seat has no projects (see Blockers).*
  - **Inbox mode** — added 2026-09-01 per Jimmy: when there are no projects
    (or he asks for his InMails), walk the four inbox folders
    (`/talent/inbox/0/{main,awaitingreply,scheduled,archived}`) and export
    every conversation partner + full thread. **Validated end-to-end on real
    data** (1 thread) → `exports/inbox-2026-09-01.xlsx` verified correct,
    including Arabic text and the profile-URL hyperlink.
- **Live recon recorded** in `references/extraction-guide.md`: login signals,
  projects-list URL + empty state, all inbox waypoints (thread rows, thread
  header incl. `/talent/profile/<id>` href, message blocks, direction rule,
  empty-folder and can't-reply-yet states). Semantic waypoints only — no CSS
  class selectors, they rotate.
- **Hard rules added 2026-09-01** (SKILL.md + safety-rules + interface
  contract): every candidate row MUST carry the LinkedIn profile link (also
  the dedupe key; missing link → loud stop), and the skill is strictly
  read-only (never click Edit/Delete/Add note/reminder/reply/call/archive).
- **Self-review (T021) done** — safety-rules §4 and the skill-interface
  contract were realigned to cover inbox mode.

## Blockers — why the remaining tasks are open

The Recruiter Lite seat (**AZM X People and Culture**) currently contains:
**zero projects**, one sent test InMail (Awaiting Reply, to Gamal Eldien
himself), no replies, no notes. Everything still open in
`specs/001-recruiter-lite-export/tasks.md` is blocked on data existing:

| Open task | Needs |
|---|---|
| T007, T010 — project-mode recon + roster validation | a real project with candidates |
| T013 — message validation at scale (3+ threads, one archived) | real InMail activity |
| T015 — interrupt/resume validation | a run long enough to interrupt |
| T016, T018 — notes/tags recon + validation | a candidate with an existing note |
| T019 — end-of-run summary check | any fuller live run |

## How to resume (next session)

1. Chrome connection: `tabs_context_mcp` — if "extension not connected",
   have Jimmy click the Claude extension icon in Chrome (Default profile) or
   restart Chrome; it must be signed into claude.ai as the same account.
   This failed twice before connecting last session.
2. Check what data now exists: `linkedin.com/talent/projects` (heading
   `Projects (N)`) and the inbox folders.
3. If a project exists → run T007 recon (fill extraction-guide "Pipeline
   roster" + "Projects list" rows), then T010 validation.
4. If only more InMails exist → re-run inbox mode at scale; that covers
   T013/T015/T019 substance; append results to quickstart.md validation log.
5. Notes recon (T016) the first time any candidate has a note.
6. Commit per phase; **ask Jimmy before every push** (repo is public).

## Gotchas learned live

- There is **no "Projects" nav link** in Recruiter Lite — go straight to
  `/talent/projects`. The logo's `/talent/hire` just redirects home.
- Sent-but-unanswered InMails live in **Awaiting Reply**, not Inbox; the
  message body of a pending InMail sits in an editable element
  ("Click to edit the message body") — extract text only, never click.
- `read_page` truncates long headlines; the screenshot/`get_page_text` has
  the full text.
- A credential-sharing reminder banner may appear above the nav — it is not
  a checkpoint; dismissable, ignorable.
- `exports/` is gitignored — keep it that way; candidate data never enters
  the public repo. Fixture data in `tests/fixtures/` is fake.

## Key files

- `.claude/skills/linkedin-recruiter-export/SKILL.md` — the skill (modes,
  phases, hard rules)
- `.claude/skills/linkedin-recruiter-export/references/extraction-guide.md`
  — live-verified waypoints (partial: project mode still unrecorded)
- `.claude/skills/linkedin-recruiter-export/references/safety-rules.md` —
  overrides everything; checkpoint stop, pacing, scope, read-only
- `specs/001-recruiter-lite-export/tasks.md` — task list with 2026-09-01
  status note
- `specs/001-recruiter-lite-export/quickstart.md` — scenarios + dated
  validation log (append future validations there)
