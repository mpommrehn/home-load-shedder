# Tesla outage load-shed research (web only, 2026-09-29)

Confidence labels: DOC = Tesla documentation; FORUM = user/community report; SPEC = speculation/inference.
Note: tesla.com support pages and TMC returned HTTP 403 to fetch; their content below comes from search-result summaries of those pages (still Tesla text, but not read in full). energylibrary.tesla.com pages were fetched directly.

## A. Does Tesla already handle it?

A1. YES, for vehicles, by design. (DOC)
- Tesla support "Vehicle Charging During Power Outage": Powerwall coordinates with Tesla vehicles during an outage, slows or stops charging to keep home loads powered. User sets a "charging threshold" (Tesla app > Settings slider). When Powerwall stored energy is below the threshold, Powerwall stops charging the vehicle; excess solar can still go to the car. Vehicle shows a message on its screen when slowed/stopped. Works with Wall Connector and Mobile Connector; connector must be on a Powerwall-backed circuit. No internet needed; the vehicle need not be on the owner's Tesla account. https://www.tesla.com/support/energy/powerwall/mobile-app/vehicle-charging-during-power-outage (403 to fetch; via search summary), also .../en_th/... version.
- Mechanism is frequency shift (DOC + FORUM). Tesla One Device Setup Guide "Frequency Overrides": section "Frequency-Based Load Shed (EV Charging Threshold)", installer-settable 10%-95% SOC; upward frequency shift up to 5.2 Hz default is for curtailing solar. https://energylibrary.tesla.com/docs/Public/EnergyStorage/Powerwall/3/DeviceSetupGuide/en-us/GUID-37716ACB-217E-4F36-B404-803F9BB39F74.html . Forum description: PW lowers frequency below threshold SOC so the car's onboard charger stops; raises it when full (TMC threads, unverified detail). So the signal is in the AC waveform, and the car obeys it, not the Wall Connector.
- Original 2020 feature announcement: needed PW firmware 1.46, app 3.10.2, Model 3/Y. Electrek https://electrek.co/2020/04/16/tesla-feature-charge-your-car-with-powerwall-during-power-outages/ (FORUM/press; dated, requirements likely relaxed).
- Caveats (FORUM): thecooldown.com article: owner found car still charging at 28% Powerwall during a storm; advice was set the off-grid vehicle threshold / solar-only, or manually stop. https://www.thecooldown.com/green-home/tesla-powerwall-charging-issues-home-backup/ . TMC threads "How to prevent charging cars from battery" and "Vehicle Charging during outage" exist (403, not read).
- NOT documented: any 15-minute delay; the threshold is a SOC gate, not a time gate. Default threshold value not found. Non-Tesla EVs or a car that ignores frequency would not obey. Behavior with Wall Connector Gen 3 specifically is not separately documented; docs say "Wall Connector and Mobile Connector". Whether the threshold applies with PW3 the same as PW2 is implied (PW3 setup guide has the setting) but the app label/default is unverified.
- The user's scenario (car charging, owners away) is exactly what the threshold is for. Owner should check the app setting; if set to e.g. 20-30%, the car is already cut off below that.

A2. Circuit shedding products. (DOC)
- Powerwall 3 has a native load-shed relay (see C). Backup Switch / Backup Gateway 2 also. Tesla "Smart Panel" not researched in depth (SPEC: not a Tesla product I could confirm with sources; skipped for budget). Wall Connector Gen 3 has no built-in outage shedding; its own doc only says it resumes 1-3 min after power returns. https://energylibrary.tesla.com/docs/Public/Charging/WallConnector/Gen3/Install/UniversalWC/en-us/GUID-80C5FCFB-E627-43E8-AFAE-B26FADCCC550.html

A3. Storm Watch. (DOC) Charges Powerwall to max when a severe-weather watch/warning is issued, returns to normal after. No outage-time shedding. Tesla's own advice: reduce large loads incl. vehicle charging during an outage. https://www.tesla.com/support/energy/powerwall/mobile-app/storm-watch ; https://www.tesla.com/support/energy/powerwall/own/best-practices-during-power-outages

## B. Wall Connector Gen 3 remote stop

B1. Local HTTP API is read-only. (FORUM, several independent sources, consistent) Endpoints /api/1/vitals, lifetime, wifi_status, version. No write endpoint found. Sources: ioBroker adapter docs ("write isn't supported by the API") https://github.com/nobl/ioBroker.tesla-wallconnector3/blob/HEAD/docs/en/README.md ; TMC https://teslamotorsclub.com/tmc/threads/gen3-wall-connector-api.228034/ ; evcc issue about polling /vitals timing out after prolonged polling https://github.com/evcc-io/evcc/issues/28525 (do not hammer it).

