# HomeShip — Domain Specification

What ships, crew, tasks and supplies are, and how their states change over time. Read `README.md` in this directory first: it defines rule IDs, settings, and the other conventions used here.

## Settings

| Setting | Default | Valid values | Meaning | Rules |
|---|---|---|---|---|
| `DAILY_ROLLOVER_HOUR` | `3` | integer 0–23 | Ship-local hour at which a new ship day begins. | `TIME-SHIP-DAY` |
| `DEFAULT_POSTPONE_DAYS` | `3` | integer 1–3650 | Days a task's due date moves out when it goes up a level without the crew choosing a new date. Also the least a recurring task can be postponed by. | `TASK-MISSED`, `TASK-POSTPONE` |
| `SUPPLY_DEADLINE_RED_DAYS` | `1` | integer 1–365, at most `SUPPLY_DEADLINE_YELLOW_DAYS` | A deadline supply that isn't in stock is red once its deadline is at most this many days away. | `SUPPLY-ALERT-DEADLINE` |
| `SUPPLY_DEADLINE_YELLOW_DAYS` | `7` | integer 1–365, at least `SUPPLY_DEADLINE_RED_DAYS` | A deadline supply that isn't in stock is yellow once its deadline is at most this many days away (and it isn't red). | `SUPPLY-ALERT-DEADLINE` |

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

A date is **past** (in the past) when it's before today, and **future** (in the future) when it's after today. Today itself is neither.

### TIME-DATES — Dates are ship days; no times are recorded

Every date the API accepts or returns for a ship or its items (e.g. a due date) is a ship day (`TIME-SHIP-DAY`) of that ship. The API records and returns dates only: no times of day, no timezones.

## Alert states

Every task and every supply has exactly one alert state at any time. Alert states are a vocabulary shared by tasks and supplies, so a ship's overall state is computed the same way from both. How an item's alert state is determined is defined per item type (see Tasks, Supplies); what alert states do to the ship is defined under Ships (`SHIP-CONDITION`, `SHIP-SPEED`, `SHIP-DESTROY`).

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

## Tasks

A task is a household chore the crew keeps track of, e.g. "Laundry" or "Water the plants".

### TASK-FIELDS — What a task records

| Field | Meaning | Values |
|---|---|---|
| `content` | What the task is. | Text. Surrounding whitespace is removed; what remains is 1–200 characters, counted as Unicode code points. Several tasks may have the same content. |
| `frequency` | How often the task recurs. | A whole number of days from 1 to 3650; or none. |
| `date_last` | The ship day the task was last completed. | A date, or none if it hasn't been completed. |
| `date_due` | The ship day by which the task should next be completed. | A date, or none. A due date the crew provides (creating, postponing, reactivating) is at most 3650 days after today; otherwise the request is refused. |
| `alert_state` | How urgent the task is (`ALERT-LEVELS`). | Derived (`ALERT-DERIVED`) by the rules in this section. |
| `days_left` | Only while the task is paused: the days that were left until its due date when it was deactivated. | An integer ≥ 0, or none. |
| `frozen_alert` | Only while the task is paused: the alert state it had when it was deactivated. | green, yellow or red; or none. |

A task with a frequency is **recurring**; a task without one is **one-off**.

### TASK-TRACKED — A task is tracked exactly when its alert state isn't inactive

A task is **tracked** when its alert state is green, yellow, red or auto-destruct, whether it's recurring or one-off. It's **inactive** when its alert state is inactive. The alert state alone decides this; the other fields follow from it:
- A tracked task always has a due date. An inactive task never has one.
- An inactive task is either **paused** or not. Deactivating a task pauses it (`TASK-DEACTIVATE`). It stays paused until it's reactivated (`TASK-REACTIVATE`), or, if it's one-off, completed (`TASK-COMPLETE`). Other inactive tasks, such as a one-off created without a due date or a completed one-off, aren't paused.
- A paused task has both frozen values, `days_left` and `frozen_alert`. Every other task, tracked or inactive, has neither.

| Task | `frequency` | `alert_state` | `date_due` | `days_left`, `frozen_alert` |
|---|---|---|---|---|
| Recurring, tracked | a number of days | green, yellow, red or auto-destruct | a date | none |
| One-off, tracked | none | green, yellow, red or auto-destruct | a date | none |
| Recurring, inactive | a number of days | inactive | none | set if paused, otherwise none |
| One-off, inactive | none | inactive | none | set if paused, otherwise none |

In every row, `date_last` is a date if the task has been completed before, and none otherwise.

A tracked task is green exactly when it's on schedule: it hasn't been postponed or gone overdue since it was created, completed or rescued (`TASK-FREQUENCY` row 3).

### TASK-DATE-LAST — `date_last` records only real completions

`date_last` is set only when the crew provides it while creating the task (`TASK-CREATE`), or when the task is completed (`TASK-COMPLETE`). No other action sets, changes or clears it, in particular deactivation, reactivation and frequency changes.

### TASK-DUE-AFTER-LAST — A task is never due on or before its last completion

Whenever a task has both a `date_due` and a `date_last`, the due date is later than `date_last`. A request that would break this is refused and changes nothing.

### TASK-CREATE — Creating a task

