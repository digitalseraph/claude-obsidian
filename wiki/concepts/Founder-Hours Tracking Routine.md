---
type: concept
title: "Founder-Hours Tracking Routine"
address: c-000006
complexity: basic
domain: project-operations
aliases:
  - "Weekly Founder-Hours Routine"
  - "Founder-Hours Scheduled Agent"
created: 2026-04-28
updated: 2026-04-28
tags:
  - concept
  - automation
  - scheduled-agent
  - project-operations
  - bitstub
status: developing
related:
  - "[[Persistent Wiki Artifact]]"
  - "[[Compounding Knowledge]]"
  - "[[index]]"
  - "[[log]]"
sources:
---

# Founder-Hours Tracking Routine

A scheduled remote Claude Code agent that adds a draft row to a project's `founder-hours.md` log every week. The pattern automates one of the highest-failure-mode operational tasks in solo-OSS work: weekly hour logging that humans reliably forget to do.

The hours field is **left blank** for the human to fill in — the agent has no way to know wall-clock time spent. What the agent does provide: a synthesized one-line Notes summary of the work that happened (from `git log` + GitHub issues), the correctly computed week-ending date, and a side-effect-of-update to the trailing-trend and tripwire-armed status fields.

## Why this exists

Solo OSS founders working on long-timeline projects (multi-year, no funding pressure, day-job sustained) face a specific failure mode: the project doesn't crash, it slow-bleeds. The principal works 60-hour weeks for two months, gets burned out, takes a month off the project, the project loses momentum, the principal blames themselves, the cycle repeats.

The countermeasure is a hard founder-hour cap with an automatic tripwire. From the bitstub plan §7.3.1: 50 hours/week cap; if exceeded for 3 consecutive weeks, automatic V1 scope-cut review per §6.6. One paid sabbatical per quarter, skip-forbidden.

The cap's enforcement depends on the log existing. The log's existence depends on the principal updating it weekly. **Manual weekly logging is itself the failure mode** — the principal who forgot to take a sabbatical is the same principal who will forget to log hours.

A scheduled remote agent breaks the cycle by ensuring the log gets a row every Monday whether or not the principal updates it. Empty/zero-hour rows are themselves signal — "I worked nothing this week" is data. Skipped weeks are not data; they are missing data.

## Mechanism

The agent runs on a cron schedule (typically Monday morning UTC, after the week ends Sunday). Its workflow:

1. **Compute the target week-ending date** as the most recent Sunday in UTC.
2. **Gather work signals** from the project repo: `git log --since='7 days ago' --no-merges` and best-effort `gh issue list --search "updated:>=…"` for issue activity.
3. **Synthesize a Notes summary** ≤120 chars capturing substantive work. If zero commits and zero issue activity: literal string "No bitstub work this week." No padding, no subjective commentary.
4. **Update the tracker file** (`docs/founder-hours.md` or equivalent) with one of three behaviors:
    - If a row for the target week already exists with `Hours: _TBD_` — replace just the Notes column.
    - If a row for the target week exists with `Hours` filled in by the human — abort, the human has already logged.
    - Otherwise — insert a new row at the top of the table data.
5. **Detect sabbatical mentions** in commit messages ("sabbatical", "off this week", "paid week off"); set the Sabbatical column conservatively, leaving "No" when uncertain.
6. **Update the Status block** with the 3-week trailing trend and tripwire-armed flag. If 3 consecutive non-sabbatical filled rows all show >50 hours, add a `> [!warning] TRIPWIRE TRIGGERED` callout pointing to the project's scope-cut procedure.
7. **Commit and push** as the project principal — never the agent's identity. Stage only the tracker file. Idempotent: running the agent multiple times in the same week does not duplicate rows.

The tracker file is the source of truth; the routine is its automated keeper. The principal can edit the file freely between runs — the agent only modifies its own draft row, never touches rows the human has filled in.

## Configuration template

Concrete values for the bitstub instance below. The pattern generalises to any solo-OSS project with the same risk profile.

| Setting | Value |
|---|---|
| Cron expression | `0 13 * * 1` — Monday 13:00 UTC year-round (absorbs DST drift: 9am EDT in summer, 8am EST in winter) |
| Model | `claude-sonnet-4-6` (Sonnet is sufficient; Opus is overkill for a structured update task) |
| Repo | The project's GitHub URL — must be accessible from the remote-agent environment |
| Tools | `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep` — no MCP connectors |
| Commit identity | The project principal's name and email, preserved from the project's local git config |
| Tracker file path | `docs/founder-hours.md` (project convention) |

