# LinkedIn Recruiter Lite Export — Claude Code Skill

A [Claude Code](https://claude.com/claude-code) skill that exports one LinkedIn **Recruiter Lite** project per run into an Excel workbook — candidates, full InMail threads, and notes & tags. Recruiter Lite has no native CSV export; this skill drives **your own logged-in Chrome** through the [Claude-in-Chrome](https://claude.com/chrome) extension, so no credentials are ever handled and nothing leaves your machine.

## What you get per run

`exports/<project>-<date>.xlsx` with three sheets:

| Sheet | Contents |
|---|---|
| **Candidates** | Name, LinkedIn profile URL (hyperlinked), headline, company, location, pipeline stage, date added, message/note counts |
| **Messages** | Every message per candidate — date, direction (`sent` / `received` / `scheduled`), subject, full text. Covers threads in any inbox state (Inbox, Awaiting Reply, Archived) |
| **Notes** | Every recruiter note (with author/date where visible) and tag, attributed by candidate |

Runs are **resumable** — raw data is saved incrementally to `exports/raw/`, so an interrupted run continues where it stopped instead of starting over.

## Safety posture

- Operates only on your own logged-in session, only on the one project you pick
- **Hard stop** on any LinkedIn security checkpoint / CAPTCHA — it hands you the browser and never attempts to bypass anything
- Human-paced navigation (seconds between page loads)
- No hardcoded CSS selectors — reads pages via accessibility structure, visible text, and link URLs
- Everything stays local; `exports/` is gitignored so candidate data can't be committed

Full rules: [`references/safety-rules.md`](.claude/skills/linkedin-recruiter-export/references/safety-rules.md)

## Installation

### Claude Code (native)

1. Clone this repo:
   ```bash
   git clone https://github.com/Gamaleldientarek/linkedin-recruiter-export-skill.git
   ```
2. Copy the skill folder to where you want it available:
   - **Globally** (all projects): `cp -r linkedin-recruiter-export-skill/.claude/skills/linkedin-recruiter-export ~/.claude/skills/`
   - **One project**: copy it into that project's `.claude/skills/` directory
3. Install the [Claude-in-Chrome extension](https://claude.com/chrome), and allow it on `linkedin.com`.
4. Make sure [`uv`](https://docs.astral.sh/uv/) is installed (used to run the workbook builder — no other Python setup needed).
5. Start a new Claude Code session, log into LinkedIn Recruiter Lite in Chrome, and say:
   > export my recruiter project

   or invoke it directly: `/linkedin-recruiter-export`

### ChatGPT

Claude skills don't run on ChatGPT natively — there's no skill system or Claude-in-Chrome equivalent to execute this as-is. What you *can* do: the whole procedure is plain Markdown. Give a browser-capable ChatGPT agent the contents of [`SKILL.md`](.claude/skills/linkedin-recruiter-export/SKILL.md) plus the two files in [`references/`](.claude/skills/linkedin-recruiter-export/references/) as instructions, and run [`scripts/build_workbook.py`](.claude/skills/linkedin-recruiter-export/scripts/build_workbook.py) yourself for the final Excel step (`uv run --with openpyxl build_workbook.py <raw-dir>`). Results depend entirely on that agent honoring the safety rules — the checkpoint-stop rule is not optional.

## Repo layout

```
.claude/skills/linkedin-recruiter-export/   the skill (SKILL.md + references + builder script)
specs/001-recruiter-lite-export/            full Spec Kit spec, plan, contracts, quickstart
tests/                                      offline tests for the workbook builder
```

Data contracts (raw JSONL layer, workbook format, skill interface): [`specs/001-recruiter-lite-export/contracts/`](specs/001-recruiter-lite-export/contracts/)

## Development

```bash
uv run --with openpyxl,pytest pytest tests/          # builder test suite
```

Validation scenarios: [`specs/001-recruiter-lite-export/quickstart.md`](specs/001-recruiter-lite-export/quickstart.md)

## Disclaimer

For exporting **your own** recruiting pipeline from **your own** account for legitimate internal use. Automated access to LinkedIn may conflict with LinkedIn's User Agreement — use at your own discretion and keep volumes at human scale. Not affiliated with LinkedIn.