The crew provides `content` and, optionally, `frequency`, `date_last` and `date_due`. The request is refused, and no task is created, if:
- a provided `date_last` is in the future;
- a provided `date_due` is in the past, or more than 3650 days after today (`TASK-FIELDS`);
- a provided `date_due` isn't later than a provided `date_last` (`TASK-DUE-AFTER-LAST`);
- the task is recurring, its scheduled date (below) is past, and no `date_due` is provided;
- the task is recurring, its scheduled date is past, and the provided `date_due` is more than the task's postpone limit after today (`TASK-POSTPONE`).

These checks apply to a provided `date_due` even when the rules below then ignore it.

A provided `date_last` is kept, for recurring and one-off tasks alike.

**One-off task** (no frequency):
- With a `date_due`: tracked, green, due on that date.
- Without one: inactive, not paused.

**Recurring task** (a frequency is given). Its scheduled date S is `date_last` + frequency, or today + frequency if no `date_last` is given:

| S | `date_due` after | `alert_state` after |
|---|---|---|
| today or later | S; a provided `date_due` is ignored | green |
| past | the provided `date_due` (required) | yellow, however far in the past S is |

Examples, with today = 2026-01-20 and the default `DEFAULT_POSTPONE_DAYS` of 3:

| `frequency` | `date_last` | `date_due` given | S | Result |
|---|---|---|---|---|
| 7 | 2026-01-15 | – | 2026-01-22 | green, due 2026-01-22 |
| 7 | 2026-01-15 | 2026-01-25 | 2026-01-22 | green, due 2026-01-22 (given date ignored) |
| 7 | 2026-01-15 | 2026-01-18 | 2026-01-22 | refused: due date in the past, even though it would be ignored |
| 7 | – | – | 2026-01-27 | green, due 2026-01-27 |
| 7 | 2026-01-12 | 2026-01-25 | 2026-01-19 | yellow, due 2026-01-25 |
| 7 | 2026-01-01 | 2026-01-25 | 2026-01-08 | yellow, due 2026-01-25 |
| 7 | 2026-01-12 | – | 2026-01-19 | refused: S is past, so a due date is required |
| 7 | 2026-01-12 | 2026-01-28 | 2026-01-19 | refused: more than 7 days after today |
| 7 | 2026-01-12 | 2026-01-18 | 2026-01-19 | refused: due date in the past |
| 1 | 2026-01-15 | 2026-01-23 | 2026-01-16 | yellow, due 2026-01-23 (postpone limit 3 days) |
| 1 | 2026-01-15 | 2026-01-24 | 2026-01-16 | refused: more than 3 days after today |
| – | – | 2026-01-25 | – | green, due 2026-01-25 |
| – | – | 2026-01-18 | – | refused: due date in the past |
| – | 2026-01-15 | 2026-01-15 | – | refused: due date not after `date_last` |
| – | – | – | – | inactive |

### TASK-COMPLETE — Completing a task

Completing is allowed at any time, whatever the task's state. `date_last` becomes today. Then:

| Task before | Result |
|---|---|
| recurring, tracked (any alert) | due today + frequency, green. An auto-destruct task is resolved (`SHIP-RESOLVED`). |
| one-off, tracked (any alert) | inactive, not paused; `date_due` none. An auto-destruct task is resolved (`SHIP-RESOLVED`). |
| inactive, not paused | nothing else changes. |
| recurring, paused | stays paused; `days_left` becomes `frequency` and `frozen_alert` green. |
| one-off, paused | stays inactive, no longer paused: both frozen values become none. |

For a paused task this is the same result as completing it while tracked and deactivating it the same day. (Leaving the frozen values as they were could make a reactivated task due on its own `date_last`, which `TASK-DUE-AFTER-LAST` forbids.)

Completing a task again on the same day gives the same result as completing it once.

*Example:* a weekly task, paused with `days_left` 2 and `frozen_alert` yellow. Completed on 2026-12-10: `date_last` 2026-12-10, `days_left` 7, `frozen_alert` green. Reactivated on 2026-12-12: due 2026-12-19, green.

### TASK-POSTPONE — Postponing a task

Postponing is for a task the crew can't do on the day it's due.
- Allowed only on the day the task is due (`date_due` is today).
- The crew picks the new due date, which must be later than today. For a recurring task it can be at most the task's **postpone limit** after today: `frequency` days, or `DEFAULT_POSTPONE_DAYS` days if that's more. For a one-off task, only the general 3650-day limit applies (`TASK-FIELDS`).
- The alert goes one level up (`ALERT-ESCALATION`): green → yellow, yellow → red.
- Refused for red and auto-destruct tasks ("no postpones left"), for inactive tasks, and on any day other than the due date. A refused postpone leaves the task unchanged.

The postpone limit is never less than `DEFAULT_POSTPONE_DAYS`, so postponing a task gives at least as much time as missing its due date does (`TASK-MISSED`).

Examples, with the default `DEFAULT_POSTPONE_DAYS` of 3. A weekly task (postpone limit 7 days), green, due 2026-01-20:

