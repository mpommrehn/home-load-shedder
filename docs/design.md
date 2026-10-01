# Design of record: home-load-shedder

Version 0.1 (spec), 2026-09-29, session F of the powerwall work (bccf42aa).
Every decision below was made with Mark in that session unless marked
otherwise (2026-10-01, session G: the dehumidifier's 5-minute tier and
compressor protection). Changing one is a decision with Mark, recorded in
`home-load-shedder-EVOLUTION.md`.

## Purpose

The house runs entirely on one Tesla Powerwall 3 (about 10.8 kWh available
to draw down, Mark). Nothing in the house is wired to the grid
directly. The Powerwall is good enough that nobody notices an outage, and the
Tesla app has no priority alert for one. In February 2026, after a snowstorm,
the two heat pumps drew the Powerwall from 100% to 88% in the first
20 minutes of a 36-hour outage before Mark noticed and switched them off.

This project sheds loads in tiers, by time since the grid dropped and by
battery level, tells the household the grid is down, and restores loads when
the grid is back and stable. It does all of this **without the internet**:
the Comcast connection drops within minutes of a neighbourhood outage and
returns 1 to 6 hours after the grid (Mark, from repeated experience).

## Done when

- A real outage, or a test opening of the main breaker, sheds each tier at its
  time (section "Tiers"), the iPad alerts and the living-room bulb pulses red.
- After the grid has been back for 10 minutes without a drop, every shed load
  except the PC is back on, and the bulb pulses green.
- Nothing is harmed on the way: no miner is power-cut when a graceful shutdown
  was possible, no Nest thermostat loses power or configuration, no load is
  switched by anything other than the shedder (or by hand).

## Constraints

- **No internet during an outage**, for its whole length plus hours after.
  No part of the shed or alert path may use a cloud service.
- Everything that must work in an outage runs on Powerwall power and on the
  LAN: the Devuan laptop, the router and switches, the iPad, the plugs.
- `pwlog` stays read-only toward the Powerwall. This project never writes to a
  Tesla device.
- The laptop runs Devuan (sysvinit, no systemd): services start from init
  scripts, as pwlog's Linux service does.
- Credentials never pass through an agent, and are never logged or committed.
- Every non-trivial change ships with tests that are run. Security review once
  at feature-complete (the pwlog rule), unless Mark decides otherwise.

## Tiers

