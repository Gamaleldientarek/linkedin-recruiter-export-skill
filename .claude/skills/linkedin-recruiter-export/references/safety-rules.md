# Safety Rules (always in force)

These rules override everything else in this skill. If any rule conflicts with making progress, the rule wins and progress stops.

## 1. Security checkpoint — absolute stop

**Detection signals** (check BEFORE every extraction step and after every navigation):

- Current URL contains `/checkpoint/`, `/uas/`, or `challenge`
- Page text contains any of: "security verification", "verify", "CAPTCHA", "unusual activity", "confirm it's you", "quick verification", "let's do a quick security check", "prove you're human"
- The page asks for a code, phone number, ID, puzzle, or any credential

**On detection, do ALL of the following and NOTHING else:**

1. Stop all browser actions immediately. Zero clicks, zero typing, zero scrolling on that page.
2. Update `state.json`: `run.status = "stopped_checkpoint"`, `run.stop_reason` = short description of what the page shows.
3. Tell the user exactly what page appeared and that they must resolve it manually in their browser.
4. Wait. Resume ONLY when the user explicitly says to continue, then re-verify login state before touching anything.

Never attempt to solve, bypass, retry, reload through, or automate any part of a verification page. This includes "just clicking OK". No exceptions.

## 2. Logged-out detection

Before starting and after any unexpected redirect: if the page shows a login form, "Sign in", or lacks the Recruiter navigation, stop and ask the user to log in themselves. Never type into a login form.

## 3. Pacing (human scale)

- Wait a randomized **2–5 seconds** between any two navigations or page-changing clicks.
- Every ~10 candidates processed, take a longer **8–12 second** rest.
- One page-changing action at a time; never fire parallel navigations.
- If any page loads slowly, wait for it — never hammer reload. One retry after a 10 s wait, then stop and report (rule 5).

## 4. Scope and data handling

- Touch only: the Recruiter projects list, the ONE chosen project's pipeline, its candidates' profile/message/note views. Nothing else — no other people's profiles, no search, no browsing.
- Never handle credentials. Never send data anywhere except local files under `exports/`.
- Only read what is visible to the logged-in account. Missing/hidden fields stay `null` — never guessed.

## 5. Fail loudly (never silently wrong)

If an expected structure cannot be found (projects list, pipeline rows, thread container, notes area):

1. Stop the run. Update `state.json`: `run.status = "stopped_error"`, `run.stop_reason = "cannot find <X> on <page>"`.
2. Tell the user exactly what could not be found and on which page — LinkedIn may have changed its UI.
3. Do not continue with partial/guessed extraction.

## 6. Interruption

Tab closed, session ended, user pressed Esc: nothing to clean up — JSONL appends are already on disk. The next invocation resumes from `state.json`. Never treat an interruption as a reason to rush or skip pacing afterwards.
