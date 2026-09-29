# home-load-shedder: construction and evolution log

One entry per session, newest at the bottom. What was decided and why; git
holds the code history, `STATUS.md` the current state.

## 2026-09-29, session F (bccf42aa): from pwlog's night check to a spec

**Where it started.** The session opened as pwlog session F: "pick up the
powerwall project". The overnight check came back clean (two nights, no
grid-services flag, reserve top-ups triggered by charge, not by a schedule).
The VPP inference decision was deferred for lack of any positive example.
Mark chose the load-shedding design session next.

**Research that shrank the problem.** Three web-only research passes, key
claims re-checked against Tesla's own manuals:
- Tesla already stops car charging in an outage below an owner-set threshold,
  signalled to the car by AC frequency, with no internet. The Wall Connector
  itself has no write interface, so the planned "stop the charger" research
  item became "set the app threshold".
- The Powerwall 3 has an off-grid load-shed relay (60 V, 2 A dry contact,
  terminals 10-11), enabled by an installer. Tesla's own example is a
  thermostat-controlled air conditioner.
- A Nest Learning Thermostat 3rd gen cannot be controlled locally; Google's
  API is cloud-only and Matter never reached that model.

**Pushback, both ways.**
- The agent questioned whether the internet really fails in an outage and
  proposed measuring it. Mark corrected it: Comcast drops within minutes and
  returns 1 to 6 hours after the grid. The fact was already on record in
  this project's predecessor and in the goldshell notes; the agent had not
  checked. It is now a cross-project memory.
- That correction exposed a hole in pwlog's notification plan: internet push
  would arrive hours after the grid came back.
- The agent warned that HVAC "delay-on-break" timers are usually
  anti-short-cycle devices, the opposite of what was needed, and specified an
  off-delay timer with a signal input (Finder 80.01, function BE).
- The agent advised against a listener on Mark's wife's Mac (a sleeping Mac
  runs nothing; her machine) and the Apple TV as a Home hub. Mark supplied a
  2018 iPad, which became the alert device.

**Decisions (Mark).**
- Heat pumps and blowers off 2 minutes after the grid drops, in every
  season, by hardware (relay, timer, pilot relays breaking Y, W, G). Reason:
  in February 2026 they drew the battery from 100% to 88% in 20 minutes.
  Losing heat is acceptable; the house lost 20 °F in 36 hours.
- Miners and the PC at 15 minutes, gracefully; lamps, TVs, dehumidifier at
  30 minutes (median outage 5-7 minutes). Below 60% at the start, shed at
  once; at 25%, shed everything.
- MQTT bus with a single shedder, on the Devuan laptop, which also takes the
  UPS's USB cable (NUT replaces PowerChute). gbox stays on the PC for now.
- Auto-restore after 10 stable minutes; the PC stays manual.
- Zigbee plugs for new loads (TP-Link loses its clock without internet;
  Shelly excluded by Mark). LIFX bulbs controlled over their LAN protocol.
- Alerts: the living-room LIFX bulb pulses red, and the iPad runs ntfy in the
  foreground under Guided Access. Internet alerts wait for the MoFi 4G
  failover.
- A new project, not part of pwlog, so pwlog stays read-only.

**Built.** `docs/design.md` (the spec), this log, `AGENTS.md`. No code.

**Left open.** The Finder reset case, the Nest's reaction to opened wires,
LIFX support on the 2017 bulbs, the iPad soak test, and the UPS on NUT's
list are all unverified and scheduled as tests in the phases.
