# PW3 load-shed relay, EV outage charging, off-delay timer (research 2026-09-29)
Confidence tags: [T]=Tesla doc read directly, [S]=search snippet of Tesla page (page itself 403), [D]=datasheet read, [F]=forum/news, [X]=speculation.

## Q1 PW3 load-shed relay
Sources (all energylibrary.tesla.com, PW3 Install Manual with Backup Switch):
- Load Shedding Feature: .../Powerwall/3/InstallManual/BackupSwitch/en-us/GUID-570320A6-25C3-48F1-B96D-7CFB4E1FF573.html
- TACO Low Voltage and Communication Wiring: .../BackupSwitch/en-us/GUID-020A10C8-4FD8-4E36-B589-17441842529F.html (also under BackupGateway/2/)
- PW3 to Backup Switch connection: .../BackupSwitch/en-us/GUID-65D9F117-C421-45C9-9A1F-430FF3243AD5.html
- Device Setup Guide, Configure Low Voltage Relay Control: .../Powerwall/3/DeviceSetupGuide/en-us/GUID-C7B6F3FE-F71B-45CD-B5B4-AF946A2A9730.html
- TACO LED page: .../DeviceSetupGuide/en-us/GUID-2B94252E-E912-4B6B-B5CB-7F0C02EBAA47.html
- Quick guide wiring connections: .../Powerwall/3/QuickGuide/Install/en-us/GUID-84363455-C64B-4F67-8166-4945B775587D.html

Quotes [T]:
- "Beginning with Powerwall software version 25.2 and Tesla One version 12.32, Powerwall 3 allows for load shedding, meaning a low voltage, dry contact control circuit can be wired to control a specific load. For instance, an air conditioning load controlled by a thermostat can be wired to the Powerwall 3 Aux wiring terminals so that the load can be shed when the Powerwall system is off-grid."
- "CAUTION: Only use the load shedding feature to control loads; the load shedding feature is not approved for AC-coupled solar." (Owner: DC-coupled solar into PW3 is fine; still confirm which you have.)
- "on-grid, the low voltage control circuit is closed and the load is powered. When the system is operating off-grid, the low voltage control circuit is open and the load is OFF."
- "The load shed relay is rated up to 60 V (DC or AC) and 2 A."
- Tesla's own method: remove ONE of the two thermostat control wires at the condenser and splice it in series with the Aux cable (i.e. relay in series with one control wire; optional override switch across it, rated >=60 V 2 A, terminals accept 24 AWG).
- "Because the Powerwall 3 load shedding relay is normally open, a connected load will turn off when the TACO is not powered."
- "The low voltage control circuit is by default open on loss of power to the TACO. If the TACO is not back fed 12V from a Backup Gateway 2, it may open when the Powerwall 3 Enable switch is turned OFF, or if the system is 'Stopped' in Tesla One. It may also open briefly during firmware updates and under abnormal fault conditions."
  -> Design implication: the relay opens on non-outage events too (firmware update, Enable off, fault). Your 2-min off-delay absorbs "brief" opens but a Stopped system or Enable-off will shed HVAC. Acceptable, but know it.
- Menu: Settings > Advanced Settings > TACO Load Control Relay > Configuration Type > "Off-grid Load Shedding" > Done.
- Setup guide: only "Off-grid Load Shedding" is listed as a Configuration Type, for both "TACO Load Control Relay" (PW3) and "Gateway Load Control Relay" (Backup Gateway 2 only). No other modes documented. [T]
- Setup guide: "systems using Backup Switch or Gateway 3 can only connect a single low voltage control circuit"; Backup Gateway 2 systems may support two (one on Gateway relay, one on TACO). [T, paraphrase via fetch tool]
  -> One relay circuit is all you get with a Backup Switch. Your plan (one circuit -> timer -> two pilot relays) fits.