| Today | Request | Result |
|---|---|---|
| 2026-01-18 | postpone to 2026-01-22 | refused: not due today |
| 2026-01-20 | postpone to 2026-01-28 | refused: more than 7 days after today |
| 2026-01-20 | postpone to 2026-01-27 | yellow, due 2026-01-27 |
| 2026-01-27 | postpone to 2026-01-29 | red, due 2026-01-29 |
| 2026-01-29 | postpone to 2026-01-30 | refused: red |

A daily task (postpone limit 3 days), green, due 2026-01-20:

| Today | Request | Result |
|---|---|---|
| 2026-01-20 | postpone to 2026-01-24 | refused: more than 3 days after today |
| 2026-01-20 | postpone to 2026-01-23 | yellow, due 2026-01-23 |

### TASK-FREQUENCY — Changing a task's frequency

The crew sets a new frequency, or none. D' = current `date_due` + (new − old frequency). R = `date_last` + new frequency. Rows 1–4 apply when the new frequency is a number of days different from the old one.

| # | Task before | Condition | `date_due` after | `alert_state` after |
|---|---|---|---|---|
| 1 | recurring, tracked, green | D' is today or later | D' | green |
| 2 | recurring, tracked, green | D' is past | unchanged | yellow |
| 3 | recurring, tracked, yellow/red/auto-destruct; or one-off, tracked (any alert), getting a frequency | has a `date_last`, and R is today or later | R | green (an auto-destruct task is resolved) |
| 4 | same as row 3 | otherwise | unchanged | unchanged |
| 5 | recurring, tracked, frequency removed | — | unchanged | unchanged |
| 6 | inactive (paused or not) | — | stays none; frozen values unchanged | inactive |
| 7 | any | new = old | unchanged | unchanged |

The new frequency is saved in every row. No row can make a task due on or before its `date_last` (`TASK-DUE-AFTER-LAST`).

Examples for rows 1–2. A green task, frequency 10, due 2026-02-10:

| Today | New frequency | D' | Result |
|---|---|---|---|
| 2026-02-05 | 7 | 2026-02-07 | green, due 2026-02-07 |
| 2026-02-08 | 7 | 2026-02-07 | yellow, due 2026-02-10 |
| 2026-02-08 | 2 | 2026-02-02 | yellow, due 2026-02-10 |

Examples for rows 3–4. A weekly task completed 2026-02-01, so due 2026-02-08. It's missed twice (`TASK-MISSED`): yellow, due 2026-02-11, then red, due 2026-02-14. Today is 2026-02-12:

| New frequency | R | Result |
|---|---|---|
| 14 | 2026-02-15 | green, due 2026-02-15 |
| 11 | 2026-02-12 | green, due 2026-02-12 |
| 10 | 2026-02-11 | unchanged: red, due 2026-02-14 |

The same task without a `date_last` (created, never completed) is unchanged by any new frequency (row 4).

Examples for rows 3–4 with a one-off. Completed 2026-02-01, then reactivated with a new due date, 2026-02-20 (green). Today is 2026-02-12:

| Frequency it gets | R | Result |
|---|---|---|
| 14 | 2026-02-15 | green, due 2026-02-15 |
| 7 | 2026-02-08 | unchanged: green, due 2026-02-20 |

*Example:* a one-off created on 2026-01-01 with `date_last` 2026-01-01, due 2026-01-30. The same day it gets frequency 30: R = 2026-01-31, so it's green, due 2026-01-31 (row 3). Then frequency 1: D' = 2026-01-02, so green, due 2026-01-02 (row 1).

### TASK-DEACTIVATE — Deactivating pauses a task

Deactivating a tracked task:
- `alert_state` becomes inactive, and `date_due` becomes none.
- `days_left` becomes the number of days from today to the old due date, and `frozen_alert` becomes the old alert state.
- An auto-destruct task is frozen as red with 0 days left. Deactivating it resolves it (`SHIP-RESOLVED`).
- `frequency` and `date_last` don't change.

Deactivating is idempotent: deactivating a task that's already inactive changes nothing, and the request succeeds.

*Example:* frequency 10, last completed 2026-12-05, due 2026-12-15, green. Deactivated on 2026-12-10: `days_left` 5, `frozen_alert` green.

### TASK-REACTIVATE — Reactivating makes an inactive task tracked again

The request may include a `date_due`.

**A paused task** resumes where it was paused:
- `alert_state` becomes `frozen_alert`, and `date_due` becomes today + `days_left`.
- Both frozen values become none.
- A provided `date_due` is ignored.

*Example:* the task above, reactivated on 2026-12-15: due 2026-12-20, green.

*Example:* a task deactivated at auto-destruct (frozen as red, 0 days left), reactivated on 2026-12-15: due 2026-12-15, red. Unless it's completed that day, it reaches auto-destruct at the next daily advance.

**A task that isn't paused** (a one-off created without a due date, a completed one-off, or such a task after getting a frequency) is scheduled as if it were being created now, with its current `frequency` and `date_last` and the provided `date_due` (`TASK-CREATE`):
- One-off: a `date_due` is required; the task becomes green, due on it.
- Recurring: S = `date_last` + frequency, or today + frequency without a `date_last`. If S is today or later: green, due S. If S is past: a `date_due` is required, at most the task's postpone limit after today (`TASK-POSTPONE`); the task becomes yellow, due on it.
- The `TASK-CREATE` checks on a provided `date_due` apply. If a required `date_due` is missing, the request is refused.