| Trigger | Loads | Mechanism |
|---|---|---|
| Grid off for **2 min** | Both heat pumps (compressor and heat call) and both air-handler blowers | Hardware only (section "Tier 0"). Restored the moment the grid returns. |
| Grid off for **5 min** | Dehumidifier (500 W) | Shedder: its Zigbee plug. Purely discretionary (Mark, 2026-10-01). Compressor protection applies (section "Compressor protection"). |
| Grid off for **15 min** | Miners, then the Windows PC | Shedder: gbox shuts the miners down gracefully and confirms; then the PC agent shuts the PC down (section "Miners and the PC"). |
| Grid off for **30 min** | TVs, lamps | Shedder: Zigbee plugs and LIFX bulbs over the LAN. |
| Outage **starts** below **60%** | The 5-, 15- and 30-minute tiers, at once | Shedder. The heat pumps still go at 2 min. |
| Battery at or below **25%** at any time in an outage | Everything the shedder controls | Shedder, whatever the timers say. |
| Car charging | Wall Connector | Tesla's own "vehicle charging during outage" threshold, set to about 90% by Mark in the Tesla app. Not controlled here. |
| Clothes dryer | | By hand (Mark's decision, 2026-09-28). |

Why these numbers: the median outage here is 5 to 7 minutes (Mark), so the
2-minute ride-through avoids shedding for transfers and blips, and 30 minutes
is well past the median. Two heat pumps at about 5 kW each would take 2.5 kWh
in 15 minutes, a quarter of the usable battery, so they go first in every
season. The house lost about 20 °F in 36 hours in February with no heat, which
Mark accepts; for a longer outage he runs resistance heaters from a 7,500 W
portable generator.

The dehumidifier moved from 30 to 5 minutes on 2026-10-01 (Mark): it is
purely discretionary, so there is no reason to carry it through an outage.
The energy saved is small (500 W for 25 minutes is about 0.2 kWh, 2% of the
usable battery); the cost is that, with a 5-to-7-minute median outage, it is
shed in roughly half of all outages, so it cycles off and on far more often
than it did at 30 minutes. That is why it gets compressor protection.

## Components and where they run

| Where | What |
|---|---|
| Powerwall 3 | Off-grid load-shed relay on the TACO Aux terminals 10-11 |
| Each air handler | Pilot relay breaking Y, W, G (tier 0) |
| Devuan laptop, next to the UPS | pwlog (moved from the PC), Mosquitto, the shedder, Zigbee2MQTT with its USB coordinator (phase 5), NUT server on the Back-UPS USB cable, ntfy server |
| Windows PC (Ryzen2) | gbox plus an MQTT subscriber; a shutdown agent; a NUT client replacing PowerChute |
| iPad (2018), office or kitchen | ntfy app on screen under Guided Access |
| Living room | The always-powered LIFX bulb |

## Tier 0: heat pumps, hardware only

No code of this project is involved, so this tier works when the laptop is
down.

**The trigger** is the Powerwall 3's load-shed relay (Tesla Powerwall 3
installation manual, Backup Switch edition, "Load Shedding Feature"; read
2026-09-29):
- One normally-open dry contact on the TACO Aux terminals **10 (Load Control +)
  and 11 (Load Control -)**, inside the Powerwall's wiring compartment behind
  the front cover. 24-16 AWG, strip 8 mm. Rated 60 V AC/DC, 2 A.
- Closed on grid, open off grid. It also opens when the TACO loses power, when
  the Powerwall's Enable switch is off or the system is Stopped in Tesla One,
  and may open briefly during firmware updates and faults.
- Needs Powerwall software 25.2 or later and Tesla One 12.32 or later. Enabled
  by an installer: Tesla One, Settings > Advanced Settings > TACO Load
  Control Relay > Configuration Type "Off-grid Load Shedding".
- The compartment holds live AC terminals and solar DC inputs that are live in
  daylight. **The installer lands the cable on 10-11** and enables the
  relay; Mark runs and terminates everything outside the Powerwall.

**The circuit** (Mark wires it; he has the training):
- A 24 VAC transformer on a Powerwall-backed circuit powers a **Finder 80.01**
  multifunction timer set to function **BE** (off-delay with signal switch),
  or the BE-only Finder 80.41. A1/A2 supply; B1 signal; 15/16/18 SPDT output.
  The Tesla contact carries supply voltage to B1 (about 1 mA). Delay
  **2 minutes** (top of the 0.1-2 min scale). Set the function before power-up.
- The timer's NO output (15-18) feeds the coils of two **IDEC
  RR3PA-ULAC24V** 3PDT relays on **SR3P-05** sockets, one per air handler.
- At each MrCool Universal air handler (MDUI18024E; outdoor units
  MDUO18024036, set to 36k), the thermostat's **Y, W and G** run through that
  relay's NO contacts. **R and C stay connected**, so the Nest keeps power and
  never reboots. The Nest connects to the air handler, which talks to the
  outdoor unit over its own cable; nothing is touched outdoors.
- Optional: Tesla's documented override switch (60 V, 2 A) across the Tesla
  contact, to bring the heat pumps back by hand.

**Behaviour:** on grid the coils are energised and everything works as today.
The grid drops: the timer holds the coils for 2 minutes, then Y, W and G open
at both units. The grid returns within 2 minutes: nothing ever switched. Any
failure of the transformer, timer or coil sheds the heat pumps (fail-safe).

**Not verified, to test on the bench and then on one unit:**
- The Finder BE reset case: close B1 again at 1:50; the output must stay on,
  and the next opening must time a fresh 2 minutes. Check B1's reference on
  the datasheet wiring diagram (the datasheet was quoted by a research agent;
  this session could not re-read the PDF).
