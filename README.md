# HomeShip API — Specifications

The files in this repository specify how the HomeShip API is expected to behave, in every implementation of it. They are the source of truth: when an implementation disagrees with a spec, the implementation is wrong, unless the spec is deliberately changed.

Specs describe behavior only (inputs, outputs, observable effects), never how that behavior is implemented.

| File | Specifies |
|---|---|
| `domain.md` | The domain: time, alert states, tasks, supplies, ships and their journey, crew, users. |

## Conventions

**Rule IDs.** Every rule is a heading that starts with a stable ID in UPPER-KEBAB-CASE, e.g. `### TIME-SHIP-DAY — What a ship day is`. The first word names the area (`TIME`, `TASK`, `SUPPLY`, …). An ID is a name, not a number: it implies no order and is never reused. When a rule's meaning changes, it gets a new ID. Rules refer to other rules by ID, and tests cite the IDs of the rules they verify.

**Examples are part of the rule.** An implementation that matches the prose but not an example is wrong. If prose and an example disagree, the spec has a bug: report it, don't choose one.

**Settings.** A setting is a value configured per deployment through an environment variable, so it can differ between deployments. Specs refer to a setting by its environment variable name in UPPER_SNAKE_CASE, e.g. `DAILY_ROLLOVER_HOUR`. In a spec, such a name always means a setting, never a code constant. Each spec file lists the settings its rules use, with their defaults and valid values, in a Settings section at its top. Rules hold for every valid value of a setting; examples assume the default and say so.

**Unmentioned means unspecified.** If a spec doesn't define a behavior, don't infer it from the implementation; ask for the spec to be extended.
