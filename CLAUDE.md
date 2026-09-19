# Day Tracker

A single-file web app that pulls today's events from Google Calendar and tracks, per event: whether you attended it, whether it was productive, and which tasks you worked on during it (with minutes logged). Tasks have their own lifecycle (status, description, optional parent/subtask hierarchy, full status-change history) independent of the calendar. Built to answer: "did I attend/was it productive, and can I compare across days" plus "which tasks am I actually spending time on."

## Architecture

- **Everything lives in `index.html`** — no build step, no bundler, no npm. HTML/CSS/JS in one file. Edit it directly.
- **No backend.** Static site hosted on GitHub Pages. All persistence is the user's own Google Sheet, written to directly from the browser via the Sheets API. Auth is Google Identity Services (`google.accounts.oauth2.initTokenClient`), a browser-only OAuth flow.
- **Repo**: `git@github.com:ravi-556/day-tracker.git`, deployed at **https://ravi-556.github.io/day-tracker/** via GitHub Pages (legacy/branch build mode, not Actions — pushing to `main` triggers a rebuild, but it doesn't happen instantly; poll `gh api repos/ravi-556/day-tracker/pages/builds/latest` or trigger one with `gh api -X POST repos/ravi-556/day-tracker/pages/builds`).
- **Local dev**: a plain `python3 -m http.server 8765` running from the parent `minimal_distraction/` directory (started via the sibling `start-no-feed.sh` script, or start it yourself) serves this file at `http://localhost:8765/day-tracker/index.html`. Both `http://localhost:8765` and `https://ravi-556.github.io` are registered as Authorized JavaScript origins on the OAuth client, so sign-in works from either.
- **OAuth Client ID** is hardcoded in the JS (`CLIENT_ID` constant) — this is safe to be public; it's a browser-only public client with no client secret, and Google enforces the origin allowlist server-side regardless of what's visible in the page source. The app is in Google's OAuth "Testing" publishing status (not verified/published), so only accounts added as test users in the Cloud Console can sign in, and everyone sees an "unverified app" warning to click through — this is expected and fine for personal use.

## Auth details and constraints

- Scopes: `calendar.readonly` + `spreadsheets`.
- Access tokens are short-lived (~1hr) **by Google's design for browser-only OAuth clients** — this app can never get a refresh token (those only exist in the server-side authorization-code flow), so re-authentication roughly once per hour of active use is an accepted, permanent constraint, not a bug to fix. Don't try to "fix" this without first discussing adding a real backend — that's the only actual way past it.
- The token is cached in `localStorage` (survives tab/browser close, not just the session) so reopening the app doesn't force sign-in unless the cached token has actually expired.
- Any API call returning 401 calls `handleAuthExpired()`, which clears the stale cached token and drops the user back to a clean sign-in screen — this replaced an earlier bug where an expired token caused unhandled promise rejections instead of a recovery path. If you add new API call sites, make sure they go through `apiFetch()` (which already handles this) and always attach a `.catch()` that shows a message via `showFeedback()` — several early call sites were missing this and failed silently; don't reintroduce that.

## Data model (Google Sheets, one per user)

The connected spreadsheet has three tabs, all schemas defined as header-row constants in the JS and enforced by `migrateSheetSchema()` (see below) — never hardcode column letters when a new field could shift them.

