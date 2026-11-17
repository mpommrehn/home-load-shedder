# ZBDongle-P or ZBDongle-E for the Zigbee coordinator (2026-11-08)

Confidence labels: [SOURCED] seen in search results, page not opened; [SPEC] inference, unverified. One round of web searches, no hardware contact. Moved here on 2026-11-17 from a public project's plug write-up, where it did not belong.

## Question

Which Sonoff USB Zigbee coordinator to buy for the shedder's Zigbee plugs (ThirdReality Gen2 metering plugs, see `docs/plugs-and-coordinator-summary.html`): the ZBDongle-P or the ZBDongle-E. What does the higher price of the E buy?

## Comparison

| | ZBDongle-P | ZBDongle-E |
|---|---|---|
| Chip | TI CC2652P | Silicon Labs EFR32MG21 |
| Zigbee2MQTT stack | Z-Stack, the mature path [SOURCED: Sonoff's own guide pairs P with Z-Stack, E with EmberZNet] | EmberZNet, newer |
| Radio | no range figure found | marketed at +20 dBm [SOURCED] |
| Thread or Matter | nothing found | possible with different firmware [SOURCED, one user report] |
| Amazon price seen 2026-10-10 | $28.99 (B0FQ712KM8) | $36.99 (B0FNLP58JS) |

Prices came from search results; seller and stock were not checked.

## Findings

- The E buys nothing a Zigbee plug network needs. Its extras are the +20 dBm marketing figure and a Thread or Matter option. [SOURCED]
- No independent range test for either dongle turned up. A range advantage for the E is marketing until measured. [SPEC]
- One Sonoff-linked review says the E's chip is not recommended with Zigbee2MQTT. Unverified and possibly dated; Zigbee2MQTT's supported-adapters page would settle it. [SOURCED]
- Sonoff now sells a Dongle Plus MG24 as the E's successor. [SOURCED]

## Recommendation

The P. Check Zigbee2MQTT's supported-adapters page before buying.

## Decision and open conflict

HLS-088 (home-load-shedder), decided 2026-11-17: the ZBDongle-P. Nothing bought.

The P is a USB dongle. GB-021 (gbox), decided 2026-11-02, chose a LAN coordinator so the shedder and gbox can run on separate machines, and GB-025 (gbox) still asks which LAN model to buy. `docs/design.md` still says a USB coordinator on the Devuan laptop. Whether the P is a stepping stone or a replacement for the LAN unit is HLS-259 (home-load-shedder), open.

## Sources

- [Sonoff ZBDongle-E (CNX Software)](https://cnx-software.com/?p=102119)
- [Sonoff Zigbee and Thread dongle choosing guide](https://sonoff.tech/blogs/news/sonoff-zigbee-thread-dongle-choosing-guide)
- [Sonoff Dongle Plus MG24](https://sonoff.tech/products/sonoff-zigbee-thread-usb-dongle-dongle-plus-mg24)