This is also how the crew sets a new deadline on a one-off that was completed or created without one.

*Example:* a one-off completed on 2026-12-01. Reactivated on 2026-12-10 with `date_due` 2026-12-20: green, due 2026-12-20. Without a `date_due`: refused.

*Example* (default `DEFAULT_POSTPONE_DAYS` of 3): a recurring task that isn't paused, frequency 7, last completed 2026-12-01. Reactivated on 2026-12-05: green, due 2026-12-08. Reactivated on 2026-12-10: S = 2026-12-08 is past, so a `date_due` between 2026-12-10 and 2026-12-17 is required; with 2026-12-12, it's yellow, due 2026-12-12.

Reactivating is idempotent: reactivating a task that's already tracked changes nothing, and the request succeeds.

### TASK-MISSED — A missed due date at the daily advance

At the daily advance that starts ship day D, every tracked task due before D goes one level up, except an auto-destruct one:
- green → yellow and yellow → red: the due date moves `DEFAULT_POSTPONE_DAYS` days later.
- red → auto-destruct: the due date becomes D. The countdown runs during D; unless the task is resolved on D, the ship is destroyed at the next advance (`SHIP-DESTROY`).

Later advances don't change an auto-destruct task: it stays auto-destruct, and its due date stays its countdown day, even after that day has passed. Inactive tasks aren't affected.

Examples, with the default `DEFAULT_POSTPONE_DAYS` of 3, at the advance that starts 2026-01-21:

| Task before | Result |
|---|---|
| green, due 2026-01-21 | unchanged |
| green, due 2026-01-20 | yellow, due 2026-01-23 |
| yellow, due 2026-01-20 | red, due 2026-01-23 |
| red, due 2026-01-20 | auto-destruct, due 2026-01-21 |
| auto-destruct, due 2026-01-15 | unchanged |

The daily advance takes effect at the very start of the ship day: no action on ship day D sees the ship as it was before D's advance. Every ship day gets its own advance, in order: advances that were missed (e.g. during an outage) are made up one ship day at a time, before anything else (`SHIP-ADVANCE`). So when an advance reaches a task, it's at most one day past due, and whenever the crew acts, every tracked task except an auto-destruct one is due today or later.

*Example:* a green task due 2026-01-10. No advance runs from 2026-01-11 until 2026-01-21, when the missed advances are made up in order:

| Advance | Task after |
|---|---|
| 2026-01-11 | yellow, due 2026-01-13 |
| 2026-01-14 | red, due 2026-01-16 |
| 2026-01-17 | auto-destruct, due 2026-01-17 |
| 2026-01-18 to 2026-01-21 | unchanged (the ship is destroyed at the 2026-01-18 advance, `SHIP-DESTROY`) |

### TASK-EDIT — Editing a task's content

`content` can be edited at any time, whatever the task's state; nothing else changes.

### TASK-DELETE — Deleting a task

Deleting removes the task, whatever its state. Deleting an auto-destruct task resolves it (`SHIP-RESOLVED`).

## Supplies

A supply is something the household buys: either something it keeps in stock, e.g. milk or shampoo, or something it needs by a date, e.g. candles for a birthday cake.

### SUPPLY-FIELDS — What a supply records

| Field | Meaning | Values |
|---|---|---|
| `name` | What the supply is. | Text. Surrounding whitespace is removed; what remains is 1–200 characters, counted as Unicode code points. Several supplies may have the same name. |
| `stock_state` | How much the household has. | in stock, running low, or out of stock. |
| `quantity` | How many units are on hand, as the crew counts them. Informational only: it never affects the stock state or the alert. | A whole number from 0 to 1000; or none. |
| `date_due` | The deadline: the ship day by which the supply must be bought. | A date, or none. A deadline the crew provides is today or later and at most 365 days after today; otherwise the request is refused. |
| `alert_state` | How urgent buying it is (`ALERT-LEVELS`). | Derived (`ALERT-DERIVED`) by the rules in this section. |

A supply with a deadline is a **deadline supply**; a supply without one is a **regular supply**.

### SUPPLY-TRACKED — A supply is tracked exactly when its alert state isn't inactive

A supply is **tracked** when its alert state is green, yellow, red or auto-destruct, and **inactive** when its alert state is inactive. An inactive supply has no deadline. Its stock state and quantity are kept, and the crew can still update them (`SUPPLY-STOCK`, `SUPPLY-EDIT`).

A supply becomes inactive when the crew deactivates it (`SUPPLY-DEACTIVATE`), when the crew removes its deadline (`SUPPLY-RESCHEDULE`), when its deadline passes while it's in stock (`SUPPLY-DEADLINE-PASSED`), or when it's set in stock at auto-destruct (`SUPPLY-STOCK`). It becomes tracked again when it's reactivated (`SUPPLY-REACTIVATE`) or given a deadline (`SUPPLY-RESCHEDULE`).

### SUPPLY-ALERT-STOCK — The alert of a regular supply

| `stock_state` | `alert_state` |
|---|---|
| in stock | green |
| running low | yellow |
| out of stock | red |

A regular supply never reaches auto-destruct.

### SUPPLY-ALERT-DEADLINE — The alert of a deadline supply

