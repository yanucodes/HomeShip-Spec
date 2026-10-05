# HomeShip — Domain Specification

What ships, crew, tasks and supplies are, and how their states change over time. Read `README.md` in this directory first: it defines rule IDs, settings, and the other conventions used here.

## Settings

| Setting | Default | Valid values | Meaning | Rules |
|---|---|---|---|---|
| `DAILY_ROLLOVER_HOUR` | `3` | integer 0–23 | Ship-local hour at which a new ship day begins. | `TIME-SHIP-DAY` |

## Time

All examples in this section use the default `DAILY_ROLLOVER_HOUR` of 3: a ship day starts at 03:00 ship-local time.

### TIME-SHIP-DAY — What a ship day is

Every ship has a timezone, given as an IANA timezone name (e.g. `Europe/Berlin`). A ship's days change at `DAILY_ROLLOVER_HOUR` ship-local time, not at midnight, so late-night activity counts toward the day it belongs to.

The ship day of an instant is:
- the ship-local calendar date, if the ship-local clock reads `DAILY_ROLLOVER_HOUR`:00 or later;
- the previous calendar date otherwise.

Equivalently: subtract `DAILY_ROLLOVER_HOUR` hours from the ship-local clock time and take the date.

Ship timezone `Europe/Berlin` (UTC+1 in January):

| Instant (UTC) | Ship-local time | Ship day |
|---|---|---|
| 2026-01-15 01:59:59 | 2026-01-15 02:59:59 | 2026-01-14 |
| 2026-01-15 02:00:00 | 2026-01-15 03:00:00 | 2026-01-15 |
| 2026-01-14 23:30:00 | 2026-01-15 00:30:00 | 2026-01-14 |
| 2026-01-15 22:30:00 | 2026-01-15 23:30:00 | 2026-01-15 |

The same instant (2026-01-15 02:30 UTC) falls on different ship days for different ships:

| Ship timezone | Ship-local time | Ship day |
|---|---|---|
| `Europe/Berlin` (UTC+1) | 2026-01-15 03:30 | 2026-01-15 |
| `America/New_York` (UTC−5) | 2026-01-14 21:30 | 2026-01-14 |
| `Asia/Tokyo` (UTC+9) | 2026-01-15 11:30 | 2026-01-15 |
| `UTC` | 2026-01-15 02:30 | 2026-01-14 |

### TIME-SHIP-DAY-DST — Daylight saving time changes only a ship day's length

Ship-local time follows the timezone's own daylight saving rules. A clock change makes the ship day it falls in shorter or longer in real time; no ship day is skipped or repeated.

| Ship day (`Europe/Berlin`) | Clock change that night | Starts | Ends | Length |
|---|---|---|---|---|
| 2026-03-28 | 02:00 CET → 03:00 CEST | 03:00 CET (02:00 UTC) | 2026-03-29 03:00 CEST (01:00 UTC) | 23 h |
| 2026-10-24 | 03:00 CEST → 02:00 CET | 03:00 CEST (01:00 UTC) | 2026-10-25 03:00 CET (02:00 UTC) | 25 h |

Instants around the changes:

| Instant (UTC) | Ship-local time | Ship day |
|---|---|---|
| 2026-03-29 00:59:59 | 01:59:59 CET | 2026-03-28 |
| 2026-03-29 01:00:00 | 03:00:00 CEST | 2026-03-29 |
| 2026-10-25 00:30:00 | 02:30 CEST | 2026-10-24 |
| 2026-10-25 01:30:00 | 02:30 CET (the repeated hour) | 2026-10-24 |
| 2026-10-25 02:00:00 | 03:00 CET | 2026-10-25 |

### TIME-TODAY — "Today" is the ship's current ship day

Wherever this spec says "today", it means the ship day (`TIME-SHIP-DAY`) of the current instant for the ship concerned. This applies to everything: actions by crew members, validation of dates they submit, and the daily advance. The server's own timezone never affects any result.

*Example:* a crew member of a `Europe/Berlin` ship acts at 00:30 local time on 2026-01-15. For that action, today is 2026-01-14.

### TIME-DATES — Dates are ship days; no times are recorded

Every date the API accepts or returns for a ship or its items (e.g. a due date) is a ship day (`TIME-SHIP-DAY`) of that ship. The API records and returns dates only: no times of day, no timezones.

## Alert states

Every task and every supply has exactly one alert state at any time. Alert states are a vocabulary shared by tasks and supplies, so a ship's overall state is computed the same way from both. How an item's alert state is determined is defined per item type (see Tasks, Supplies); what alert states do to the ship is defined under Ships.

### ALERT-LEVELS — The alert states and their order

| Alert state | Meaning |
|---|---|
| green | On track. |
| yellow | Needs attention. |
| red | Critical. |
| auto-destruct | Self-destruct sequence running: unless the item is resolved in time, the ship's journey is lost. |
| inactive | Muted: the item still exists but is not tracked, and has no effect on the ship. |

The tracked states are ordered by severity: green < yellow < red < auto-destruct. Inactive is outside this order: it is neither better nor worse than any other state.

### ALERT-DERIVED — Alert states are never set directly

No one sets an item's alert state directly. It always follows from the item's own data, the crew's actions on it (such as postponing a task or changing a supply's stock), and the passing of days, as defined per item type.

### ALERT-ESCALATION — Escalating raises an alert by one level

Escalating moves an alert one step up the severity order. Auto-destruct is the top of the order and stays auto-destruct; inactive is outside the order and stays inactive. Which events escalate an item is defined per item type.

| Before | After escalating |
|---|---|
| green | yellow |
| yellow | red |
| red | auto-destruct |
| auto-destruct | auto-destruct |
| inactive | inactive |
