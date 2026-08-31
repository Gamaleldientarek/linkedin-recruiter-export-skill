# Data Model: LinkedIn Recruiter Lite Export

Raw layer = JSONL (one JSON object per line) + `state.json`, under `exports/raw/<project-slug>/`. All strings UTF-8. Empty/unknown fields are `null`, never guessed (spec: hidden-profile edge case).

## Entity: Project

Captured once per run into `state.json` (not its own file).

| Field | Type | Notes |
|---|---|---|
| `name` | string | As shown in Recruiter Lite |
| `slug` | string | Lowercased, hyphenated, filesystem-safe; derived from name |
| `url` | string | Project URL in Recruiter Lite |
| `candidate_count_seen` | int | Unique candidates observed on roster pass |

## Entity: Candidate — `candidates.jsonl`

Identity/dedupe key: `profile_url` (FR-008).

| Field | Type | Notes |
|---|---|---|
| `profile_url` | string | Canonical LinkedIn profile URL (`/in/...` preferred; talent URL kept in `talent_url` if different) |
| `talent_url` | string\|null | Recruiter-internal profile URL if distinct |
| `full_name` | string | |
| `headline` | string\|null | Headline / current title |
| `company` | string\|null | Current company |
| `location` | string\|null | |
| `stage` | string\|null | Pipeline stage label as displayed |
| `date_added` | string\|null | As displayed; ISO `YYYY-MM-DD` when parseable, else raw text |
| `captured_at` | string | ISO timestamp of extraction |

## Entity: Message — `messages.jsonl`

Belongs to one Candidate via `profile_url`.

| Field | Type | Notes |
|---|---|---|
| `profile_url` | string | FK → Candidate |
| `candidate_name` | string | Denormalized for readable sheets |
| `seq` | int | 1-based order within the thread as displayed |
| `date` | string\|null | As displayed; ISO when parseable |
| `direction` | enum | `sent` \| `received` \| `scheduled` (FR-004; scheduled = written, not yet sent) |
| `subject` | string\|null | InMail subject if shown (usually first message only) |
| `text` | string | Full message body |

## Entity: Note / Tag — `notes.jsonl`

Belongs to one Candidate via `profile_url`.

| Field | Type | Notes |
|---|---|---|
| `profile_url` | string | FK → Candidate |
| `candidate_name` | string | Denormalized |
| `kind` | enum | `note` \| `tag` |
| `text` | string | Note body, or tag label |
| `author` | string\|null | Where visible (FR-005) |
| `date` | string\|null | Where visible |

## Entity: Export Run — `state.json`

One per project raw dir. Rewritten atomically (write temp + rename) at every checkpoint; the JSONL files are the crash-safe record, state is the index.

```json
{
  "project": { "name": "...", "slug": "...", "url": "...", "candidate_count_seen": 0 },
  "run": {
    "mode": "fresh | incremental | resume",
    "started_at": "ISO",
    "completed_at": "ISO | null",
    "status": "in_progress | complete | stopped_checkpoint | stopped_error",
    "stop_reason": "string | null"
  },
  "candidates": {
    "<profile_url>": {
      "roster": "done | pending",
      "messages": "done | pending | none",
      "notes": "done | pending | none"
    }
  }
}
```

### State transitions

- Run: `in_progress` → `complete` (all candidates fully `done`/`none` and roster pass exhausted) | `stopped_checkpoint` (FR-010) | `stopped_error` (FR-012).
- Candidate phase fields: `pending` → `done` (rows appended) or `none` (verified empty — distinct from pending so resume skips it).
- Resume rule (FR-007): any run with `status != complete` resumes; only `pending` work is executed.
- Completed-project re-run (FR-015): user chooses `fresh` (raw dir archived to `raw/<slug>-archived-<timestamp>/`, new state) or `incremental` (existing state kept; roster re-walked; only unknown profile URLs become `pending`).

## Validation rules

- A Candidate line with empty `profile_url` or `full_name` is invalid → run stops with named error (FR-012).
- Duplicate `profile_url` in `candidates.jsonl` is tolerated on disk (append-only); consumers dedupe keeping the latest line.
- Messages/notes referencing a `profile_url` absent from candidates → workbook builder fails with a clear message (guards misattribution, SC-002).