- **`Sheet1`** — the daily event log. One row per (date, calendar event) the user has touched. Columns: `Date, EventID, Title, Start, Attended, Productive, Tasks, UpdatedAt`. `Tasks` is a JSON blob: `[{taskId, done, minutes}]` — the link between an event and the tasks worked on during it, plus minutes logged for that specific task in that specific event.
- **`Tasks`** — the master task list. Columns (`TASKS_HEADER`): `TaskID, Name, Description, Status, ParentID, CreatedAt, UpdatedAt`. Status is one of `not_started` (labeled "Open" in the UI), `in_progress`, `paused`, `completed`. `ParentID` supports exactly **one level** of subtask nesting (a subtask can't itself have subtasks — enforced in the UI, not the data layer).
- **`TaskHistory`** — append-only log of every status change. Columns (`TASK_HISTORY_HEADER`): `TaskID, FromStatus, ToStatus, ChangedAt`. Never overwritten, only appended to — this is what powers the "chronology" shown on a task's detail page. A task's creation is logged here too (`FromStatus` empty).

**Schema migrations are automatic.** `migrateSheetSchema(tabName, expectedHeader)` reads whatever header a tab currently has, and if it doesn't match the expected one, remaps every existing row into the new column layout by field name (not position) and rewrites the tab. This means adding/reordering a column in `TASKS_HEADER` or `TASK_HISTORY_HEADER` is enough — existing user data migrates itself on next load. Don't ask users to manually clear their sheet when changing schema; that was the old (bad) approach before this existed.

**Time is derived, not stored on the task.** A task's "time spent" and "which events" are computed on demand by scanning all of `Sheet1`'s `tasks` links for a matching `taskId` (see `getTaskSessions`) — deliberately not duplicated onto the Tasks row, to avoid two sources of truth drifting apart.

## Google API gotchas worth remembering

- **Calendar events endpoint**: `calendarId` is a **path segment**, not a query param — `/calendar/v3/calendars/primary/events`, not `/calendar/v3/events?calendarId=primary`. Got this wrong once; the wrong version 404s on the CORS preflight itself, which shows up in Safari/WebKit consoles as a confusing "access control checks" error, not a normal 404.
- **Sheets custom-method endpoints** (`:append`, `:batchUpdate`, `:clear`) need the literal colon to stay **outside** anything passed through `encodeURIComponent`. `sheetRangeUrl()` encodes the range; always concatenate `':append'`/`':clear'` etc. *after* calling it, never include the verb inside the string you encode — encoding it turns `:` into `%3A` and Google's router won't recognize the method suffix, which again manifests as a CORS-looking failure rather than an obvious error.
- Deleting a task doesn't use `deleteDimension` (which would need the tab's numeric sheet ID and would shift every other cached row number). It just clears the row's values via `:clear`, and `loadTasks()` filters out any row with a blank ID. Simpler, avoids invalidating other tasks' cached `.row` numbers.

## UI structure

Three tabs: **Today** (calendar events with attended/productive toggles and a task-attach combo), **Tasks** (list with filters + a separate create page + a separate detail page), **History** (day-by-day attendance/productivity comparison table — this is a different "history" from `TaskHistory`, don't confuse the two).

- Task creation and task detail are **separate pages**, not modals, navigated via `showTaskCreatePage()` / `navigateToTaskDetail()` / `showTaskListPage()`. The detail page's Back button uses a real navigation stack (`taskDetailBackStack`) so it returns to wherever you actually came from (list, or another task's detail page via a subtask/parent link), not always the list.
- The task detail page is explicit-edit: Title/Status/Description are disabled until you click **Edit**, and nothing saves until **Save changes** — deliberately not autosave-on-keystroke (that was the original design and was rejected).
- On wide screens (≥1180px) the detail page is 3 columns: task details | subtasks + status history | time log. Below that it degrades to 2 columns, then 1 on mobile.
- Subtasks don't appear as their own rows in the flat Tasks list (only top-level tasks do, each showing a subtask count) — reachable via a parent's "Subtasks" panel or the event-attach search.
- The event-attach task combo only offers `not_started`/`in_progress` tasks (not paused/completed), and never offers a task that already has subtasks (those are containers, not directly-workable items).
- Completing every subtask under a parent auto-completes the parent (`maybeCompleteParent`), which also logs a `TaskHistory` entry for the parent.

## Known accepted limitations (don't "fix" without discussion)

- No push notifications — deliberately relies on Google Calendar's own native reminders instead of building a notification system.
- ~1hr forced re-auth — inherent to the backend-less architecture (see Auth section above).
- No delete for calendar events/history rows, only for tasks.