- How the Nest reacts when Y/W/G are opened behind it (a wire-error alert such
  as E103 is possible; it should not lose power or configuration).
- The MrCool units' own restart delay after a call returns.

## The bus

Mosquitto on the laptop, listening on the LAN only. Every client has its own
username and password; per-topic ACLs enforce who publishes what. Passwords
live in files beside each service's data, never in the repositories.

| Topic | Publisher | Retained | Content |
|---|---|---|---|
| `home/power/grid` | pwlog | yes | `{"state": "on"|"off", "since": <ISO time>}` |
| `home/power/battery` | pwlog | yes | `{"soe_pct": .., "load_w": .., "time": ..}` |
| `home/power/heartbeat` | pwlog | yes | time of the last good poll |
| `home/shed/cmd/<target>` | shedder only | no | `shutdown`, `off`, `on`, `pulse` |
| `home/shed/report/<target>` | the executor for that target | yes | `down`, `up`, `failed: <reason>` |
| `home/shed/state` | shedder | yes | episode, tiers shed, next action and time |
| `alerts/grid` | shedder only | no | `{"level": "info"|"high", "text": ..}` |
| `home/ups/<name>` | NUT bridge | yes | status, charge, runtime |

pwlog publishes facts, never commands. The shedder is the only place that
decides; executors only carry out commands and report. Adding a PC or a miner
means adding an executor, with no new rules.

**pwlog change** (in the pwlog repo, its own decision): pwlog is
standard-library-only by rule. MQTT 3.1.1 publish with QoS 0 or 1 and a
retained flag is small enough to write with the standard library; the
alternative is a recorded dependency exception for a client library. Decide in
phase 2.

## Shedder behaviour

**Outage episode.** Starts at the first `grid: off` fact. Ends only after the
grid has been on for 10 minutes without a drop. A drop inside those 10
minutes continues the same episode: the tier timers are measured from the
episode start and never restart, so a flickering grid cannot keep resetting
the 15-minute clock.

**Charge rules.** At episode start, battery below 60% sheds the 5-, 15- and
30-minute tiers at once. At any time in an episode, battery at or below 25%
sheds everything the shedder controls.

**Miners and the PC** (15-minute tier):
1. Command gbox to shut the miners down gracefully.
2. Wait for `report: down` from gbox, up to 5 minutes.
3. No report in time: switch the miners' plugs off (the HS110 feeds the
   SC-BOX; further miners get plugs).
4. Then command the PC agent to shut the PC down. The PC's UPS and NUT client
   remain the last line if the Powerwall itself fails.

**Restore.** When the episode ends: loads come back in groups one minute
apart. The miners come back by a plug power-cycle (off, then on), because
gbox is not running while the PC is off, and the miners start mining on
power-up (to confirm per model). The PC stays off: it has no wake-on-LAN and
needs a manual power-supply switch before its power button, so Mark starts it
by hand.

**Compressor protection** (Mark, 2026-10-01). A compressor switched off runs
with high pressure on its outlet side for a few minutes. Restarted into that
pressure, the motor stalls and its thermal overload switch trips. Mark's
case: the dehumidifier is shed at 5 minutes, the grid returns at 6 minutes.
- Each load has a **minimum off time**, `min_off`, in its configuration:
  0 by default, **3 minutes for the dehumidifier** and for any later load
  with a compressor. The shedder never sends `on` to a load until `min_off`
  has passed since its last `off`, whatever path asks for it: restore, a
  manual command, a test, or a retry after a failed report. A blocked `on`
  is postponed, not dropped, and logged.
- The time of each `off` is written to disk before the command is sent, so a
  restart of the shedder or of the laptop cannot forget it. With no record
  for a load, the shedder treats it as switched off at its own start.
