---
name: project-p1monitor-influxdb-pipeline
description: "Working architecture for P1 Monitor -> MQTT -> Telegraf -> InfluxDB -> EMHASS/HA, confirmed live 2026-09-01"
metadata: 
  node_type: memory
  type: project
  originSessionId: 4cb00618-d47b-4d31-bb3c-4b14c860b24c
  modified: 2026-09-22T10:32:53.968Z
---

Live, confirmed-working architecture (as of 2026-09-01):

```
P1 Monitor -> MQTT (mosquitto, 172.10.1.13:1883) -> Telegraf (172.10.1.26) -> InfluxDB 2.7 (172.10.1.23, bucket "p1monitor", org "home")
                                                                                    |
                                                                                    v
                                                                                 EMHASS (use_influxdb: true)
                                                                                    |
                                                                                    v
                                                                              Home Assistant (publish-data)
```

**Why this exists:** HA's own API/MQTT load from EMHASS polling HA directly for P1 Monitor sensor history had become too high (exact metric that was too high not preserved in memory — ask user if it resurfaces). Bypassing HA entirely for EMHASS's *input* data (MQTT -> Telegraf -> InfluxDB direct) removes that load. EMHASS still *publishes* its results back into HA normally (`sensor.p_pv_forecast`, `sensor.optim_status`, etc.) — only the input side changed.

**Key implementation facts, all git-committed on `feature/p1monitor-mqtt-influxdb` and already deployed live:**

