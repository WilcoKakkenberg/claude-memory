---
name: snapserver-bluetooth-open-items
description: "Openstaande punten Snapserver-Bluetooth (Synergy 65 als snapclient, MkDocs bijgewerkt en gemerged) per 2026-10-26"
metadata:
  node_type: memory
  type: project
  originSessionId: b95a9a90-eaf0-467d-a759-52272b569cda
  modified: 2026-10-26T17:53:30.890Z
---

Op 2026-10-26 werkt Pixel 9a → BlueALSA → `process:///usr/local/bin/bt-capture` → Snapserver-stream Bluetooth/Auto op de snapserver-Pi (192.168.10.110). Gedocumenteerd in MkDocs-repo (`~/github/MkDocs`), branch `docs/snapserver-bluetooth`, gemerged in main (PR #1). Vervolg `docs/snapserver-config` (snapserver.conf + buffer 400 ms) gemerged in main (PR #2).

Open: Bluetooth-speaker Synergy 65 (`FC:58:FA:1F:16:0E`) als extra snapclient via `bluealsa:DEV=...,PROFILE=a2dp`, service `snapclient-synergy` (nog niet aangemaakt). Speaker was niet beschikbaar; gebruiker wil dit "de volgende keer" doen. Eerst: `speaker-test` naar de speaker en `systemctl cat snapclient` (User) bekijken; daarna latency in snapweb, eventueel reconnect-timer, en MkDocs bijwerken.

**Why:** gebruiker stelde het uit tot de speaker weer beschikbaar is.
**How to apply:** bij Snapcast/Bluetooth-vragen hier verder gaan i.p.v. opnieuw uitzoeken. Let op: snapserver is 32-bit (kan BlueALSA-plugin niet laden), snapclient is 64-bit (kan het wel); WirePlumber-Bluetooth staat uit.
