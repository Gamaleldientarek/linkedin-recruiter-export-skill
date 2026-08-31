# Contract: Skill Interface

## Invocation

- **Name**: `linkedin-recruiter-export`
- **Location**: `.claude/skills/linkedin-recruiter-export/SKILL.md`
- **Triggers**: user invokes `/linkedin-recruiter-export`, or asks to "export a Recruiter project", "export LinkedIn Recruiter Lite data", "pull my recruiter pipeline", etc. (description frontmatter must cover these phrasings).
- **Optional argument**: a project name — skips the project-selection prompt when it uniquely matches.

## Runtime preconditions (skill must verify, in order)

1. Claude-in-Chrome MCP tools available in the session (load via ToolSearch if deferred). If unavailable → stop with instructions to enable the extension. **Browser steps run in the main session only — never delegated to a subagent.**
2. A Chrome tab context is obtainable; a dedicated work tab is created for the run.
3. `linkedin.com/talent/home` loads showing a logged-in Recruiter Lite account. Logged-out or checkpoint page → stop, ask user to log in / resolve, offer resume.

## User interaction points (exactly these; everything else is autonomous)

| Point | When | Form |
|---|---|---|
| Project selection | Start of run (skipped if argument matched) | List project names, user picks one |
| Fresh vs incremental | Only when chosen project's raw state says `complete` (FR-015) | Two-option question |
| Checkpoint handoff | Any security/verification page (FR-010) | Stop + explain; user resolves manually; user explicitly says resume |
| Final summary | End of run (FR-013) | Project name, counts (candidates / messages / notes), output path, skipped candidates + reasons |

## Outputs

- `exports/raw/<project-slug>/` — per contracts/raw-data-schemas.md
- `exports/<project-slug>-<YYYY-MM-DD>.xlsx` — per contracts/workbook-format.md; `-HHMMSS` suffix if the name exists (never overwrite)

## Hard guarantees

- No credential input, storage, or handling (FR-001)
- No interaction of any kind with security-verification pages (FR-010, SC-005)
- No hardcoded obfuscated CSS selectors (FR-011); missing expected structure → loud stop naming what was not found (FR-012)
- Randomized 2–5 s inter-navigation delays; 8–12 s rest every ~10 candidates (FR-009)
- All data stays local (FR-014)