Physical [T]:
- TACO = Tesla Asset Controller, a board inside the PW3 wiring compartment. Manual: routing "using the wire management tab to prevent them from blocking the Tesla Asset Controller"; route comms "into the left side of the enclosure". TACO LEDs "are not visible when the Powerwall 3 front cover is installed" -> the TACO is behind the front cover.
- TACO low-voltage terminal block, numbered in the manual: 1 CN- (orange), 2 CN+ (blue), 3 GND, 4 12V (comms to Backup Switch); 5 RS-485+, 6 RS-485- (Tesla remote meter); 7 RJ45 Ethernet jacks; 8 RSD+, 9 RSD- ("The terminal between the two RSD terminals is not used"); 10 Load Control +, 11 Load Control - (the load-shed relay, called "Aux" in the load shedding page); 12 TACO low voltage harness (12 V power to TACO).
  So the relay shares the same TACO field-wiring area with CAN/12V, RS-485 meter, Ethernet, and RSD. Load Control +/- is polarity-labelled but the relay is dry contact rated AC or DC; treat polarity as irrelevant [X] for AC 24 V.
- Wire: 24-16 AWG (0.2-1.5 mm2), strip 5/16 in (8 mm) for all terminals. Connector type: the manual says the comms connector is "insulation displacing ... pierces the insulation of the unstripped wires" with "a 3/32-inch slotted screwdriver opens terminal levers" - that text is for the Backup Switch comms connector; for the TACO block the table gives strip length 8 mm, so it is a lever/push-in type with stripped wire [X on exact style]. Torque: none given for these terminals (only ground 35 in-lb, conduit hub T20 1.5 Nm).
- CAN needs twisted pair; recommended min 18 AWG for GND/12V. Max run figures for CAN/RSD (45 m) are given; none for load control. Cable type: Backup Switch side says LFNC or TC-ER cable in a 1/2" NPT fitting/raceway. The load-control cable entry into the PW3 is not spelled out in the fetched pages ("route ... into the left side of the enclosure, use wire management tabs") [T partial].
- Backup Switch itself: no relay documented for load shed; the Gateway Load Control Relay exists only on Backup Gateway 2 (per setup guide). [T]

Owner access / safety [T + S]:
- PW3 owner manual: no user-serviceable parts; do not remove cover; installation/service "should only be performed by Tesla Certified Installers" (snippets of owner manual + energylibrary Owner Manual/Warranty pages). Warranty: registered/commissioned and internet connected; Tesla disclaims liability for unqualified repairs. Not verified whether opening the cover voids the warranty specifically; would be settled by reading the Warranty page https://energylibrary.tesla.com/docs/Public/EnergyStorage/Powerwall/3/OwnerManual/en-us/GUID-B100B118-F801-447F-A5EC-12159A1464B5.html.
- Safety, plainly: the same compartment holds PW3 AC terminals (up to ~64 A), PV/MPPT DC inputs (solar DC can be live in daylight even with the Enable switch off and breakers open) and the battery. Manual says PV wiring requires the Enable switch OFF first. The low-voltage TACO block is adjacent. Even a trained low-voltage wirer working in a live DC/AC enclosure is at real arc/shock risk. The relay must ALSO be enabled by an installer in Tesla One (installer login), so an installer visit is required anyway.
- Recommendation [opinion]: have the Tesla installer (or certified electrician) land the two conductors on terminals 10/11 and enable the setting; you supply and run the low-voltage cable to a junction point outside. Terminate at your timer end yourself.

