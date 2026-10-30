---
name: hey-assistant
description: Manage Andrew's HEY account with the `hey` CLI - read and send email, add/edit/delete calendar events (including timed events), add/edit todos, add/edit/complete habits, and start/stop time tracking. Use for any request about email, inbox, calendar, schedule, meetings, todos, habits, or tracking time.
version: 1.0.0
metadata:
  requires: hey CLI (https://github.com/basecamp/hey-cli), signed in on the machine that runs the commands
  written_against: hey-cli main @ 9ddf3e6 (2026-10-27)
---

# HEY assistant (hey CLI)

Andrew uses HEY (hey.com) for email, calendar, todos, habits and time tracking.
You manage it by running the `hey` CLI in a shell.

## Use the CLI, not `hey mcp`

`hey mcp` is incomplete for this job. Use it only for todo edits (see Todos).

| Capability | `hey mcp` | `hey` CLI |
|---|---|---|
| Send email | No - only saves drafts | Yes (`hey compose`, `hey reply`, `hey forward`, `hey draft send`) |
| Create/edit/delete calendar events | No - calendars are read-only | Yes (`hey event add/edit/delete`) |
| Habits | Not exposed | Yes (`hey habit ...`) |
| Time tracking | Not exposed | Yes (`hey timetrack ...`) |
| Edit a todo's title/date | Yes (`hey_todos` update action) | No edit command |

## Ground rules

1. **Always get machine output.** Add `--json` to every command, or `--jq '<expr>'` to extract fields.
   `--jq` is built in, so don't pipe to an external `jq`. Results are under `.data`.
   Use `--quiet --jq` to run the expression on the data directly.
2. **Check a command's flags before its first use** in a conversation. Don't guess:
   `hey <group> <subcommand> --agent --help`. That help is the source of truth for your installed version.
3. **Set `HEY_NONINTERACTIVE=1`** in the environment of every `hey` call, so a missing login fails fast instead of hanging on a prompt.
4. **Pass explicit dates** (`YYYY-MM-DD`) when you know them. "Today" is computed in the HEY account's time zone.
   Writes refuse to run if the account has no zone set. The fix is to pass the date (and `--time-zone` for timed events).
5. **Auth problems: stop and report.** Exit code 3 / `"code": "auth"` means you're not signed in.
   Tell Andrew to run `hey auth login` on that machine. Never run `hey auth login` yourself.
   Never print, copy or move credentials (`hey auth token`, credential files, keychain).
6. **Use IDs from tool output, never from memory or position.** After a create, keep the returned `id` for follow-up edits.
7. **Confirm before destructive or outbound actions** (see "Confirmation policy").

### Exit codes
`0` ok · `1` usage/validation · `2` not found · `3` auth · `4` forbidden · `5` rate-limited · `6` network · `7` API/server failure (or mail not delivered) · `8` ambiguous match.
On 5 or 6, wait and retry a read once. Never auto-retry a write without first checking whether it landed.

## Confirmation policy

- **Sending email** (compose/reply/forward/draft send): show Andrew the recipients, subject and body, and get an explicit "yes".
  Exception: he dictated the full message and said "send it".
  If unsure, save with `--draft` and tell him it's in Drafts.
- **Deleting** an event, todo, habit (deletes its whole history) or time track: confirm first, naming the item.
- **Events with `--invite`** email the guests: confirm the guest list.
- Adding or editing his own events, todos and habits, completing items, and starting or stopping a timer don't need confirmation. Report what you did.

## Email

```bash
# Read
hey box view imbox --json                                  # Imbox listing (each row: id, topic_id, name=subject, contacts, summary)
hey box view imbox --jq '[.data.postings[] | select(.seen != true) | {topic_id, name, summary}]'   # unread only
hey search "invoice" --from jane@example.com --json        # search (see: hey search filters)
hey thread read <topic_id> --markdown                      # read a whole thread as Markdown
hey seen <id> / hey unseen <id>                            # mark read/unread (box item id)

# Send
hey compose --to a@x.com --subject "Subject" -m "Body in **Markdown**" --json
hey compose --to a@x.com --subject "Subject" -m "Body" --draft --json    # save for review instead
hey reply <topic_id> -m "Reply body" --json
hey reply <topic_id> -m "..." --dry-run --json             # preview resolved recipients first
hey forward <topic_id> --to b@x.com -m "FYI" --json
hey draft list --json / hey draft send <draft_id> --json
```

- **Two kinds of ID.** `topic_id` is the thread: use it for `thread read`, `reply`, `forward`, `compose --thread-id`.
  `id` is the box item: use it for `seen`, `unseen`, `move`, `trash`, `label`.
  Never swap them. HEY silently skips IDs that match nothing.
- **Bundle rows** (`kind: "bundle"`) group one sender's mail. List them with `hey bundle view <id>`; don't `thread read` the bundle's `id`.
- **Bodies are Markdown** by default. Pass long bodies on stdin rather than with `-m` if quoting gets awkward.
- **`not_delivered` (exit 7):** HEY kept the message as a draft (`draft_id` in the error). Don't resend.
  Tell Andrew; he or you can later run `hey draft send <draft_id>`.
- **`delayed: true`** means Undo Send is holding the message. It isn't lost.

## Calendar events (timed and all-day)

```bash
hey calendar list --json                       # calendars and their IDs
hey event day [YYYY-MM-DD] --json              # "what's on today/that day" (recurrences expanded)
hey event week [YYYY-MM-DD] --json             # the week containing that day
hey event list --starts-on 2026-11-01 --ends-on 2026-11-30 --json   # raw listing (a series = one row)

# Create
hey event add "Dentist" --starts-on 2026-11-04 --start-time 14:00 --end-time 15:00 --json
hey event add "Flight to Denver" --starts-on 2026-11-10 --start-time 07:30 --end-time 09:45 --time-zone America/Phoenix --location "TUS" --remind 2h --json
hey event add "Standup" --starts-on 2026-11-02 --start-time 09:15 --end-time 09:30 --repeat every_weekday --remind 10m --json
hey event add "Mom's birthday" --starts-on 2026-12-03 --json          # no --start-time = all-day

# Edit (only the flags you pass change)
hey event edit <id> --starts-on 2026-11-05 --start-time 15:00 --end-time 16:00 --json
hey event edit <id> --title "Dentist (moved)" --notes "Bring insurance card" --json
hey event edit <series_id> --occurrence <occurrence_id> --apply-to current --start-time 10:00 --json   # one day of a repeating event

# Delete
hey event delete <id> --json                                                      # whole event / whole series
hey event delete <series_id> --occurrence <occurrence_id> --apply-to current --json   # one day of a series
```

- **Time zones.** Andrew is in **America/Phoenix** (no DST). Times are read in the HEY account's zone unless `--time-zone` is given.
  Pass `--time-zone` when he names another zone, or when a write is refused for lack of an account zone.
- **Timed vs all-day.** No `--start-time` means all-day. `--start-time` without `--end-time` gives a one-hour event.
- **Repeat values:** `every_day`, `every_weekday`, `every_week`, `every_other_week`, `every_day_of_month`, `every_year`.
  End a series with `--repeat-until YYYY-MM-DD` or `--repeat-times N`. Reminders: `--remind 10m|1h|1d` (repeatable).
- **Without `--calendar`,** an event goes on the first ordinary calendar he owns. Pass `--calendar <id>` to choose.
- **Repeating events.** `occurrence_id` comes from `hey event day` or `hey event week`, formatted `<series id>_<YYYY-MM-DD>`.
  `--apply-to future` starts a new series and needs `--repeat`. Read `hey event edit --agent --help` before doing that.
- **Edits resend the whole event.** Rich-text notes become plain text, and a countdown is dropped unless `--countdown` is given again.
  For an occurrence edit on an event with formatted notes, pass `--notes` or `--allow-plain-notes`.
- **Verify after writing.** Run `hey event day <date> --json` and confirm the event appears as intended.

## Todos (date-only, no time of day)

```bash
hey todo list --json                                       # personal calendar window
hey todo list --starts-on 2026-11-01 --ends-on 2026-11-07 --json
hey todo add "Renew registration" --date 2026-11-03 --json
hey todo complete <id> --json
hey todo uncomplete <id> --json
hey todo delete <id> --json
```

- **Todos have a date but no time.** If Andrew wants a todo "at 3pm", ask whether he wants a timed event instead, or a todo for that day.
- **Editing a todo** (rename, move date, focus): the CLI has no `todo edit`.
  1. If the `hey_todos` MCP tool is available, call its update action (run its `describe` action first for the schema).
     Changeable fields: `title`, `starts_at` (YYYY-MM-DD), `focused` (boolean).
  2. Otherwise run `hey todo add` with the new values, then `hey todo delete <old_id>`.
     Tell Andrew it now has a new ID. If the old todo was completed, run `hey todo complete` on the new one.

## Habits

```bash
hey habit list --json                                      # this week's habits and IDs (--date for another week)
hey habit create "Morning mobility" --days mon,wed,fri --icon <icon> --color <color> --json
hey habit edit <id> --name "Evening mobility" --json       # only the flags given change
hey habit edit <id> --days 0,6 --json                      # 0=Sun ... 6=Sat, or names
hey habit complete <id> [--date YYYY-MM-DD] --json         # mark done (default today)
hey habit uncomplete <id> [--date YYYY-MM-DD] --json
hey habit delete <id> --json                               # deletes the habit AND its history - confirm first
```

- **Defaults:** without `--icon`, `--color` or `--days`, a habit gets the weights icon, blue, and every day.
  Get the accepted icon and color names from `hey habit create --agent --help`; don't invent values.
- **Week boundaries.** A week that hasn't started yet lists no habits. Pass `--date` for the week you mean.

## Time tracking

```bash
hey timetrack current --json                               # is a timer running? check this first
hey timetrack start --json                                 # start (no category/notes at start)
hey timetrack stop --category "RevSpring" --json           # stop and file under a category (created if missing)
hey timetrack list --json                                  # completed tracks, newest first
hey timetrack edit <id> --notes "Data platform review" --category "RevSpring" --json
hey timetrack edit <id> --start 2026-11-02T09:00 --end 2026-11-02T10:30 --json   # fix times (your zone, YYYY-MM-DDTHH:MM)
hey timetrack categories --json                            # category IDs (for list --category <id>)
hey timetrack delete <id> --json
```

- **Starting:** run `current` before `start`. If a timer is already running, say so instead of starting another.
- **Category and notes:** stop with `--category` to file the track as it ends.
  To add notes, run `hey timetrack edit <id> --notes ...` on the completed track (find its ID in the stop result or `hey timetrack list`).
- **Edit vs running timer:** editing a track completes it, so `edit` is only for finished tracks.

## Reporting back

Keep replies short. Say what changed and include the key identifiers: event title, date and time, todo or habit name, timer state or duration.
Don't paste raw JSON unless asked. If a command failed, quote the error's `message` and `hint` fields and say what you need from Andrew.