- `telegraf/telegraf.conf`: one `[[inputs.mqtt_consumer]]` block *per topic* (28 total: smartmeter kWh/kW high/low + tarifcode + gas, all 15 phase topics, 3 watermeter topics) — not a single wildcard subscription, because each topic needs a different `name_override` (= the unit: W/kWh/m3/L/V/A/count/tarifcode) and a distinct `entity_id` tag (`p1monitor_<name>`).
- Schema deliberately mirrors HA's classic InfluxDB output plugin (measurement=unit, tag `entity_id`, field `value`) specifically so EMHASS's `use_influxdb` mode (`SELECT mean("value") FROM "<measurement>" WHERE "entity_id"='<id>'`, confirmed from EMHASS's `retrieve_hass.py`) can read it via InfluxDB 2.x's v1-compatibility API.
- `tarifcode` (string "D"/"P") is its own dedicated measurement — mixing a string payload with float payloads under one measurement+field causes an InfluxDB type conflict.
- The `tarifcode` mqtt_consumer needed an explicit `client_id` (`telegraf-p1monitor-tarifcode`) after auto-generated client_ids collided across the 28 consumer inputs, silently killing that one subscription (~2026-08-25).
- Telegraf connects to MQTT via the internal docker-network IP (`172.10.1.13`), not the external LAN IP.
- Bucket is `p1monitor` (org `home`) — this is now the ONLY InfluxDB bucket in use (see below).
- EMHASS config (`/share/Container Data/emhass/share/config.json` on the NAS): `sensor_power_load_no_var_loads: "p1monitor_consumption_kw"` is *not* an HA entity_id — with `use_influxdb: true` it's the InfluxDB `entity_id` tag value to query. Don't "fix" this back to a `sensor.*` HA entity id, that would break it.
- `p1monitor_consumption_kw` now spans 2023-08-25 to present (not just since the live pipeline's 2026-08-25 start): 30,848 points tagged `resolution: "hour_backfill"` were added 2026-09-07 by converting the pre-existing `electricity`/`verbr_kwh_periode` (`resolution: "hour"`) backfill data into this schema via a Flux `to()` query, on top of the 89,921 `resolution: "live"` points. See [project_emhass_ml_forecast_refit](project_emhass_ml_forecast_refit.md) for the exact query and why it was needed (EMHASS's ML model had almost no usable history otherwise).

**`energy` bucket removed (2026-09-01):** InfluxDB used to also have a separate `energy` bucket (org `home`, default bucket from container init) fed by HA's own built-in `influxdb:` integration, documented in `packages/energy_influxdb.yaml`. That YAML file was deleted from master on 2026-08-21 (commit `eea07425`) — after that, the HA integration's connection stayed configured via the UI but lost its entity filter and `override_measurement: state` setting, so it degraded into an unfiltered dump of ALL HA entities (weather, disk usage, persons, etc., not just energy). Verified via InfluxDB Data Explorer that both Grafana dashboards ("P1Monitor — Historie" and "P1Monitor — Live") query `from(bucket: "p1monitor")` exclusively — nothing depended on `energy`. User deleted the `energy` bucket on 2026-09-01. If HA's InfluxDB integration (Settings > Devices & Services) is still enabled pointing at a now-nonexistent bucket, it will start erroring — worth checking/removing that integration if it resurfaces as a HA log warning.

**EMHASS-specific DNS breakage (found 2026-09-07):** `emhass`'s compose
service had a bind-mounted `/etc/resolv.conf` pointing at the LAN DNS
(`192.168.10.10`), which can't resolve Docker container names — this
broke `emhass`'s InfluxDB reads (`Failed to resolve 'influxdb'`) while
other containers on `docker_network` (e.g. `telegraf`) were fine, because
their resolv.conf mount had already been dropped in an earlier recreate
in favor of Docker's embedded DNS (`127.0.0.11`). If any container on
this network suddenly can't resolve another container's name, compare
`docker exec <container> cat /etc/resolv.conf` against a working sibling
before assuming a network-level problem — it may just be a stale
bind-mounted resolv.conf on that one service. See
[project_emhass_ml_forecast_refit](project_emhass_ml_forecast_refit.md)
for the full incident (this DNS bug plus two further config bugs -
`var_model` and `historic_days_to_retrieve` - it uncovered, plus the
history backfill above).

**Historical-import (from a P1 Monitor SQL export):** separate from the live MQTT->Telegraf pipeline above, there used to be a local historical import (on the old Windows 11 laptop, `C:\Users\Wilco\Documents\P1monitor\`) that loaded readings from a P1 Monitor SQL export into InfluxDB (measurement `electricity`, field `verbr_kwh_periode`, tagged `resolution: "hour"`/`"day"` - this is the data the 2026-09-07 Flux backfill in [project_emhass_ml_forecast_refit](project_emhass_ml_forecast_refit.md) later converted into the `p1monitor_consumption_kw` schema). Mechanism confirmed by user 2026-09-22: via Telegraf's `[[inputs.sql]]` plugin (consistent with Telegraf already being the tool for MQTT->InfluxDB in this pipeline), NOT a standalone script.

**P1 Monitor's real export format/schema (confirmed 2026-09-22)** by inspecting an actual re-downloaded export (`p1mon-sql-export<ts>.zip`, extracted on the Fedora laptop into `/home/wilco/github/P1monitor/`): it's a zip of per-table files, each a sequence of `replace into <table> (...) values (...)` (or `update config set ...` for the config table) statements - not a real `mysqldump` (no `CREATE TABLE`), and misidentified as "CSV" by the `file` command despite being SQL. Key table: `e_history_min` (minute resolution, spans 2020-01-01 to present - further back than the 2023-08-25 currently backfilled into InfluxDB). Columns: `TIMESTAMP, VERBR_KWH_181, VERBR_KWH_182, GELVR_KWH_281, GELVR_KWH_282, VERBR_KWH_X, GELVR_KWH_X, TARIEFCODE, ACT_VERBR_KW_170, ACT_GELVR_KW_270, VERBR_GAS_2421`. `_181`/`_182`/`_281`/`_282` are cumulative meter readings (DSMR OBIS 1.8.1/1.8.2/2.8.1/2.8.2, low/high tariff consumption/delivery); `VERBR_KWH_X`/`GELVR_KWH_X` are already-computed **per-minute deltas of whichever tariff `TARIEFCODE` currently has active** (verified by hand: a row's `GELVR_KWH_X` equals that row's cumulative `GELVR_28x` minus the previous row's, for the active tariff) - i.e. `VERBR_KWH_X` already IS the per-minute version of the target `verbr_kwh_periode`, just needs summing per hour/day. Other tables in the export: `weer_history_uur`/`weer` (weather), `temperatuur`, `faseminmax_dag` (per-phase daily min/max), `powerproduction_solar`, `watermeter`, `e_financieel_dag` (daily cost), `statistics`, `config`.

A from-scratch Telegraf config template using this confirmed schema (hourly `SUM(VERBR_KWH_X)` grouped from `e_history_min`) lives at `/home/wilco/github/P1monitor/p1monitor-historical-import.conf` - the query logic is verified by hand against sample rows but the template has not actually been run end-to-end yet (run as a one-off `telegraf --config ... --once` against a locally restored copy of the dump). If more history is ever wanted for EMHASS's ML model, this export already has genuine minute-resolution data back to 2020-01-01 - much further than InfluxDB's current 2023-08-25 backfill start.

**How to apply:** before proposing changes to this pipeline, re-read `telegraf/telegraf.conf` and the EMHASS `config.json` live from the NAS rather than assuming — this area has already been debugged through several non-obvious gotchas (see above) that are easy to accidentally "fix" back into a broken state. Also don't assume a repo file still exists just because it was read earlier in a session — `packages/energy_influxdb.yaml` is a confirmed example of a file that got deleted from master mid-session. See also [project_emhass_ml_forecast_refit](project_emhass_ml_forecast_refit.md) for the related ML-cache warning (variable name changes each time the load sensor reference changes, currently `p1monitor_consumption_kw`).
