# Extraction Guide — Recruiter Lite semantic waypoints

Recorded from live recon runs against the real Recruiter Lite UI. Waypoints are
semantic (URLs, headings, roles, visible labels) — never CSS class names, which
LinkedIn obfuscates and rotates.

> Recon 2026-09-01 (seat: AZM X People and Culture, Recruiter Lite): the seat
> had **zero projects**, so project-pipeline waypoints are not yet recorded.
> The InMail **inbox** was fully recon'd instead — inbox export is the
> operative mode for this seat. Re-run project recon if/when a project with
> candidates exists.

## Login and entry (verified 2026-09-01)

- Recruiter Lite home: `https://www.linkedin.com/talent/home`.
- Logged-in signal: page title "LinkedIn Talent Solutions", top nav reading
  "Recruiter Lite" with links "Create a project", "Post a free job", "Reports",
  and a left sidebar showing the seat name (e.g. "AZM X People and Culture").
- There is **no "Projects" link in the top nav**. The logo links to
  `/talent/hire`, which redirects back to `/talent/home`.
- A dismissible banner about not sharing Recruiter credentials may sit above
  the nav — ignore it; it is not a checkpoint.

## Projects list (verified 2026-09-01 — empty state only)

- Direct URL: `https://www.linkedin.com/talent/projects`.
- Page heading: `Projects (N)` — read N from this heading.
- Empty state (N = 0): illustration + heading "Create a project" + "Create new"
  button. On zero projects, do not stop dead: tell the user and offer the
  **inbox export** mode below.
- Project rows: not yet recorded (seat had zero projects at recon time).

## Inbox (verified 2026-09-01)

The inbox is a complete export source on its own: every InMail thread carries
the candidate's identity, profile link, and full message history, whether or
not any project exists.

### Folders

Left sidebar under heading "MY MESSAGES", a list of four links:

| Folder | URL | Contains |
|---|---|---|
| Inbox | `/talent/inbox/0/main` | threads where the candidate has replied |
| Awaiting Reply | `/talent/inbox/0/awaitingreply` | sent InMails with no reply yet (status "Pending") |
| Scheduled | `/talent/inbox/0/scheduled` | scheduled, not-yet-sent messages |
| Archived | `/talent/inbox/0/archived` | archived conversations |

A full inbox export walks **all four folders**. A candidate's thread lives in
exactly one folder at a time; the same person should still be deduped by
profile URL across folders.

### Thread list (per folder)

- Search box placeholder: "Search in <Folder>" (e.g. "Search in Awaiting
  Reply") — confirms which folder is active.
- Threads are `listitem`s in a list. Each row contains:
  - the candidate name (also on the row's avatar image),
  - a **link** whose accessible name is `"<Name>. Message preview: <text>"`
    with href `/talent/inbox/0/<folder>/id/<opaque-base64-id>` — this is the
    thread permalink; the trailing id uniquely identifies the thread,
  - a date label (e.g. "Aug 31"),
  - a status label where applicable (e.g. "Pending" in Awaiting Reply),
  - the preview text as plain generic text.
- Empty folder: Inbox shows "Your message threads will appear here when the
  candidates reply." and a "Get started with Inbox" panel pointing at the
  Awaiting Reply folder. Treat matching "no messages / will appear here" text
  as a legitimate empty folder, not an error.

### Thread view (candidate header)

Opening a thread shows an `article` whose header region contains:

- candidate name as a **link** to
  `https://www.linkedin.com/talent/profile/<PROFILE_ID>?trk=…` — **this is the
  canonical profile URL for candidates.jsonl** (strip the `?trk` query),
- optional verification badge button ("Profile verified…"),
- connection degree as text (e.g. "· 2nd"),
- headline text (may be truncated on screen; take what is rendered),
- location text (e.g. "Riyadh, Saudi Arabia"),
- industry text prefixed with "· " (e.g. "· Design Services").

Below the header: an "Activity" label and a button reading `"N message"` /
`"N messages"` — read N as the expected message count for the thread and use
it to verify extraction completeness.

### Messages within a thread

Messages are `listitem`s in a list inside the conversation application. Each
message region contains, in order:

- sender avatar + sender name (e.g. "AZM X People and Culture" or the
  candidate's name) — **direction rule**: sender name == seat name → 
  `outbound`; otherwise `inbound`,
- full date-time text (e.g. "August 31, 2026 at 4:24 PM"),
- subject line (bold, first text block — present on the first message of an
  InMail; may be absent on replies),
- body text (a sent-but-unanswered message is editable in place, so the body
  may sit in an element labeled "Click to edit the message body" with
  Edit/Delete buttons beside it — extract the rendered text, never click
  Edit/Delete),
- the sender's signature lines are part of the body text as rendered.

`get_page_text` on the thread view returns the full message text reliably;
use it to capture bodies, then attribute them using the accessibility tree
order.

### Thread states

- Awaiting-reply thread: footer shows heading "Sorry, you can't reply yet" and
  "You'll be able to reply to a candidate once they've accepted or replied to
  your message." — expected, not an error.
- "Add note" / "Add reminder" buttons appear under the thread — notes exist
  per candidate even without a project. Never click "Add reminder" or the
  call-panel controls; the export is read-only.

## Pipeline roster (project mode)

Not yet recorded — requires a project with candidates. To capture on next
recon: candidate row shape (name, headline, current company, location, stage,
date added), profile href shape, pagination vs lazy-load, end-of-list signal.

## Notes & tags

Partially observed: an "Add note" button exists on the thread view, implying
notes are readable there too. Shape of existing notes not yet recorded — needs
a candidate that already has a note.
