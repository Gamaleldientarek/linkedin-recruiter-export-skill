# Contract: Raw Data Layer

Directory: `exports/raw/<project-slug>/`. Field-level definitions live in [data-model.md](../data-model.md); this contract fixes the file behavior both producers (browser phase) and consumers (workbook builder) rely on.

## Files

| File | Format | Writer behavior |
|---|---|---|
| `state.json` | Single JSON object | Atomic rewrite (temp file + rename) after each candidate completes |
| `candidates.jsonl` | One Candidate object per line | Append-only |
| `messages.jsonl` | One Message object per line | Append-only |
| `notes.jsonl` | One Note/Tag object per line | Append-only |

## Rules

1. **Append order**: a candidate's rows are appended to the JSONL files BEFORE their `state.json` entry flips to `done`. (Crash between the two → rows re-appended on resume; consumers dedupe.)
2. **Dedupe on read**: consumers key candidates by `profile_url` keeping the last occurrence; messages by (`profile_url`, `seq`) keeping the last; notes by (`profile_url`, `kind`, `text`).
3. **Encoding**: UTF-8, no BOM. Newlines inside message/note text are escaped per JSON (`\n`) — one record is always one physical line.
4. **Unknown fields**: consumers ignore fields they don't recognize (forward compatibility).
5. **Archival**: a `fresh` re-run renames the whole dir to `raw/<slug>-archived-<YYYYMMDD-HHMMSS>/` before starting; nothing is deleted.

## Example lines

```jsonl
{"profile_url":"https://www.linkedin.com/in/example-person","talent_url":null,"full_name":"Example Person","headline":"Senior Accountant","company":"Acme Co","location":"Riyadh, Saudi Arabia","stage":"Contacted","date_added":"2026-08-12","captured_at":"2026-08-31T17:05:00+02:00"}
```

```jsonl
{"profile_url":"https://www.linkedin.com/in/example-person","candidate_name":"Example Person","seq":1,"date":"2026-08-13","direction":"sent","subject":"Opportunity at AZMX","text":"Hi Example,\nWe are hiring..."}
```

```jsonl
{"profile_url":"https://www.linkedin.com/in/example-person","candidate_name":"Example Person","kind":"tag","text":"shortlist","author":null,"date":null}
```