## Q2 EV charging during an outage
Tesla support page tesla.com/support/energy/powerwall/mobile-app/vehicle-charging-during-power-outage returned 403 on every locale and archive.org is blocked; content below from search snippets and secondary sources.
- Setting [S]: Tesla app > Powerwall/home screen > Settings > (EV charging / "Vehicle Charging During Power Outage") with a slider for "charging threshold". "When a power outage occurs, Powerwall will charge your Tesla vehicle based on your charging threshold, which determines how much power you want Powerwall to share with your vehicle." "Once your Powerwall's stored energy level is less than your threshold, Powerwall stops charging your vehicle and continues to provide backup power to your home." Resume: lower the threshold, wait for grid, or wait for solar to lift the battery above the threshold. Also "Powerwall continuously responds to the changing power needs of your home and will slow or stop your vehicle's charging to keep your home loads powered."
- Behavior [F, Teslarati 2020]: grid up = normal; outage with surplus = car charges, slower if home load high; below threshold = charging stops. So above threshold = allowed and throttled by load; below = stopped.
- Range/default of the app slider: NOT found. [unverified] Installer-side range is 10-95%.
- Installer side [T]: Device Setup Guide GUID-37716ACB-217E-4F36-B404-803F9BB39F74: "Frequency-based Load Shed is the same setting, made available in Tesla One. The installer can set the charge level between 10% and 95%; once Powerwall reaches that percentage, it will prioritize vehicle charging." (Tesla's wording is loose; the app page says stops below threshold.) So it is one setting, two front ends.
- Requirements [F, Teslarati at launch]: Model 3/Y then, vehicle software 2019.40.1+, Tesla app 3.10.2+, Powerwall software 1.46+. Later snippets: "Tesla vehicles using a Tesla home charging solution, including Wall Connector and Mobile Connector." Same-account requirement: not stated anywhere I found [unverified]; mechanism suggests it is not needed, since signal is on the AC line.
- Signal [S]: "Powerwall will lower the off-grid frequency when it's below the configured state of charge, which will instruct the Tesla onboard charger to stop charging the battery pack." Frequency shifting, no internet needed for that link [X inference: it is local AC-line signalling; no source says internet is required, none says it isn't].
- Non-Tesla EVs [X/inference]: onboard chargers of other makes do not implement the frequency response, so they ignore it. No Tesla statement found. Your Wall Connector Gen 3 + Tesla vehicle is the supported case.
- Failure reports [F]: TMC "Model 3 charging is draining the powerwall" (a car pulled a Powerwall from full to <20% in ~4 h during an outage; older thread, pre-feature); a report that a firmware update triggered a "fail-safe charging mode" that drained 3 Powerwalls until the car recovered. Threads: teslamotorsclub.com/tmc/threads/how-to-prevent-charging-cars-from-battery.279060/, .../vehicle-charging-during-outage.349794/, .../model-3-charging-is-draining-the-powerwall.226870/ (all 403 to my fetcher; only snippets read). Treat the frequency signal as best-effort, not a hard guarantee.
- Related: Tesla Best Practices During Power Outages page https://www.tesla.com/support/energy/powerwall/own/best-practices-during-power-outages.
- Independent belt-and-braces (X): a contactor on the Wall Connector circuit driven by the same load-shed timer, or setting the car's own charge limit/schedule, would not depend on the frequency signal.

## Q3 Off-delay timer with separate trigger
Terminology confirmed: what you want is "off-delay with control (signal) switch" = Finder "BE", Omron "D: Signal OFF-delay", Macromatic "Off Delay" (control switch). Not "true power off-delay" (Omron H3DT-H, Finder 80.61T "BI") which have no trigger input and need the timer's own supply to be cut. ICM anti-short-cycle: ruled out, not researched further.

Trigger caveat common to all three: the trigger is a VOLTAGE input (a contact that connects supply to the trigger pin), not a dry contact loop. Your PW3 relay is a dry contact rated 60 V / 2 A, so wire it as the "switch" carrying 24 VAC from the timer's supply to its trigger pin. Draw is ~1 mA (Finder B1 <1 mA) so well inside 2 A, and 24 VAC < 60 V.

A) Finder 80.01.0.240.0000 (12-240 V AC/DC, multi-function; select BE; SPDT 16 A) or mono-function 80.41.0.240.0000 (BE only; 24-240 V; check exact code on Finder) [D]
- Datasheet: https://cdn.findernet.com/app/uploads/T8001_11_41_61TEN.pdf (I read it via pdftotext).
- Quote: "(BE) Off-delay with control signal. Power is permanently applied to the timer. The output contacts transfer immediately on closure of the Signal Switch (S). Opening the Signal Switch initiates the preset delay, after which time the output contacts reset." Function must be set before energising.
- Range dial ranges: 0.1-2 s, 1-20 s, 0.1-2 min, 1-20 min, 0.1-2 h, 1-24 h -> use 0.1-2 min or 1-20 min scale (2 min sits at top of 0.1-2 min; use 1-20 min scale for margin).
- Terminals: A1/A2 supply, B1 control signal (contact between A1 and B1, or signal S), 15 common, 16 NC, 18 NO. Operating range 10.8-265 V AC/DC (80.01), 16.8-265 V (80.41). Supply <1.8 VA. Minimum control impulse 50 ms. Contact 16 A/250 V, min load 500 mW (10 V/5 mA): 24 VAC coils fine.
- Trigger reset: closing S during timing re-transfers immediately [inferred from function description; verify in bench test].
- 80.01 24 VAC lower limit is fine (10.8 V).
- Output is SPDT only: one contact feeding two 24 V coils in parallel is fine (coils are ~1-3 VA each).

B) Omron H3DS-ML (screw terminal) / H3DS-MLC (screw-less), mode D "Signal OFF-delay" [D]
- Datasheet: https://files.omron.eu/downloads/latest/datasheet/en/h3ds_solid-state_timer_datasheet_en.pdf?v=1 (read via pdftotext). US docs on omron247/automation.omron.com.
- 24-230 VAC / 24-48 VDC, SPDT 5 A/250 VAC, voltage input on B1 ("voltage can be applied by turning on the contact between B1 and A1"). Terminals: A1, A2 supply; B1 signal; 15 common, 16 NC, 18 NO. Time ranges include 0.1-1.2 min, 1-12 min (so 2 min in the 1-12 min scale) (this table was for H3DS-X, check H3DS-M table when buying) [D partial]. Mode selected with a DIP/rotary switch.

