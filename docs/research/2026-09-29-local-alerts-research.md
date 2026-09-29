# Local (no-internet) in-house alerts: research report, 2026-09-29

Confidence: [D] official doc, [C] credible project/forum, [S] speculation/inference.
Budget note: ~17 tool calls used; search snippets were summarised by the tool, and primary pages were fetched only for the ntfy known-issues page. Nothing here was tested on hardware.

## Recommendation (2 layers)
- Layer A, attention-getter in the house: mains Zigbee siren via Zigbee2MQTT (frient SIRZB-110/111 preferred, Heiman HS2WD-E has documented Z2M control issues), PLUS LIFX red pulse (works today, no purchase) PLUS a laptop speaker alarm tone (works today).
- Layer B, reaches the phone: there is NO reliable, no-purchase, no-internet way to wake a locked iPhone. Best available: (1) Apple home-hub + HomeKit sensor (Homebridge) is unproven offline and probably depends on iCloud/APNs [S]; (2) treat the phone as best-effort and make Layer A loud enough that the phone is unnecessary. Android: ntfy app with instant delivery over the LAN works with zero internet [C].
- Works NOW with no new hardware: LIFX SetWaveform pulse, laptop speaker (espeak/tone), ntfy Android app (if any Android users). Needs purchase: Zigbee dongle + siren (~$25 dongle, ~$35-60 siren), optionally a Sonos/HomePod.

## 1. iPhone push without internet
APNs: iOS background push only arrives via Apple's APNs, and only the app's publisher can send to it [D/C: ntfy docs, noted.lol]. With Comcast down, the phone (on home WiFi with no route out) cannot hold an APNs connection, so no APNs push. Confirmed.

### 1a. ntfy
- iOS: self-hosted server needs `upstream-base-url: https://ntfy.sh`; the server sends a poll_request to ntfy.sh -> Firebase -> APNs -> phone, then the phone fetches the message from your server [D: docs.ntfy.sh/config]. Without upstream, "delivery can take hours, depending on the state of the phone" [D]. During the outage the upstream hop cannot happen, so iOS gets nothing while backgrounded. Foreground/app-open: the app can subscribe to the LAN server directly (only while active) [C]. Verdict: unusable as the primary iPhone alert. Notification also does not use Critical Alerts; Focus/silent would suppress it [S].
- Android: without Firebase for self-hosted servers, "instant delivery" keeps a foreground service with a WebSocket/JSON stream to the LAN server; persistent notification required; battery ~0-1% per 17 h [D: docs.ntfy.sh/subscribe/phone, faq]. Works locally with zero internet [C]. Can use max priority to bypass DND (ntfy priority 5 with Android channel settings) [C, not verified in this pass].

### 1b. HomeKit via Homebridge/HAP + Apple hub
- Homebridge plugins exist to expose an MQTT-driven smoke sensor (homebridge-mqttthing supports SmokeSensor; homebridge-mqtt-smoke-alarm; mhawkshaw/homebridge-mqttsmokesensor) [C]. Laptop needs Node.js; Devuan/no systemd is fine with a sysvinit/runit script [S].
- Local control: HomeKit hub (Apple TV/HomePod) runs automations locally with internet down [C: makeuseof, smart-home guides]. Accessory pairing/HAP is LAN-local [D: Apple security guide].
- Notification delivery to iPhone: I found NO report or Apple doc confirming that Home app notifications (including Critical smoke/CO alerts) are delivered during an internet outage. Notifications to a phone at home still go hub -> iCloud/APNs -> phone as far as anyone documents; Apple's docs only describe the HomePod sound-recognition CRITICAL alert and don't state offline behavior [D/silent]. Assessment: probably fails without internet [S]. Also note: Critical alerts for smoke/CO are tied to the accessory service type, and Apple says a HomeKit smoke sensor triggers the critical notification, which breaks through silent/Focus [C, but only when delivery works].
- Verdict: worth a cheap test if an Apple TV/HomePod is already present (unplug WAN, trigger the sensor, watch a locked phone). Do not rely on it unproven.
- Also the HomePod itself can play a chime on a HomeKit event? Not confirmed; skip.

