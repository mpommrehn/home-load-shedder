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

**Continued the same evening: the first hardware test (LIFX).** Mark asked
for a quick test on a real bulb. The first sweep of the main LAN found no
LIFX device: Mark keeps most IoT on the router's isolated guest network, and
this PC's VPN also swallowed traffic to that subnet. Mark moved one bulb to
the main LAN, and a throwaway standard-library script drove it over the LIFX
LAN protocol with no cloud:
- Red pulses every 15 s for 2 minutes, all acknowledged and seen by Mark;
  each pulse returns the bulb exactly to its previous state.
- Infrared is controllable separately: on with the visible light off, off,
  and at 10%, each confirmed by Mark with night-vision glasses.
- A bulb that is off shows no pulse, even though it acknowledges one; at 1%
  it shows both colours. So the shedder turns an alert bulb on first.
- Mark chose the alert pattern from what he saw: red at the grid drop, three
  pulses every 30 s at 30% for 5 minutes, then one at 10% every 30 s for the
  rest of the outage; green at the grid's return, the same shape, ending 15
  minutes after the bright phase. Full brightness was "REALLY bright".
- New requirement from the isolation finding: a router rule letting only the
  laptop reach the IoT network, and a fixed address per bulb.

These go into the spec together with Mark's review; `STATUS.md` holds the
details until then.

## 2026-10-01, session G (9b6b5556): Mark's review begins, dehumidifier

**Mark's words.** "Dehumidifier should go down sooner than 30 minutes as
purely discretionary. I think 5 minutes. It should have a software
start-delay time of 3 minutes if that's possible to engineer, in case the
power goes out at 5 and comes back at 6 minutes. I don't want its compressor
to start into pressure and pop the motor safety thermal switch." He has a
second change he could not recall; it is still to come.

**Decisions (Mark).** The dehumidifier gets its own 5-minute tier; TVs and
lamps stay at 30. A 3-minute start delay protects its compressor.

**How the agent engineered the start delay.** A per-load minimum off time,
enforced on every path that can send `on`, with the off time written to disk
first so a restart cannot forget it. The agent pointed out, rather than
claim more than the timer does, that Mark's exact case is already covered by
the 10-stable-minutes restore rule (restore at 16 minutes or later); the
timer is the invariant for every other path.

**Findings that changed the design.**
- The real short-restart risk is the Powerwall's transfer gap, not the
  shedder: a running compressor is cut and re-powered within a second while
  the plug stays on. That happens today. Software cannot see it; the
  protection is the appliance's own restart delay or a plug-in delay-on-make
  protector. Decision waits for the model.
- A dehumidifier that returns on standby after power loss would make every
  restore a silent no-op. The plug's power reading now confirms each restore.

**Pushback.** The 5-minute tier saves only about 0.2 kWh per outage and makes
the unit cycle in about half of all outages. Raised to Mark as he signed
off for the night, not argued; his reason (purely discretionary) stands
unless he revisits it. The cycling is why compressor protection is in.

**Left open.** The dehumidifier's model, its restart delay, and whether it
resumes after power loss. Mark's second change.

## 2026-10-10, session H (0b18f330): plug shopping, dehumidifier facts

**Mark's words.** "Read my projects, find Claude's recommendations for smart
plugs with power reading capability." Then: check prices and availability
for the Zigbee plugs on Amazon, check the Zigbee2MQTT issue tracker for the
ThirdReality Gen2, and "record the findings."

**Findings that changed the design (Mark's observations, 2026-10-10).**
- The dehumidifier resumes by itself after a power loss. The restore check's
  "confirm it resumes" item is closed: a plug `on` restores it.
- It holds its compressor for 5 minutes at start-up when the humidity is
  above the set-point. Mark reads this as covering every return of power, so
  the appliance protects itself across the Powerwall's transfer gap and no
  delay-on-make protector is needed.
- At start-up it runs only its fan, an induction motor Mark puts at about
  75 W. So the plug's relay never closes onto the compressor.

**What the agent added.** The relay still opens under compressor load when
the shedder cuts the unit; at about 500 W on a 15 A relay that was judged
acceptable. The 3-minute `min_off` is now redundant for this unit and stays,
as cover for a later unit without the hold. Two consequences were written
into the spec as open: the restore alert's "about 5 minutes" window is
shorter than the hold, and below the set-point the compressor never starts,
so no compressor draw is not proof of a failed restore.

**Plug research (no purchase decided).** Amazon blocked the agent's browser
with its bot check; the agent did not click through, and Mark read the
listings himself. His readings: the GE 45853GE is unavailable and poorly
reviewed, so it is out; the ThirdReality Gen2 metering 4-pack is $39, the
Gen3 4-pack $44 with fewer reviews. The agent checked Zigbee2MQTT: both
models are supported and both expose the power-on behaviour setting. Its
issue tracker has no open issue on the Gen2; the serious faults (wrong
decimals, a plug switching on then off) were firmware bugs fixed in 2023,
and most later trouble came in by firmware update. Recommendation: Gen2,
and no firmware updates once the plugs work.

**Pushback.** Mark proposed moving the HS110 to the dehumidifier so all
miners share one plug model. The agent pointed out that gbox drives only
old-protocol Kasa plugs, so a Zigbee plug on a miner leaves gbox without its
power cycle and wall-watts reading until it gains a Zigbee path. Not decided.

**Left open.** Which coordinator (guides favour the ZBDongle-P for
Zigbee2MQTT; Amazon price not yet read). The dehumidifier's model. The
restore-alert window. Mark's second review change from session G.