C) Macromatic TR-6162U (Off Delay, 11-pin octal, DPDT 10 A) [D, product page]
- https://macromatic.com/product/tr-6162u/ ; brochure https://www.macromatic.com/wp-content/uploads/2025/01/TR-6-brochure.pdf
- 24-240 V AC / 12-125 V DC; 16 ranges 50 ms-100 h; "Off-delay operation. When triggered, the output energizes immediately. Upon trigger removal, a time delay begins before de-energizing." Needs an 11-pin octal socket (e.g. Macromatic 70169-D or Idec SR3P-05 style; verify socket).
- Terminals (Macromatic standard, from KB): control (trigger) switch between pins 5 and 6 for Off Delay/Single Shot/Watchdog; supply pins 2 and 10 on the 11-pin [supply pins not confirmed in what I read; confirm on datasheet before wiring]. Output DPDT.
- Also TR-5 series (TR-51xxx, 8-pin SPDT/11-pin DPDT) with fixed ranges; e.g. TR-51522-22 is 24 VAC/DC control voltage, 3-300 s, DPDT (shown in Macromatic listings, single-shot function, so not the off-delay part; check the Off Delay variant number).

Recommendation: Finder 80.41 or 80.01 (cheapest, DIN, 17.5 mm, screw terminals, clear datasheet). Macromatic if you want DPDT and an octal socket. Bench-test the reset behaviour (trigger closes at 1:50, output must stay energised and timer restart).

Pilot relays (coil 24 VAC, 3 poles)
- IDEC RR3PA-ULAC24V (3PDT, 10 A, 11-pin octal, LED indicator) + SR3P-05 socket (11-pin, M3.5 screw terminals, DIN or panel). https://us.idec.com/idec-us/en/USD/Relays-&-Timers/Relays/RR-Power-Relay/p/RR3PA-ULAC24V [D via listing]. Use NC or NO poles: for "break Y, W, G when timer output drops" wire Y, W, G through NO contacts of a coil that is energised while on-grid (or NC contacts if you want the relay energised only during shed; normal choice is NO on an energised relay so a failed timer/coil also sheds the HVAC, which is fail-safe).
- Alternatives: Omron LY3 / MY3, Functional Devices RIB series (RIBU1C is SPDT; check 3-pole availability). Not verified.
- Contacts switch thermostat calls (24 VAC, tens of mA to the air handler board/contactor): far below 10 A; dry-circuit minimum loads may want gold/silver-nickel contacts; RR3PA has AgSnO, fine at 24 V/50 mA in practice [X]. A cleaner option: one 3PDT relay per air handler, Y/W/G in series. Also consider breaking R (the 24 V feed) instead of Y/W/G: one pole and it kills everything. [X, but common practice]. MrCool universal air handler may run heat-strip via W/E and needs G for blower; with Y and G opened the blower stops.

## Design notes / assumptions I am least sure of
1. Whether the timer trigger must actually carry AC (yes for Finder/Omron/Macromatic), which passes 24 VAC through the PW3 relay: fine electrically, but keep the cable inside a low-voltage Class 2 route [X].
2. Fail-safe direction: relay is NO and opens when TACO is unpowered/Enable off/Stopped/firmware update; with the timer arrangement HVAC drops 2 min after any of those, including a PW3 firmware update longer than 2 min. Consider the override switch Tesla documents (60 V, 2 A) in parallel with the timer trigger path.
3. Tesla's own recommended wiring is relay in series with one thermostat wire; yours is different (relay drives a timer) but within the 60 V/2 A rating.