B2. Power management. (DOC + FORUM)
- Tesla: Static, Dynamic (needs energy meter, installer sets ceiling), Group (up to 6 Gen 3). Configured in Tesla One/at commissioning; no internet required. https://www.tesla.com/support/charging/wall-connector/power-management ; https://energylibrary.tesla.com/docs/Public/Charging/WallConnector/TeslaOne/en-us/GUID-8CD32F0F-C5BA-43D9-951A-EB3E17043427.html
- Community trick (FORUM, working GitHub projects): emulate the Neurio/Generac meter over RS485/Modbus into the TWC3; the WC then dynamically limits current, and reporting 0 A available halts charging; fail-safe defaults to 0 A if the controller loses data. https://github.com/PVi1/esphome-twc-control ; https://github.com/Klangen82/tesla-wall-connector-control ; blog https://www.friism.com/diy-tesla-wall-connector-with-load-management/ . Caveats: the README notes TWC3 firmware 26.26.1 only reacts when reported load exceeds its configured limit by ~1.1-1.2 A, i.e. behavior is firmware-dependent; this is a wiring/firmware hack that touches the WC's meter port (warranty/safety consideration). Fully local, no internet.

B3. Via the vehicle.
- BLE: `tesla-control -ble` in https://github.com/teslamotors/vehicle-command sends commands directly to the car over Bluetooth; no Fleet API/internet needed for BLE. Key pairing: add the controller's public key to the car (approve on the car screen with a key card/existing key present); one-time. Range: BLE is roughly 30 ft (FORUM/general; ESP32 proxies extend it). Works only when the car is parked in range at home (the relevant case). Charging start/stop and amps are supported (ESPHome Tesla BLE library, Teslemetry BLE doc). https://teslemetry.com/docs/topics/bluetooth ; https://github.com/BogdanDIA/tesla-ble ; https://shankarkumarasamy.blog/2024/01/28/tesla-developer-api-guide-ble-key-pair-auth-and-vehicle-commands-part-3/
- Caveat (SPEC): a sleeping car may need a BLE wake; a parked-and-charging car is awake, so likely fine. Not verified. The vehicle-command README's BLE range/offline statements were not explicit; "offline" is inferred from BLE being local.
- Fleet API charge_stop: needs internet, developer registration, signed commands via the vehicle-command proxy (README, DOC). Useless if ISP is down.
- Home Assistant: Teslemetry beta BLE integration (needs HA 2026.7+ per its doc), ESPHome Tesla BLE. TeslaMate is read-only logging (SPEC; not verified).

B4. Dumb alternative. A contactor/relay in the 240 V feed to the Wall Connector is straightforward electrically (needs a properly rated, licensed install; SPEC). WC behavior on power restore: resumes automatically in 1-3 min, button press on handle resumes immediately (DOC, link in A2). That means a relay-based shedder must stay open until restored, and it will re-start charging by itself after close.

## C. Powerwall 3 exposes load control (DOC, best find)
- PW3 has a low-voltage dry-contact load-shed relay on the Aux terminals (PW3 "TACO"); Backup Gateway 2 has one on its Aux terminals too (two total possible with BG2). Requires PW software 25.2+ and Tesla One 12.32+. Rating 60 V AC/DC, 2 A. Config in Tesla One: Settings > Advanced Settings > TACO Load Control Relay (or Gateway Load Control Relay) > "Off-grid Load Shedding".
- Behavior: closed on-grid, OPEN when off-grid (grid status only, no SOC or delay setting documented). Also open by default on loss of power to the TACO (e.g. during firmware updates/faults), so the load sheds then too. Not approved for AC-coupled solar (fine for the owner's grid-tied DC-coupled PW3 presumably; verify).
- Sources: https://energylibrary.tesla.com/docs/Public/EnergyStorage/Powerwall/3/InstallManual/BackupSwitch/en-us/GUID-570320A6-25C3-48F1-B96D-7CFB4E1FF573.html ; https://energylibrary.tesla.com/docs/Public/EnergyStorage/Powerwall/3/DeviceSetupGuide/en-us/GUID-C7B6F3FE-F71B-45CD-B5B4-AF946A2A9730.html ; BG2 variant .../InstallManual/BackupGateway/2/en-us/GUID-570320A6-25C3-48F1-B96D-7CFB4E1FF573.html
- Implication: the relay gives an instant, offline, grid-loss island signal; a small controller could take a delay (15 min) and SOC logic itself, but it needs its own SOC source (local Powerwall API), while the relay alone sheds immediately with no delay. The relay is configured by an installer in Tesla One, not by the owner.

## Open questions only the owner can answer
1. Does the Tesla app (Powerwall Settings) show a vehicle charging/off-grid threshold, and what is it set to?
2. Is the Wall Connector on the Powerwall-backed side (whole-home backup) and is the vehicle a Tesla?
3. Does the PW3 have Aux terminals wired, and is software 25.2+ installed; is Backup Gateway 2 or Backup Switch present?
4. Does the car have an off-grid charge-stop notice in its history from a past outage?
5. Is the solar DC-coupled (PW3 built-in) or is there any AC-coupled solar?