A deadline supply that's in stock is green, however close its deadline is. Otherwise (running low or out of stock), its alert follows the number of days from today to its deadline:

| Days until the deadline | `alert_state` |
|---|---|
| more than `SUPPLY_DEADLINE_YELLOW_DAYS` | green |
| more than `SUPPLY_DEADLINE_RED_DAYS`, up to `SUPPLY_DEADLINE_YELLOW_DAYS` | yellow |
| 0 (due today) up to `SUPPLY_DEADLINE_RED_DAYS` | red |
| passed | `SUPPLY-DEADLINE-PASSED` |

Running low counts as not in stock: having some but not enough still means something must be bought by the deadline. The alert follows the calendar: since the daily advance takes effect at the very start of the ship day (`TASK-MISSED`), a deadline supply's alert always reflects today.

Examples, with the default `SUPPLY_DEADLINE_RED_DAYS` of 1 and `SUPPLY_DEADLINE_YELLOW_DAYS` of 7. Candles, out of stock, deadline 2026-01-20:

| Today | Days until the deadline | `alert_state` |
|---|---|---|
| 2026-01-10 | 10 | green |
| 2026-01-12 | 8 | green |
| 2026-01-13 | 7 | yellow |
| 2026-01-18 | 2 | yellow |
| 2026-01-19 | 1 | red |
| 2026-01-20 | 0 | red |

The same candles in stock: green on every one of these days.

A deadline supply keeps its deadline when it's bought. If it runs out again before the deadline, it escalates against the same deadline. *Example:* drinks for a party on 2026-01-20, bought on 2026-01-15 (green). The crew drinks them on 2026-01-18 and sets them out of stock: yellow on 2026-01-18 (2 days left), red on 2026-01-19.

### SUPPLY-DEADLINE-PASSED — When a deadline passes

At the daily advance that starts ship day D, for every tracked deadline supply whose deadline is before D:
- **In stock:** the supply becomes inactive, and its deadline is removed. The event has passed, so the supply is no longer needed. Its stock state and quantity are kept.
- **Running low or out of stock:** the supply goes to auto-destruct and keeps its deadline. The countdown runs during D; unless the supply is resolved on D, the ship is destroyed at the next advance (`SHIP-DESTROY`).

An auto-destruct supply stays at auto-destruct until it's resolved (`SHIP-RESOLVED`). Later advances don't change it. It's resolved when it's:
- deactivated (`SUPPLY-DEACTIVATE`);
- given a new deadline (`SUPPLY-RESCHEDULE`), e.g. because the party was postponed;
- given no deadline (`SUPPLY-RESCHEDULE`): it becomes inactive;
- set in stock (`SUPPLY-STOCK`): it becomes inactive, as if it had been in stock when the deadline passed;
- deleted (`SUPPLY-DELETE`).

So a tracked supply never has a passed deadline unless it's at auto-destruct, and a passed deadline never causes an auto-destruct during a ship day, only at an advance.

*Example* (default settings): candles, deadline 2026-01-20.

| Ship day | Candles not bought | Candles bought on 2026-01-18 |
|---|---|---|
| 2026-01-19 | red | green (bought the day before) |
| 2026-01-20 | red | green |
| 2026-01-21 (advance) | auto-destruct; the countdown runs during 2026-01-21 | inactive, no deadline |
| 2026-01-22 (advance) | the ship is destroyed, unless the candles were resolved on 2026-01-21 (`SHIP-DESTROY`) | — |

*Example:* the candles reached auto-destruct on 2026-01-21. The same day, the party is moved to 2026-01-31, and the crew changes the deadline to 2026-01-31: out of stock, 10 days away, so green. The ship isn't destroyed.

*Example:* the candles reached auto-destruct on 2026-01-21. Someone buys them that day and sets them in stock: inactive, no deadline. The ship isn't destroyed.

*Example:* the candles reached auto-destruct on 2026-01-21, and nobody acts. The ship is destroyed at the advance on 2026-01-22, and the candles stay at auto-destruct. On 2026-01-23 the crew deactivates them: inactive, no deadline, and the auto-destruct is resolved (`SHIP-RESOLVED`).

### SUPPLY-CREATE — Creating a supply

The crew provides `name` and, optionally, `stock_state` (out of stock if not given), `quantity` and `date_due`. The request is refused, and no supply is created, if a provided `date_due` is in the past or more than 365 days after today, or a provided `quantity` is outside its range (`SUPPLY-FIELDS`).

The new supply is tracked. Its alert follows `SUPPLY-ALERT-STOCK` or `SUPPLY-ALERT-DEADLINE`.

### SUPPLY-STOCK — Changing the stock state

The crew sets a new stock state, at any time:
- **Tracked supply, not at auto-destruct:** the alert follows the new stock state (`SUPPLY-ALERT-STOCK`, `SUPPLY-ALERT-DEADLINE`).
- **Auto-destruct supply:** set in stock, it becomes inactive and its deadline is removed; the auto-destruct is resolved (`SHIP-RESOLVED`). Set running low or out of stock, the stock state is saved and it stays at auto-destruct.
- **Inactive supply:** the stock state is saved, and the supply stays inactive.

### SUPPLY-RESCHEDULE — Changing the deadline

