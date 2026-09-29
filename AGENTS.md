# AGENTS.md: home-load-shedder

Sheds household loads in tiers during a grid outage, by time since the grid
dropped and by Powerwall charge, alerts the household on the LAN, and restores
loads when the grid is stable. Works with no internet. Private project.
Claude Code reads this file through `CLAUDE.md`, whose entire contents are the
import line for this file. Do not replace that import with a link.

## Category

- type: personal
- target: Devuan laptop (sysvinit), plus a small agent on the Windows PC
- status: design (spec written 2026-09-29, no code yet)

## Ground truth and the two logs

Read these at session start, in this order:

1. `STATUS.md`: checkpoint. Gitignored; exists only on the machine where the
   work happens. If it is missing, say so and work from the two files below.
2. `home-load-shedder-EVOLUTION.md`: the construction and evolution log, one
   entry per session. Never a credential, an address (IP, MAC, email), or
   another person's data. Rule and entry shape: root `AGENTS.md`, "The project
   construction and evolution log".
3. `docs/design.md`: the design of record. Changing a decision in it is a
   decision with Mark, recorded in the evolution log.

Related: `../powerwall-grid-logger` (pwlog, the source of grid and battery
facts; stays read-only toward the Powerwall) and the goldshell project (gbox,
which shuts the miners down).

## Rules that bound every change

- **No internet in the outage path.** Nothing that sheds, alerts or restores
  may depend on a cloud service.
- **Never write to a Tesla device.**
- **Only the shedder decides.** pwlog publishes facts; executors carry out
  commands and report.
- **Credentials never pass through an agent** and are never logged, printed
  or committed. Mark creates broker passwords himself.
- Nothing touches a live load, the live pwlog, gbox or the PC's power without
  Mark's word. Tests use a fake broker and a simulated clock.
- Every non-trivial change ships with tests, and the tests are run.
- Security review once at feature-complete, not after every change.
- Push to GitHub only at Mark's word.
- Tier 0 (heat pumps) is hardware that Mark wires. Agents document it; they
  do not give electrical sign-off.
