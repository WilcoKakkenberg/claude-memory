---
name: feedback-no-guessing-ha-packages
description: "Never guess unverified facts (IP ranges, entity IDs, device names, etc.) when writing Home Assistant packages/config"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e92d54a2-547d-460f-8ff7-10100628d5f6
  modified: 2026-08-18T18:06:46.407Z
---

Do not guess unverified technical facts (e.g. IP/subnet ranges, entity IDs, device attributes) when creating or editing Home Assistant packages. Always ask the user or verify from the repo/live state first.

**Why:** In [[project_public_ip_security_checks]]-adjacent work, an assumption was made that the main LAN was `192.168.178.0/24` (AVM Fritz!Box factory default) purely because the confirmed guest network was `192.168.179.0/24`. This was wrong/unverified and the user called it out as guesswork they don't want.

**How to apply:** Before writing IP ranges, entity IDs, device attributes, or other concrete values into templates/automations/packages, either (a) find the value in the repo/config, (b) verify against live HA state, or (c) explicitly ask the user. Never present an assumption as fact without flagging it clearly — and prefer asking outright over flagging-and-proceeding.
