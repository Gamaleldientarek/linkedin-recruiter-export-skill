# Contract: Workbook Format & Builder CLI

## Builder CLI

```
uv run --with openpyxl scripts/build_workbook.py <raw-dir> [--out <path>]
```

- `<raw-dir>`: e.g. `exports/raw/finance-managers/`
- `--out` default: `exports/<project-slug>-<YYYY-MM-DD>.xlsx` (date = today). If the path exists, builder writes `<name>-HHMMSS.xlsx` instead — it NEVER overwrites (FR-006).
- Exit 0 on success, printing the absolute output path and row counts per sheet.
- Exit non-zero with a one-line named error on: missing/unparseable input files, invalid candidate records, messages/notes referencing unknown `profile_url` (misattribution guard).

## Workbook

One file, three sheets, in this order. Header row frozen (`A2` freeze panes), bold headers, sensible column widths, cell wrapping on long-text columns. Text is written as strings (no lossy type coercion); Arabic/RTL text preserved verbatim (SC-006).

### Sheet "Candidates" — one row per unique candidate

| Col | Header | Source |
|---|---|---|
| A | Name | `full_name` |
| B | Profile URL | `profile_url` (hyperlinked) |
| C | Headline | `headline` |
| D | Company | `company` |
| E | Location | `location` |
| F | Stage | `stage` |
| G | Date Added | `date_added` |
| H | Messages | count of that candidate's message rows |
| I | Notes/Tags | count of that candidate's note rows |

### Sheet "Messages" — one row per message, grouped by candidate, thread order

| Col | Header | Source |
|---|---|---|
| A | Candidate | `candidate_name` |
| B | Profile URL | `profile_url` |
| C | # | `seq` |
| D | Date | `date` |
| E | Direction | `direction` (`sent` / `received` / `scheduled`) |
| F | Subject | `subject` |
| G | Message | `text` (wrapped) |

### Sheet "Notes" — one row per note or tag

| Col | Header | Source |
|---|---|---|
| A | Candidate | `candidate_name` |
| B | Profile URL | `profile_url` |
| C | Type | `kind` (`note` / `tag`) |
| D | Text | `text` (wrapped) |
| E | Author | `author` |
| F | Date | `date` |

## Acceptance hooks

- Row-count parity: Candidates sheet rows == unique `profile_url` count after dedupe (SC-001).
- A candidate with zero messages/notes appears only on Candidates (spec US2/US3 acceptance).
- Empty project → all three sheets exist with headers only (edge case: empty project).
