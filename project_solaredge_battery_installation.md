---
name: project-solaredge-battery-installation
description: "SolarEdge battery expected to be installed 2026-09-07 - impacts EMHASS config, zero-export automation, and InfluxDB/Grafana"
metadata: 
  node_type: memory
  type: project
  originSessionId: 4cb00618-d47b-4d31-bb3c-4b14c860b24c
  modified: 2026-09-01T16:41:52.951Z
---

Wilco expects a SolarEdge battery to be installed on **maandag 2026-09-07**
("als het goed is" - not fully certain yet, could slip).

**Why this matters:** several already-built pieces currently assume
`battery: false` / no battery, and were explicitly designed to work around
its absence:

- EMHASS's own config has `battery` disabled - once installed, this needs
  `battery: true` plus capacity/charge-discharge-power/SoC parameters, and
  EMHASS can then genuinely dispatch battery charge/discharge as part of
  its optimization (relevant to the "option 1: EMHASS-driven automations"
  work just started, see [[project_p1monitor_influxdb_pipeline]] and the
  EMHASS day-ahead automation in `packages/energie/emhass_integration.yaml`).
- [[project_zonnepanelen_jaarbalans]]'s real-time zero-export automation
  (`packages/energie/zonnepanelen_zeroexport.yaml`) has an explicit code
  comment acknowledging its inverter-throttling approach only "dicht het
  export-lek overdag" and that genuine net-import for the saldo-inloop
  ("de motor van de inloop") should come from "de batterij-grid-charge en
  de PHEV-burn-down" - i.e. the batterij was already assumed as the real
  mechanism, this automation was always the stopgap.
- No SolarEdge battery entities (SoC, charge/discharge power) exist yet in
  HA or in any InfluxDB/Grafana dashboard - old `energy_influxdb.yaml` had
  a `# TODO: batterij-entiteiten toevoegen zodra de SolarEdge-batterij
  geinstalleerd is` comment, but that whole file was deleted 2026-08-21 (see
  [[project_p1monitor_influxdb_pipeline]]) before the battery ever arrived,
  so that TODO's original destination no longer exists.

**How to apply:** once the user confirms the battery is actually installed
(don't assume it happened just because 2026-09-07 has passed - ask/verify
first), this unlocks/requires:
1. Updating EMHASS's `config.json` (battery params) - verify real capacity/
   power specs from the installer, don't guess.
2. Revisiting the zero-export automation's role now that a real
   "motor"/import mechanism (battery grid-charge) exists.
3. Deciding where new battery entities should be tracked (HA only, or also
   a bucket/dashboard - `p1monitor` bucket is telegraf/MQTT-only, so a new
   ingestion path would be needed if InfluxDB history is wanted).
4. Updating MkDocs (`architectuur/energie-dataflow.md` and the EMHASS/
   zero-export pages) once the above lands, same pattern as the
   `energy`-bucket cleanup - don't let docs drift from reality again.