- In Mark's exact case the restore rule already waits longer: the episode
  ends only 10 stable minutes after the grid returns, so the dehumidifier
  comes back at 16 minutes or later. `min_off` is the guard that holds when
  that rule does not apply, and it stays correct if the 10 minutes is ever
  shortened.
- **What software cannot cover: the transfer gap.** When the grid drops,
  and possibly when it returns, the Powerwall's transfer interrupts the
  supply for a moment (the PC shutdowns of 2026-09-21 and 09-24 were such
  gaps). A running compressor loses power and gets it back within a second,
  with the plug still on, so the shedder never sees it. That is a short
  restart into pressure, and it happens today, with or without this
  project. Protection there has to be in the appliance or in hardware:
  1. the dehumidifier's own restart delay (many units hold the compressor
     about 3 minutes after power-up; check this model's manual);
  2. failing that, a plug-in delay-on-make protector (an appliance or
     refrigerator protector with a restart delay of about 3 minutes)
     between the plug and the dehumidifier.
  Decide once the model is known.
- **Restore check.** Many dehumidifiers come back on standby after a power
  loss instead of resuming. If this one does, a plug `on` restores nothing.
  Confirm it resumes (manual, then a test). The plug's power reading confirms
  every restore: fan draw at once, compressor draw within about 5 minutes; if
  not, the shedder raises an alert.

**Missing information.** pwlog's heartbeat older than 60 seconds: the shedder
neither sheds nor restores, and raises an alert. Tier 0 does not depend on
any of this.

**Dry-run mode.** Logs every decision it would take and switches nothing. It
runs in dry-run through several weeks of real blips before going live.

**Log.** Every episode, decision, command and report goes to a local log, with
a summary per episode.

## Alerts

No iPhone can be woken without the internet: background notifications go
through Apple's push servers, and Apple's local-push mechanism is restricted
to special-purpose apps (research, 2026-09-29). The alert path is therefore
local:

| When | Living-room LIFX bulb | iPad (ntfy) |
|---|---|---|
| 2 min (heat pumps shed) | slow red pulse, repeated every few minutes until the grid is back | normal-priority message |
| 15 min (before miners and PC) | continues | high-priority message, loud sound |
| Episode ends | one green pulse | quiet message |
| pwlog silent > 60 s | | high-priority message |

- **LIFX:** LAN protocol, UDP 56700, SetWaveform (message 103) pulse with
  return to the original colour. No cloud. The bulb gets a DHCP reservation.
  Whether every 2017 bulb supports it identically is to be tested.
- **iPad:** a 2018 iPad (LCD, so no burn-in), plugged in, auto-lock Never,
  minimum brightness, locked into the ntfy app with Guided Access, subscribed
  to the laptop's ntfy server. The ntfy iOS app receives without Apple's
  servers only while it is on screen and unlocked, hence Guided Access.
  ntfy requires iOS 15.0 or later (App Store, checked 2026-09-29). **Not
  verified:** that the app holds its LAN connection for weeks; a one-week soak
  test with a daily test alert comes first. The iPhone 7 Plus is the spare.
- Rejected, with reasons: the home alarm's Zigbee siren (would read as an
  intruder alarm); the laptop's speaker (in the basement); Mark's wife's Mac
  (a sleeping Mac runs nothing, and it is her machine); the Apple TV as a
  Home hub (a few watts around the clock for a long shot).
- **Later:** internet alerts to phones once the MoFi 4G modem is cabled into
  the main router's failover WAN port.

## Plugs and bulbs

- **Zigbee** plugs with a USB coordinator (Sonoff ZBDongle-E or -P) on the
  laptop, via Zigbee2MQTT: no WiFi, no cloud, no clock dependency. Candidates
  with energy monitoring: OWON WSP404, ThirdReality Gen2. Check 15 A and UL
  on the label before buying. Z-Wave (Zooz ZEN15/ZEN04) is the alternative.
- **Not TP-Link for new plugs.** Kasa and Tapo set their clocks from the
  internet; after a power blip with the internet down, on/off still works but
  other features stop (TP-Link support, 2024). Some newer models fail local
  login (python-kasa issue, 2026-09).