The crew sets a new deadline, or none, at any time. A new deadline is checked as in `SUPPLY-FIELDS` (today or later, at most 365 days ahead).

**Tracked supply:**
- With a new deadline, the alert follows it (`SUPPLY-ALERT-DEADLINE`). Changing the deadline has no penalty: the alert follows the new deadline alone, unlike postponing a task (`TASK-POSTPONE`).
- With no deadline, a deadline supply becomes inactive, the same result as deactivating it (`SUPPLY-DEACTIVATE`): its stock state and quantity are kept. For a regular supply, which has no deadline to remove, nothing changes: it stays tracked, and the request succeeds.
- An auto-destruct supply that gets a new deadline, or none, is resolved (`SHIP-RESOLVED`).

**Inactive supply:**
- With a new deadline, it becomes tracked with that deadline: the same result as reactivating it with that deadline (`SUPPLY-REACTIVATE`).
- With no deadline: nothing changes, it stays inactive, and the request succeeds.

*Example:* candles at auto-destruct since 2026-01-21, out of stock. The party is cancelled, and the crew removes the deadline on 2026-01-21: inactive, still out of stock. The auto-destruct is resolved (`SHIP-RESOLVED`).

*Example* (default settings): sunscreen, deactivated for the winter, out of stock. On 2027-05-01 the crew sets a deadline of 2027-05-05, as a reminder to buy it before it gets too sunny: tracked, 4 days away, so yellow.

*Example* (default settings): candles at auto-destruct since 2026-01-21. The crew sets the deadline to 2026-01-21, the same day: resolved, out of stock with 0 days left, so red. Unless they're bought that day, they go to auto-destruct again at the advance on 2026-01-22, and the countdown runs during 2026-01-22.

### SUPPLY-DEACTIVATE — Deactivating stops tracking a supply

Deactivating a tracked supply:
- `alert_state` becomes inactive, and the deadline is removed.
- `stock_state` and `quantity` don't change.
- An auto-destruct supply is resolved (`SHIP-RESOLVED`).

Deactivating is idempotent: deactivating an inactive supply changes nothing, and the request succeeds.

### SUPPLY-REACTIVATE — Reactivating makes an inactive supply tracked again

The request may include a `date_due`, checked as in `SUPPLY-CREATE`. The supply becomes tracked, with that deadline (or none), and its alert follows `SUPPLY-ALERT-STOCK` or `SUPPLY-ALERT-DEADLINE`.

Reactivating is idempotent: reactivating a tracked supply changes nothing, and the request succeeds.

*Example:* sunscreen, deactivated for the winter while in stock. In December 2026 someone uses up the last bottle and sets it out of stock; it stays inactive. Reactivated on 2027-05-01 with a deadline of 2027-06-15 (a holiday): out of stock, more than 7 days away, so green.

### SUPPLY-EDIT — Editing a supply

`name` and `quantity` can be edited at any time, whatever the supply's state; `quantity` can also be removed. Nothing else changes: the stock state and the deadline have their own operations (`SUPPLY-STOCK`, `SUPPLY-RESCHEDULE`), because changing them follows rules of its own.

### SUPPLY-DELETE — Deleting a supply

Deleting removes the supply, whatever its state. Deleting an auto-destruct supply resolves it (`SHIP-RESOLVED`).

## Ships

A ship is a household. Its crew keeps its tasks and supplies in order, and the ship travels a little further each day they do: the better the crew keeps up, the faster it goes.

### SHIP-FIELDS — What a ship records

| Field | Meaning | Values |
|---|---|---|
| `shipname` | The ship's name. | Text. Surrounding whitespace is removed; what remains is 1–50 characters, counted as Unicode code points. Several ships may have the same name. |
| `timezone` | Where the household lives; decides its ship days (`TIME-SHIP-DAY`). | An IANA timezone name, e.g. `Europe/Berlin`. Set at creation, `UTC` if none is given; a name that isn't in the IANA timezone database is refused. Never changed afterwards. |
| `start_date` | The ship day the current journey began. | A date, or none while the ship is destroyed (`SHIP-DESTROY`). |
| `distance` | Light-years travelled on the current journey. | A number ≥ 0, in whole hundredths (`SHIP-SPEED`). |
| `last_advanced_on` | The ship day of the ship's latest daily advance (`SHIP-ADVANCE`). | A date. |

A ship also has two derived values: its condition (`SHIP-CONDITION`) and its speed (`SHIP-SPEED`).

### SHIP-CREATE — A new ship starts its journey on its creation day

A new ship has `start_date` today, `last_advanced_on` today, and `distance` 0. Its creation day counts as already advanced: its first daily advance is the one that starts the next ship day, and that advance credits travel for the whole creation day, however late the ship was created. Who creates a ship and who joins it is defined under Crew (`CREW-CREATE-SHIP`, `CREW-ADD`).

Examples, with the default `DAILY_ROLLOVER_HOUR` of 3 and timezone `Europe/Berlin`. The ship has no items, so its speed is 1:

| Created (ship-local time) | Today | First advance | `distance` after it |
|---|---|---|---|
| 2026-10-20 13:00 | 2026-10-20 | 2026-10-21 03:00 | 1 |
| 2026-10-21 01:00 | 2026-10-20 | 2026-10-21 03:00, two hours later | 1 |

