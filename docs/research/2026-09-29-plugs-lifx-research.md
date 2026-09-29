# LIFX and smart plug research for LAN-only outage load shedder (2026-09-29)

Confidence labels: [OFFICIAL] vendor/project doc, [CREDIBLE] forum/issue tracker/project README, [SPEC] my inference, unverified. About 22 tool calls; no LAN contact.

## Q1. LIFX local control

Bottom line: LIFX LAN control is local UDP, needs no cloud token, and is the right way to do this. Confidence high on protocol and no-cloud; medium on "2017 bulbs work identically" (not directly confirmed by a source I could open).

- Protocol: binary UDP, port 56700, discovery by broadcast GetService(2) -> StateService(3). Official docs say to discover the device and use the port it reports. [OFFICIAL] https://lan.developer.lifx.com/docs/introduction , https://lan.developer.lifx.com/docs/communicating-with-device , spec repo https://github.com/LIFX/lifx-protocol-docs
- Cloud: LAN docs are separate from the HTTP (cloud) API; the LAN protocol needs no account or token. [OFFICIAL] same links. The docs warn against excessive message rates (no number given in what I read); keep commands paced. [OFFICIAL]
- 2017-era bulbs (A19, Mini, Color 1000): the LAN protocol has been on all LIFX bulbs since the original 2012 line; the docs page I read does not list per-model firmware support. [SPEC] Verify by one discovery packet from the box before the outage (do not do this from me).
- Outage/power cycle: local control works while the cloud is down (multiple vendor-support and integration pages). [CREDIBLE] https://support.lifx.com/hc/en-us/articles/36434861480727-LIFX-Appears-Disconnected , https://www.smarthomepoint.com/lifx-bulbs-no-internet/ . A bulb power-cycled with the router up but internet down rejoins WiFi from stored credentials and answers LAN; I found no source contradicting this, but no source explicitly testing it either. [SPEC, likely]
- Libraries: lifxlan (mclarkk, sync, fixed-source-port issue #114 makes firewall rules awkward: https://github.com/mclarkk/lifxlan/issues/114), aiolifx (asyncio, used by Home Assistant lifx integration), photons (delfick). Home Assistant's lifx integration is LAN-polling: https://www.home-assistant.io/integrations/lifx/ [OFFICIAL]. Note: Python libs not re-checked for 2026 maintenance status. [SPEC]
- Gotchas:
  - HA issue: bulbs stayed unavailable after a power loss until integration reload ("Duplicate LIFX discovery response ignored"), 2022, fixed by PR #71673. Lesson: after bulbs power-cycle, have your controller re-run discovery and tolerate changed IPs (DHCP). [CREDIBLE] https://github.com/home-assistant/core/issues/71585
  - 2.4 GHz only; bulbs can drop off weak WiFi; the "Restart" (power cycle) fixes DHCP renewals. [CREDIBLE] LIFX support link above. Reserve DHCP leases or track by MAC/serial, not IP.
  - Ownership: Feit Electric bought LIFX assets from Lifi Labs Aug 2022; said it would keep the app and cloud running. [CREDIBLE] https://homekitnews.com/2022/08/05/lifx-acquired-by-feit-electric/ , https://9to5mac.com/2022/08/08/lifx-lives/ . I found no evidence of a cloud shutdown or LAN-protocol removal; LIFX status page showed no incidents as of a 2026-08-29 snippet. https://status.lifx.com/ . Firmware changing the LAN protocol: no reports found. [SPEC: low risk]. Advice: do not update bulb firmware before the season.
  - Bulb "off" over LAN = standby (~0.5 W per bulb, your figure; not verified from a source). Bulb switched off at the wall is unreachable; to turn back on you need a plug/switch that restores power, and the bulb boots and rejoins WiFi (takes seconds to a minute). Power-on state after power restore is configurable in some firmware; default is on. [SPEC]

## Q2. Smart plugs

### Critical finding on TP-Link (a and the reboot question)

- python-kasa: newer Kasa and all Tapo devices use KLAP (port 20002 discovery) and need the TP-Link account username/password once. Devices never connected to the cloud may work without credentials. [OFFICIAL] https://python-kasa.readthedocs.io/en/latest/topics.html , https://github.com/python-kasa/python-kasa
- KLAP auth is local after that: the plug compares hashes stored at provisioning; no cloud round trip at control time. Home Assistant's tplink integration and forks store only the credential hash. [CREDIBLE] https://github.com/gigabytegrove/tapo-local (README describes this; it does not itself test outage reboot). You can store the hash instead of the plaintext password.
- PROBLEM: reboot with no internet. TP-Link support (Dec 2024) confirmed on a Tapo P110: the plug syncs its clock from the internet; after a power outage with no internet, "all time-related features will stop". User reported it enters "zombie mode": basic on/off still works but energy monitoring, settings and other features are lost until it gets internet once. [CREDIBLE, vendor support quote] https://community.tp-link.com/en/smart-home/forum/topic/719736
- Related: Kasa KP125M (Matter) plugs would not reconnect to Home Assistant after power loss without internet; issue opened 2024-03-13, marked "waiting-for-upstream", unresolved in what I could read. [CREDIBLE] https://github.com/home-assistant/core/issues/113196
- Tapo P100 users report plugs going offline after an internet outage until physically replugged. [CREDIBLE, weaker] https://community.tp-link.com/en/smart-home/forum/topic/235950
- Firmware risk: python-kasa issue #1752 (2026-09-04): P115(US) on latest firmware fails local KLAP with valid credentials, survives factory reset, unresolved. https://github.com/python-kasa/python-kasa/issues/1752 [CREDIBLE]. Also see https://community.tp-link.com/us/smart-home/forum/topic/861124 (KLAP thread).
- Verdict: TP-Link Kasa/Tapo are NOT reliable for this use case. They may keep on/off after a reboot without internet, but this is documented as flaky, and a firmware update can break local auth. Reasonable to use as fallback only, with auto-update disabled (Kasa/Tapo app setting) [SPEC].

### b. Matter (WiFi or Thread)
- Local by design; HA's Matter integration (python-matter-server) pairs over LAN/Thread and is described as fully local. Commissioning needs a phone or controller once; multi-admin. [CREDIBLE] https://community.home-assistant.io/t/looking-for-local-plug-with-energy-monitoring-matter-wifi/867254
- Matter plugs reporting energy (Aug 2026 list): Eve Energy (Thread, US available), Meross MSS315 (WiFi, US, energy arrived via firmware update), TP-Link Tapo P110M (WiFi, US), IKEA Grillplats (Thread). https://matter-smarthome.de/en/practice/which-matter-smart-plugs-report-energy-data/ [CREDIBLE]. Many cheap Matter plugs give on/off only (e.g., Eightree thread: no energy: https://community.home-assistant.io/t/eightree-matter-smart-plug-with-energy-monitoring-no-energy/892088).
- Thread plugs need a Thread border router (Apple TV/HomePod, Nest Hub, or HA SkyConnect/ZBT-1/ZBT-2 with OTBR) which adds a dependency; WiFi Matter plugs need none but keep WiFi dependency. Whether Matter WiFi plugs (KP125M, P125M) suffer the same clock-sync/reconnect quirk: KP125M evidence above says yes for at least one HA case. Meross MSS315 behavior offline: not verified. [SPEC]
- Chip-tool works for scripting but is heavy; python-matter-server is simpler.

### c. Zigbee with a USB coordinator (Sonoff ZBDongle-E/P, ZHA or Zigbee2MQTT)
- No WiFi, no cloud, no internet clock dependency; mains plugs act as routers. Energy-monitoring US options seen: ThirdReality Smart Plug Gen2 / E2 (energy monitor), OWON WSP404 (15 A, US), GE Zigbee plug 45853GE. Not confirmed for 15 A rating on each; check the label. [CREDIBLE-ish] https://smarthomescene.com/top-picks/best-smart-plugs-with-energy-monitoring-for-home-assistant-eu/ , https://3reality.com/product/smart-plug-gen2-with-energy-monitoring/ , https://deepwiki.com/Koenkk/zigbee2mqtt.io/6.4-smart-plug-and-power-monitoring-documentation
- Cost: coordinator ~$30 plus plugs $12-25. Needs a stable USB passthrough on Linux; Zigbee2MQTT + mosquitto or ZHA. Reboot behavior: devices rejoin their network automatically; the coordinator holds the network key; a controller box reboot needs the coordinator dongle unchanged. [SPEC]
- Sonoff S31 Lite zb / IKEA Inspelning: I did not verify US availability or exact ratings; Inspelning in US is 15 A? unverified.

### d. Z-Wave (Zooz ZEN04 800LR, ZEN15 800LR) with USB stick, Z-Wave JS
- ZEN15 is the heavy-duty plug (15 A, power monitoring, suitable for a dehumidifier; motors under 15 A per Zooz). ZEN04 is for lamps and small appliances, has power monitoring, not for motors over 1/3 hp; a 500 W dehumidifier compressor is a motor: use the ZEN15. [OFFICIAL/retailer] https://www.getzooz.com/zooz-zen15-power-switch/ , https://www.getzooz.com/zooz-zen04-smart-plug/ , https://theprivatesmarthome.com/review/zooz-zen15-power-switch/
- Sub-GHz (908 MHz US), better wall penetration, no WiFi. Price higher ($30-45 each). Z-Wave JS UI is mature. Reliability reputation good (reviews mostly Hubitat/HA). [CREDIBLE, retailer reviews]

### e. Pre-flashed ESPHome/Tasmota (Athom US plugs, Cloudfree)
- Fully local API/MQTT/HTTP, no accounts, no cloud, no clock dependency for on/off or energy. Athom PG03-US16A / V3 US plugs with energy monitoring (HLW8032) ship pre-flashed ESPHome or Tasmota. Downsides: bulky, blocks adjacent outlet; still WiFi; brand is small; check UL listing (Athom US plugs: not verified). https://www.athom.tech/blank-1/us-v2-plug-for-esphome , https://yuweiss.dev/review-athom-smart-plug/ [CREDIBLE]
- Reboot without internet: ESPHome/Tasmota reconnect to WiFi from stored config and serve HTTP/API/MQTT locally; on-state after power restore configurable (restore_mode) which matters for a dehumidifier. Firmware OTA updates are yours to control. [SPEC, but well-known design]

## Ranked top 3 for outage, no internet, Linux controller, reliability first

1. Zigbee plugs + Sonoff ZBDongle-E/P (Zigbee2MQTT or ZHA). No WiFi, no cloud, no time-sync surprise, mesh of mains routers, cheap plugs. Use a 15 A-rated energy plug (OWON WSP404, ThirdReality Gen2) for the dehumidifier; verify its rating. Avoids WiFi entirely. Confidence: medium (design sound; brand-level reliability from limited sources).
2. Z-Wave (Zooz ZEN15 for the dehumidifier, ZEN04 for lamps/TVs) + USB stick with Z-Wave JS. No WiFi, best range, proper heavy-load plug. Costlier. Avoids WiFi entirely. Confidence: medium-high on tech, retailer reviews only.
3. Pre-flashed ESPHome (Athom US16A energy plug). Fully local, no auth model, no vendor firmware surprises, WiFi on backup power is fine. Uses WiFi, so one more dependency; less certifications known. Confidence: medium.

Others:
- Matter WiFi (Meross MSS315, Tapo P110M): local, but offline reboot behavior unverified; Kasa KP125M had exactly the reboot-without-internet problem in HA. Medium-low.
- Matter Thread (Eve Energy): local, needs a Thread border router, pricier. Avoids WiFi for the plug but adds the border router. Medium.
- TP-Link Kasa/Tapo (KP115, EP25, P110, P115): not recommended as primary; zombie mode after outage per TP-Link support, KLAP/firmware breakage. Reasonable only for on/off if already owned.
- LIFX bulbs stay on LAN control (Q1); if practical, put lamps on a plug-controlled circuit only for shedding at the plug level, since LAN off leaves ~0.5 W standby (negligible).

## Unverified items to settle by tests before the season
1. Pull power on the chosen plug with WAN blocked at the router, confirm command works and energy reads after boot.
2. Same for one LIFX bulb: power-cycle with WAN down, run discovery from the box.
3. Confirm 15 A rating and UL listing on each candidate plug label.