### 1c. Local Push Connectivity (NEAppPushProvider)
- Real API (iOS 14+): the app's extension holds a persistent connection to a local server when the phone joins configured SSIDs, and can post local notifications/report calls without APNs [D: developer.apple.com/documentation/networkextension/local-push-connectivity, WWDC20 10113].
- Entitlement is restricted, must be requested from Apple, and is aimed at enterprise/hospital/cruise-ship deployments [D]. Apple developer forum threads discuss lifecycle limits for safety-critical use (thread 807012) [C].
- I found no general-purpose consumer app (ntfy, Pushcut etc.) that uses it. Pushcut has a HomeKit guide but relies on its cloud/APNs [C]. Building your own app needs a paid developer account plus the entitlement request: not practical [S]. Local notifications posted by the extension would not be Critical Alerts unless that separate entitlement is also granted [S].

## 2. In-house audible/visual
### 2a. Zigbee sirens (Zigbee2MQTT)
- Heiman HS2WD-E: supported, router, exposes mode/level/strobe; several open Z2M issues report parameters ignored / not working (#14424, #20240, #13986) [C]. Mains + backup battery, available on Amazon US in some variants (verify plug type) [S].
- NEO/Tuya NAS-AB02B2: TS0601, Tuya-based, listed compatible on blakadder; battery/USB (mains via USB) siren with temp/humidity; detailed Z2M behavior not verified [C]. Battery models reported to drain fast (forum: "max 1 day" for some) [C].
- frient/Develco SIRZB-110 (and -111 mains): supported by Z2M [D: zigbee2mqtt.io/devices]. Best fit for reliability; -111 mains-powered per Z2M discussion. US availability of the 110 vs EU plug variants: verify [S].
- ThirdReality vibration sensor with 110 dB alarm: works with Z2M but is a sensor, not a siren [C].
- Zigbee is fully local: works with zero internet [D]. Latency ~1 s. It is not a phone alert; a whole-house siren at night wakes people.
- Dongle: Sonoff ZBDongle-E/P supported by Z2M; runs under Devuan with udev [C].

### 2b. LIFX
- LAN protocol message 103 SetWaveform: Saw/Sine/HalfSine/Triangle/Pulse, period, cycles, transient flag, skew ratio [D: lan.developer.lifx.com/docs/waveforms]. Pulse red for 30 s = e.g. period 1000 ms, cycles 30, transient=1 (returns to previous colour). Works entirely over UDP 56700, zero internet [D].
- Libraries: lifxlan (Python), aiolifx, photons, node-lifx-lan [C]. 2017 bulbs support the LAN protocol (firmware updates have been cloud-only, no matter). Bulbs must be powered on (a wall-switched-off bulb won't work). They should stay on the Powerwall-fed circuits: yes, all house.
- Does not wake a sleeping person reliably in a dark stormy night? A pulsing red bedroom bulb is actually quite effective at night [S]. Silent so useless for people asleep in rooms without a LIFX bulb.

### 2c. Speakers
- Sonos: local network playback works without internet according to Sonos community reports; but Sonos says operating offline is "not supported"; reports of playback stopping within seconds when internet drops during play [C]. SoCo `play_uri` with a URL served by the laptop (http.server) is the documented pattern [C: SoCo faq]. S2 controller app needs the LAN; UPnP still exposed [C]. Verdict: plausible, test it [S].
- HomePod via AirPlay from Linux: OwnTone and pyatv can stream to HomePods, but there are open issues with HomePod compatibility (owntone #2047, #1344, #1623) [C]. HomePod peer-to-peer AirPlay is possible without internet on the speaker [C: macobserver/idownloadblog]. Linux-to-HomePod offline: unverified [S]. HomePod also needs its own auth/pairing; newer HomePod software may demand HomeKit auth for AirPlay [S].
- Google Nest/Echo: your premise was that they need cloud. Search found: Cast devices use the Cast protocol on the local network, and local casting works without internet access per a third-party blog [C, weak]; pychromecast should work offline in theory [S]. Google Assistant/Alexa voice features definitely need cloud [D]. Initial setup needs internet; after a power/internet outage, some Nest devices show "no internet" and may stall [C: Apple/Google support pages]. Not verified; do not rely on it.
- Echo: uses Alexa cloud; no supported local API [S].

### 2d. Laptop speaker
- espeak-ng / `aplay` / `mpg123` alarm tone: zero dependencies, zero internet, trivial on Devuan; alsa. Limitation: laptop speaker is small and may be in one room; use external powered speaker for volume. Not wake-capable in other rooms.

## 3. Combination
Layer 1 (in-house, works with no internet): Zigbee siren (purchase) + LIFX red pulse (free) + laptop tone (free). Layer 2 (phone): none is reliable on iPhone. Options ranked: (a) HomeKit smoke-sensor via Homebridge and an Apple hub, test offline first; (b) ntfy iOS in foreground only; (c) build custom local-push app (not practical). For Android family members: ntfy with instant delivery. Since the laptop already knows grid state within ~10 s, publish to MQTT topic `alerts/grid`; subscribers: LIFX script, Z2M siren, espeak, Homebridge, ntfy. Also send an ntfy push to the upstream ntfy.sh normally, it will arrive when internet returns and on "grid restored" alerts if the internet is up.
Failure modes: Zigbee dongle on same laptop (single point); sirens can be annoying/false-trigger (add a 30 s debounce and a mute); if Powerwall drops and WiFi router is not on battery-backed circuit (house is all backed so ok); iOS Focus modes suppress everything except Critical Alerts.

## Sources
- https://docs.ntfy.sh/known-issues/ and https://github.com/binwiederhier/ntfy/blob/main/docs/config.md
- https://noted.lol/ntfy/
- https://docs.ntfy.sh/subscribe/phone/ , https://docs.ntfy.sh/faq/
- https://developer.apple.com/documentation/networkextension/local-push-connectivity
- https://developer.apple.com/videos/play/wwdc2020/10113/
- https://developer.apple.com/forums/thread/807012
- https://www.pushcut.io/guides/homekit
- https://www.makeuseof.com/switched-smart-home-off-cloud-finally-works-when-internet-down/
- https://support.apple.com/guide/homepod/alerts-smoke-carbon-monoxide-detectors-apd1f1921722/homepod
- https://www.npmjs.com/package/homebridge-mqttthing , https://github.com/mhawkshaw/homebridge-mqttsmokesensor
- https://www.zigbee2mqtt.io/devices/HS2WD-E.html , https://github.com/Koenkk/zigbee2mqtt/issues/14424 , /20240
- https://zigbee.blakadder.com/Neo_NAS-AB02B2.html
- https://www.zigbee2mqtt.io/devices/SIRZB-110.html , https://www.zigbee2mqtt.io/devices/SIRZB-111.html
- https://community.home-assistant.io/t/zigbee-comparison-of-sirens/720297
- https://lan.developer.lifx.com/docs/waveforms
- https://github.com/ChristopherThorpe/lifxlan , https://github.com/futomi/node-lifx-lan
- https://en.community.sonos.com/music-services-and-sources-228994/make-sonos-work-without-internet-6795315
- https://docs.python-soco.com/en/latest/faq.html
- https://github.com/owntone/owntone-server/issues/2047
- https://www.macobserver.com/tips/how-to/airplay-homepod-without-internet/
- https://www.googlenestcommunity.com/t5/Speakers-and-Displays/Local-Network-Access/m-p/252093

## Gaps (not verified)
No source confirmed or refuted HomeKit notification delivery during an internet outage; no app found using NEAppPushProvider for general use; Sonos S2 offline behavior is anecdotal; NAS-AB02B2 Z2M behavior and US siren availability unchecked; ntfy Android DND bypass unverified.
