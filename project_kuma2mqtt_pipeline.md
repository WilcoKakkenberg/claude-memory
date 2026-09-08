---
name: project-kuma2mqtt-pipeline
description: kuma2mqtt.sh bridges Uptime Kuma monitor status to MQTT; new/retagged monitors need an uptime-kuma restart to appear
metadata: 
  node_type: memory
  type: project
  originSessionId: ad49bf27-a591-4b5f-8854-5b1ad9964846
  modified: 2026-09-05T08:47:03.380Z
---

kuma2mqtt.sh (`/share/Container Data/kuma2mqtt/kuma2mqtt.sh` on Wilco-QNAP, runs in the `kuma2mqtt` container) polls Uptime Kuma's Prometheus `/metrics` endpoint every `INTERVAL` (default 60s) and republishes each `monitor_status{...}` line to MQTT at `uptimekuma/monitor/<tag>/<check>/status` (retained). The `<tag>` is derived from whichever non-standard Prometheus label key is present (i.e. the Uptime Kuma Tag assigned to that monitor, name+value concatenated and slugified, e.g. tag "Docker Host: Wilco-QNAP" → `dockerhostwilcoqnap`). Monitors with no Tag fall back to `DEFAULT_TAG=overig`.

**Known bug (found 2026-09-05):** Uptime Kuma's built-in `/metrics` exporter uses an in-memory monitor list that is NOT refreshed when a monitor is created or its Tags are edited via the UI/API. New or newly-retagged monitors are simply absent from `/metrics` (not even under `overig`) until Uptime Kuma itself is fully restarted (`docker restart uptime-kuma`) — restarting the `kuma2mqtt` container itself does nothing, since it only pulls fresh data from Kuma each cycle.

**Why:** Confirmed by curl'ing `/metrics` directly inside the `kuma2mqtt` container (`docker exec -it kuma2mqtt sh -c 'curl -sf -u ":${API_KEY}" "$KUMA_URL" | grep monitor_id=\"<id>\"'`) before/after restarting uptime-kuma — monitors 67 (kuma2mqtt), 68/69 (mkdocs-sync/web), 49 (emhass), 70/71/72 (Device Tracker group + Pixel-9a + Galaxy-a56-5g) were completely missing pre-restart and appeared correctly tagged immediately after `docker restart uptime-kuma`.

**How to apply:** Whenever a monitor in this Kuma instance seems to be missing from MQTT Explorer, or lands in `uptimekuma/monitor/overig/...` despite having a Tag assigned in the Kuma UI, first check: was this monitor created or retagged recently? If so, the fix is `docker restart uptime-kuma` — not touching kuma2mqtt.sh or the tag itself. Also: kuma2mqtt.sh publishes with mosquitto retain (`-r`) and never clears old topics, so after a tag change or rename, the OLD topic/tag combo stays as a stale retained ghost in MQTT Explorer until manually deleted (right-click topic → delete) — this looks identical to the "still in overig" symptom but is a separate, cosmetic issue.

Related: [[project_p1monitor_influxdb_pipeline]]