### SHIP-EDIT — Renaming a ship

`shipname` can be changed at any time, whatever the ship's state; nothing else changes. No other field can be edited: `start_date`, `distance` and `last_advanced_on` follow from the journey, and `timezone` is fixed.

### SHIP-CONDITION — The ship's overall state

The ship's condition is the most severe alert state among its tracked items (tasks and supplies), in the order of `ALERT-LEVELS`. With no tracked items, it's green. Inactive items never affect it.

A destroyed ship's condition is auto-destruct, since an item at auto-destruct is what keeps it destroyed (`SHIP-REVIVE`). `start_date` tells the two cases apart: during a countdown the ship still has one; once it's destroyed, it has none.

### SHIP-SPEED — How fast the ship travels

Speed is in light-years per ship day, from the ship's tracked items:

| Ship's items | Speed |
|---|---|
| at least one red or auto-destruct | 0 (the ship is frozen) |
| otherwise, G green and Y yellow, G + Y > 0 | G / (G + Y), rounded to two decimals |
| no tracked items | 1 |

Rounding is the usual one: to the nearest hundredth, and a value exactly halfway rounds up. So every speed, and every distance (`SHIP-ADVANCE`), is a whole number of hundredths.

Inactive items don't count, so pausing or deleting an item changes the speed at once.

*Example:* 3 green tasks, 1 yellow supply, 2 inactive tasks: 3 / 4 = 0.75 light-years per day. One task turns red: 0. The red task is paused: 0.75 again.

*Example:* a new ship with no items, or one whose items are all inactive: 1.

Rounding examples:

| G | Y | G / (G + Y) | Speed |
|---|---|---|---|
| 2 | 1 | 0.6666… | 0.67 |
| 1 | 7 | 0.125 | 0.13 (halfway, rounds up) |
| 1 | 199 | 0.005 | 0.01 (halfway, rounds up) |
| 1 | 200 | 0.004975… | 0 |

*Example:* 2 green and 1 yellow for three days: the distance grows by 0.67 three times, 2.01 in total.

### SHIP-ADVANCE — What the daily advance does to the ship

Every ship day after the creation day (`SHIP-CREATE`) has exactly one daily advance, and it takes effect at the very start of that day: no action on ship day D sees the ship as it was before D's advance. Every ship day after `last_advanced_on`, up to today, gets its advance, one day at a time, in order, before anything else. So advances that were missed (e.g. during an outage) are made up, with the same rules as any other advance. Each advance sets `last_advanced_on` to its ship day.

At the advance that starts D, in this order:
1. **Countdown:** if the ship isn't destroyed and an item is at auto-destruct, the ship is destroyed (`SHIP-DESTROY`).
2. **Travel:** if the ship isn't destroyed, `distance` grows by the ship's speed as it was at the end of D − 1. So D − 1's travel reflects everything the crew did on D − 1.
3. **Items:** tasks and supplies go through their own advance rules (`TASK-MISSED`, `SUPPLY-DEADLINE-PASSED`).

Because the items come last, an item that goes up a level at D's advance first affects travel at D + 1's advance. That's the grace day: a red item reached at D's advance freezes travel for D, and an item that reaches auto-destruct at D's advance destroys the ship only at D + 1's advance.

*Example:* a ship created on 2026-01-10 with one weekly task, last completed 2026-01-05, so green, due 2026-01-12. Nobody completes it. Default `DEFAULT_POSTPONE_DAYS` of 3.

| Advance | Countdown | Travel (for the day before) | Task after | `distance` after |
|---|---|---|---|---|
| 2026-01-11 | — | +1 (green) | green, due 2026-01-12 | 1 |
| 2026-01-12 | — | +1 | green, due 2026-01-12 | 2 |
| 2026-01-13 | — | +1 (it was still green on 01-12) | yellow, due 2026-01-15 | 3 |
| 2026-01-14, 01-15 | — | +0 (yellow alone is 0 / 1) | yellow, due 2026-01-15 | 3 |
| 2026-01-16 | — | +0 | red, due 2026-01-19 | 3 |
| 2026-01-17 to 01-20 | — | +0 (frozen) | at 01-20: auto-destruct, due 2026-01-20 | 3 |
| 2026-01-21 | destroyed | — | auto-destruct | 0, no `start_date` |

### SHIP-DESTROY — Auto-destruct destroys the journey once

An item at auto-destruct has a countdown of one ship day: the day its alert reached auto-destruct. Unless it's resolved (`SHIP-RESOLVED`) that day, the next advance destroys the ship:
- `distance` becomes 0, and `start_date` becomes none.
- This happens once. A destroyed ship stays destroyed, without traveling, until it's revived (`SHIP-REVIVE`). Further advances, and further items reaching auto-destruct, don't destroy it again.
- Its items don't stop: every advance still applies their own rules to them (`SHIP-ADVANCE` step 3), so missed tasks go up a level and deadlines pass as on any other ship. Once revived, the ship travels again only while none of its items is red (`SHIP-SPEED`).

An item at auto-destruct resolved during its countdown day cancels the destruction: the ship keeps its journey.