- **The existing HS110** (SC-BOX) stays: old-protocol Kasa, local and
  unauthenticated, works offline. Its cloud binding is removed and its stored
  schedules checked, so nothing but the shedder switches it (the goldshell
  project traced a 06:30 switch-off to a likely cloud rule).
- Shelly excluded by Mark (reliability reports).

## UPS

The Back-UPS ES 650G1 feeds the PC and the SC-BOX. Its USB cable moves to the
laptop. NUT (Network UPS Tools) serves its state on the LAN; the PC runs a NUT
client that replaces PowerChute's "hibernate after 1 minute on battery". A
NUT-to-MQTT bridge publishes `home/ups/...`. The UPS is not the outage
signal: it sits on the Powerwall and sees only the transfer gap; it goes on
battery only if the Powerwall itself fails or empties. **Not verified:** the
ES 650G1 on NUT's compatibility list.

## Failure modes

| Failure | Result |
|---|---|
| Laptop down | Tier 0 still sheds. Nothing else sheds; loads stay as they are; no alerts. |
| Powerwall TACO unpowered or Stopped | Tier 0 sheds after 2 min (fail-safe). |
| pwlog silent | Shedder holds state and alerts. |
| gbox never confirms | Plugs cut the miners after 5 min. |
| PC hangs during shutdown | No damage; UPS and NUT client are the last line. |
| iPad loses its connection | The bulb still pulses; the daily test alert exposes it. |
| Zigbee coordinator fails | Those plugs are not switched; shedder logs command failures and alerts. |
| Shedder restarts mid-episode | Off times are on disk; `min_off` still holds. No record: a full `min_off` from the restart. |
| Dehumidifier does not resume after a plug `on` | No compressor draw within about 5 minutes: alert. |

## Security

- Broker LAN-only, per-client passwords, ACLs as in "The bus". TLS on the LAN
  is optional; decide at phase 2.
- The shedder can switch loads and shut down a PC: its commands are the
  sensitive part. Only the shedder account may publish on `home/shed/cmd/#`,
  and the PC agent accepts only `shutdown` on its own topic.
- The iPad runs only ntfy, under Guided Access.
- No credential in any repository, log, or transcript.

## Phases

0. **Mark, independent of the code:** set the Tesla app EV outage threshold
   (about 90%); check Powerwall software ≥ 25.2; book the installer (enable
   the relay, land 10-11); buy and bench-test the timer and pilot relays;
   wire tier 0 one unit at a time.
1. Move pwlog to the laptop, with the L2 fix (POSIX secret temp file) that
   waits for the Linux move.
2. Mosquitto on the laptop; pwlog publishes facts.
3. The shedder core, with a simulated clock and tests; run in dry-run.
4. Executors: LIFX, ntfy and the iPad (one-week soak), the HS110, the gbox
   subscriber and PC agent, NUT and its bridge.
5. Zigbee coordinator and plugs (purchase); go live after the dry-run weeks.

Each phase gets its own plan and done-when.

## Deferred

- Internet alerts through the MoFi 4G failover.
- Moving gbox to the laptop (removes the miners-before-PC ordering).
- A second, independent stop for car charging (a contactor on the Wall
  Connector circuit driven by tier 0's timer).
- The pwlog VPP inference (unchanged, in pwlog's design).

## Sources

Research reports of 2026-09-29 (web only, no LAN device contacted), kept in
the session scratchpad and summarised above. Primary documents:
- Tesla Energy Library, Powerwall 3 install manual, Backup Switch edition:
  "Load Shedding Feature" (GUID-570320A6) and TACO wiring pages.
- Tesla Powerwall 3 Device Setup Guide: Frequency-based Load Shed (GUID-37716ACB).
- LIFX LAN protocol: lan.developer.lifx.com.
- ntfy documentation (iOS instant delivery, self-hosting).
- Finder 80 series datasheet (T8001_11_41_61TEN.pdf).
