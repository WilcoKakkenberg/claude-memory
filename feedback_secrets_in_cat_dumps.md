---
name: feedback-secrets-in-cat-dumps
description: "User's broad `cat *.yaml`/`cat config/` dumps on the QNAP repeatedly sweep up secrets files - never use the values, always flag and ask for rotation"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 4cb00618-d47b-4d31-bb3c-4b14c860b24c
  modified: 2026-09-01T19:42:29.504Z
---

When the user pastes QNAP file contents via `sudo cat` to get documentation
material (for MkDocs), broad globs (`cat *.yaml`, `cat config/*`) have
repeatedly swept up secrets alongside the config: an InfluxDB CLI admin
token (`influx-configs`) and an ESPHome `secrets.yaml` (WiFi password, API
encryption key, OTA password, AP password) both landed in chat this way on
2026-09-01, on top of an earlier raw HA long-lived access token pasted
directly.

**Why this matters:** these are real, active credentials for shared home
infrastructure (WiFi affects every device in the house, InfluxDB admin
token has full read/write on all buckets). Treat every credential that
lands in chat as compromised, same handling each time:

**How to apply:**
1. Never write the literal secret value into any file (docs, code, memory,
   anywhere) - not even to show "what it looks like" or redacted-but-
   partially-visible.
2. Tell the user plainly, in that turn, that the value is exposed and
   should be rotated - don't silently omit it and move on.
3. Still do the useful work: document the *structure* (file exists at path
   X, holds these named secrets, referenced via `!secret foo` from config
   Y) without the values - this is usually still exactly what the docs
   task needs.
4. Worth proactively noting when a requested `cat` is likely to hit a
   secrets file (e.g. `cat *.yaml` in a dir known to contain
   `secrets.yaml`) - a narrower glob avoids the exposure in the first
   place, though the user drives what they paste.