*Example* (default `DEFAULT_POSTPONE_DAYS` of 3): the ship is destroyed at the 2026-01-21 advance. Another task, green and due 2026-01-22, isn't completed: at the 2026-01-23 advance it goes yellow, due 2026-01-25, although the ship is destroyed.

### SHIP-RESOLVED — Resolving an auto-destruct item

An item at auto-destruct is resolved when its alert leaves auto-destruct, whatever the action, or when it's deleted. The item sections list these actions:
- tasks: completing (`TASK-COMPLETE`), a frequency rescue (`TASK-FREQUENCY` row 3), deactivating (`TASK-DEACTIVATE`), deleting (`TASK-DELETE`);
- supplies: setting it in stock (`SUPPLY-STOCK`), a new deadline or none (`SUPPLY-RESCHEDULE`), deactivating (`SUPPLY-DEACTIVATE`), deleting (`SUPPLY-DELETE`).

Only crew actions resolve an item. The daily advance never moves an item out of auto-destruct, so a ship is never revived without someone acting.

### SHIP-REVIVE — A destroyed ship starts a new journey

When the last item at auto-destruct on a destroyed ship is resolved, the ship is revived at once, by that action: `start_date` becomes today, and `distance` stays 0. The new journey's first travel is at the next advance.

*Example* (continuing `SHIP-ADVANCE`): the ship was destroyed at the 2026-01-21 advance. On 2026-01-22 the crew completes the task: green, due 2026-01-29; the ship is revived with `start_date` 2026-01-22. At the 2026-01-23 advance it travels 1 light-year for 2026-01-22.

*Example:* two tasks reach auto-destruct, one at the 2026-01-19 advance and one at the 2026-01-20 advance. The 2026-01-20 advance destroys the ship (the first task's countdown ran out). On 2026-01-20 the crew completes the first task only: the ship stays destroyed, since the second task is still at auto-destruct. At the 2026-01-21 advance the second countdown runs out, but the ship is already destroyed, so nothing more happens. On 2026-01-21 the crew completes the second task: the ship is revived, `start_date` 2026-01-21.

## Crew

A ship's crew is the people who share the household. Each crew member is a user, an account defined under Users, on that ship.

### CREW-MEMBERS — A membership links one user to one ship

A crew membership says that a user is a member of a ship, and with which role. It's identified by the pair of ship and user: there's no other identifier, and a user is a member of a given ship at most once. Being a member of a ship means exactly that such a membership exists.

| Field | Meaning | Values |
|---|---|---|
| `ship_id` | The ship. | An existing ship. Never changes. |
| `user_id` | The user who is a member. Other members see only the user's display name and `user_id`, never their email or username. | An existing user. Never changes. |
| `role` | The member's title on this ship, e.g. "Captain", "Science Officer" or "Mom". Only a title: it gives no rights. | Text. Surrounding whitespace is removed; what remains is 1–30 characters, counted as Unicode code points. "Crew Member" if none is given. |

- A user can be a member of any number of ships, with a separate membership, and so a separate role, on each.
- A ship has at least 1 member (`CREW-LEAVE`) and at most 20 (`CREW-ADD`).

*Example:* Anna is a member of two ships: "Captain" on her family's ship, and "Cook" on the ship she shares with friends at a holiday home. That's two memberships, (family ship, Anna) and (holiday ship, Anna). Changing her role on one doesn't change the other. Ben is a member of the family ship too, as "Engineer": a third membership, (family ship, Ben).

### CREW-RIGHTS — Every member has the same rights

Every member can do everything on the ship: all task and supply operations, renaming the ship (`SHIP-EDIT`), adding members (`CREW-ADD`), and changing their own role (`CREW-ROLE`). Roles make no difference. A user who isn't a member can't see or change anything on the ship.

### CREW-CREATE-SHIP — Creating a ship

Any user can create a ship (`SHIP-CREATE`). The creator becomes its first member, with the role they give, or "Crew Member".

### CREW-ADD — Adding a member

A member adds another user by that user's email, with a role the adding member chooses (or "Crew Member"). The user is a member at once; they aren't asked. Which user an email belongs to, ignoring upper and lower case, is defined under Users.

The request is refused, and nothing changes, if:
- no user has that email;
- that user is already a member of the ship (whatever role the request gives);
- the ship already has 20 members.

*Example:* Ben signed up as `ben@example.com`. Anna adds `Ben@Example.com` with the role "Science Officer": Ben is a member at once, as "Science Officer". Anna adds him again with the role "Cook": refused; he stays "Science Officer".

### CREW-ROLE — Changing a role

A member can change their own role at any time, and only their own. A role set by whoever added them stays until they change it.

### CREW-NO-REMOVAL — Members can't remove each other

No member can remove another member from the ship. A member stops being one only by leaving (`CREW-LEAVE`) or by deleting their account (`CREW-ACCOUNT-DELETED`).

### CREW-LEAVE — Leaving a ship

Any member can leave a ship at any time. If they're the last member, the ship is deleted, with all its tasks and supplies. This can't be undone.

### CREW-ACCOUNT-DELETED — Deleting a user account

Deleting a user account removes the user from every ship, exactly as if they had left each one (`CREW-LEAVE`): ships where they were the last member are deleted.