The full prompt embedded in the routine is self-contained — the agent starts each fire with zero context and must read the tracker file fresh. This is by design: the routine is auditable in isolation, and the principal can swap in a different agent prompt by updating the routine without touching the tracker file format.

## bitstub instance (current)

The first instance of this pattern, created 2026-04-28 against `digitalseraph/bitstub`:

- **Routine ID:** `trig_013b5w322UwGZVhEn3SzRJEA`
- **Manage at:** https://claude.ai/code/routines/trig_013b5w322UwGZVhEn3SzRJEA
- **First fire:** Monday 2026-05-04 at 13:08 UTC
- **Tracker file:** [`docs/founder-hours.md`](https://github.com/digitalseraph/bitstub/blob/main/docs/founder-hours.md) in the bitstub repo
- **Sabbatical schedule the routine watches:** Q1 by ~2026-07-31, Q2 by ~2026-10-31, Q3 by ~2027-02-28, Q4 by ~2027-04-30
- **Tripwire procedure:** if the routine sets tripwire-armed = Yes, the principal initiates a scope-cut review within 7 days per [`docs/plan.md`](https://github.com/digitalseraph/bitstub/blob/main/docs/plan.md) §6.6 and takes the next sabbatical week within 14 days, replacing the regular cadence for that quarter.

The bitstub repo is currently private. The first Monday run on 2026-05-04 will reveal whether the remote-agent environment has the credentials to pull a private digitalseraph repo. If it fails with auth errors, the alternative is running an equivalent script locally via cron or launchd against a local clone.

## When this pattern fits

The pattern is overkill for projects that already have:

- An employer with HR systems tracking hours.
- A team with multiple contributors (the social signal of others' presence carries the same forcing function).
- Funding pressure that creates external accountability (investor reports, grant reports).

It is well-suited for:

- Solo-OSS projects with no funding pressure (the case bitstub fits).
- Multi-year timelines where burnout is a year-2 problem the year-1 founder cannot see.
- Projects with explicit scope-cut tripwires that depend on hour data being recorded reliably.

The pattern is a circuit breaker, not a productivity tool. The signal it produces is most useful at the *boundary* — too few hours (project drift), too many hours (burnout risk). The middle band where most weeks live is the least informative output and that is the point. The routine is silent when things are normal and loud when they are not.

## Composability

The routine composes naturally with other automated artifacts in the same project:

- **Build-in-public posts** — monthly summary feeds the principal's public Mastodon thread; transparency without effort.
- **Tripwire callout** — when the routine sets tripwire-armed, downstream automations (or the principal's own Monday review ritual) can act on the same signal.
- **Co-maintainer onboarding** — when the project gains a co-maintainer, the routine extends naturally to track per-maintainer hours by adding columns to the table; the agent prompt updates without changing the routine's identity.

The pattern is an instance of the broader idea that [[Persistent Wiki Artifact]]s plus scheduled agents form a low-friction substitute for project-management software. The artifact is a markdown file in the repo. The agent is a cron-triggered prompt. The principal owns both.

## Limitations and watch items

- **Hours are unverifiable by the agent.** The principal can log any number, including a wrong one. The pattern relies on the principal's honesty with themselves; it does not enforce.
- **Private-repo auth is the most likely failure mode.** First fire reveals whether the routine can clone the project repo.
- **Cron drift across DST.** The routine fires at the same UTC time year-round, which means the local fire time shifts by an hour twice a year. For a 9am-ish start this is acceptable; for a strict office-hours window the cron would need replacement at DST transitions.
- **Agent prompt drift.** The full agent prompt is embedded in the routine's config and cannot be version-controlled in the same repository as the tracker file. A change to the prompt requires a routine `update` call. For load-bearing prompts, version-controlling the source of truth elsewhere (e.g., as a file in the repo, with the routine prompt being a one-line directive to "follow the procedure in `<file>`") is a future improvement.
- **No notification when the routine fails silently.** The remote-agent run history at the manage URL above is the only detection signal. A future improvement is wiring a notification (email, push) on routine failure.

## See also

- [[Persistent Wiki Artifact]] — the broader pattern of "durable Markdown page as the LLM's memory object" of which the founder-hours tracker is one instance.
- [[Compounding Knowledge]] — why repeated low-effort artifact-updates compound into a durable record over time.
- [[log]] — this wiki vault's own scheduled-update log, an analogue at the meta layer.
