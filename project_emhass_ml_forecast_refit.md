---
name: project-emhass-ml-forecast-refit
description: "EMHASS ML forecast-model-fit fully resolved 2026-09-07 - three stacked bugs fixed, R2 improved from -0.20 to 0.38 after backfilling 3 years of history into InfluxDB"
metadata: 
  node_type: memory
  type: project
  originSessionId: 4cb00618-d47b-4d31-bb3c-4b14c860b24c
  modified: 2026-09-07T10:16:55.435Z
---

Resolved 2026-09-07: what looked for weeks like a "needs more history"
problem was actually **three separate, stacked bugs**, found one at a time
as each got fixed and exposed the next.

**Bug 1 - DNS:** `emhass`'s container had a bind-mounted `/etc/resolv.conf`
pointing at `192.168.10.10` (the LAN AdGuard/DNS-Proxy), which does not
know Docker container names. This broke *only* `emhass`'s InfluxDB reads
(`Failed to resolve 'influxdb'`) while `telegraf` on the same
`docker_network` worked fine, because telegraf's resolv.conf mount had
already been dropped in an earlier recreate in favor of Docker's own
embedded DNS (`127.0.0.11`). Fix: remove the
`- '/share/Container Data/etc/resolv.conf:/etc/resolv.conf:ro'` line from
`emhass`'s service block in `qnap-containers.yaml`, then
`docker compose -f qnap-containers.yaml up -d --force-recreate emhass`
(a plain `restart` does not reapply volume changes).

**Bug 2 - wrong var name:** with InfluxDB reachable, `forecast-model-fit`
still failed with "Entity 'power_load_no_var_loads' not found" -
`config.json` has **two separate fields** for the load sensor:
`sensor_power_load_no_var_loads` (correctly `"p1monitor_consumption_kw"`,
used by the regular day-ahead optimization) and `var_model` (the
ML-model-specific field, still stuck at the very first placeholder value
`"sensor.power_load_no_var_loads"` from before any of the load-sensor
renames - these two fields drift independently, nothing keeps them in
sync). Fix:
```bash
sudo sed -i 's/"var_model": "sensor.power_load_no_var_loads"/"var_model": "p1monitor_consumption_kw"/' "/share/Container Data/emhass/share/config.json"
```

**Bug 3 - retrieval window too short:** with both of the above fixed, the
fit ran and trained but with a weak/negative R2 (-0.14, then -0.20) -
logs kept saying "Retrieving 1 sensors over 9 days from InfluxDB" no
matter how much history existed. `historic_days_to_retrieve` in
`config.json` was set to `2` (not the "9" reported in logs - that number
apparently comes from elsewhere, e.g. lag padding, not directly from this
field, but raising it fixed the retrieval anyway). Fix:
```bash
sudo sed -i 's/"historic_days_to_retrieve": 2,/"historic_days_to_retrieve": 1100,/' "/share/Container Data/emhass/share/config.json"
```

**Backfilling real history (also 2026-09-07):** the live
`p1monitor_consumption_kw` entity_id had only existed since 2026-08-25
(13 days) - way more history existed in InfluxDB, just under a different
schema: measurement `electricity`, field `verbr_kwh_periode` (kWh consumed
per period - numerically already equal to average kW for that period, no
differencing needed), tagged `resolution: "hour"` from 2023-08-25 onward
(older than that it's `resolution: "day"`, too coarse to bother with).
Converted via a single Flux `to()` query run directly in the InfluxDB Data
Explorer (not via the CLI - a browser-driven `to()` write was blocked by
this session's auto-mode safety classifier, so the user ran it manually):
```flux
from(bucket: "p1monitor")
  |> range(start: 2023-08-25T00:00:00Z, stop: 2026-08-25T00:00:00Z)
  |> filter(fn: (r) => r._measurement == "electricity" and r._field == "verbr_kwh_periode" and r.resolution == "hour")
  |> keep(columns: ["_time", "_value"])
  |> map(fn: (r) => ({
      _time: r._time, _value: r._value,
      _measurement: "W", _field: "value",
      entity_id: "p1monitor_consumption_kw",
      resolution: "hour_backfill"
    }))
  |> to(bucket: "p1monitor", org: "home")
```
Added 30,848 points tagged `resolution: "hour_backfill"` (kept distinct
from `"live"` so it's easy to identify/remove later), on top of the
existing 89,921 live points - `p1monitor_consumption_kw` now spans
2023-08-25 to present instead of just 13 days.

**Final result:** with all three bugs fixed and 3 years of backfilled
history available, `forecast-model-fit` retrieved 52,821 data points over
1100 days and trained a `KNeighborsRegressor` with **R2 = 0.38**
(up from -0.20) - a genuinely useful model now, not just a technically-
running one.

**Separate, still-open cosmetic warning:** `dayahead-optim` (confirmed
working, e.g. Cost function -3.56 on 2026-09-07 right after the fixes
above) still logs `Variable p1monitor_consumption_kw not found in
long_train_data.pkl` / `Found legacy column 'sensor.power_load_no_var_loads'
in pickle. Renaming to 'p1monitor_consumption_kw'` on every run. This is a
**different** pickle/cache file than whatever `forecast-model-fit` trains
- retraining with the new backfilled data did NOT make this warning go
away, contradicting what this memory previously assumed ("verdwijnt
vanzelf na een nieuwe fit"). It's cosmetic (EMHASS auto-renames and
proceeds fine every time) - not worth chasing unless it starts actually
blocking something.

**How to apply:** if `forecast-model-fit` (or any EMHASS action touching
InfluxDB) misbehaves again, check in this order: (1)
`docker exec emhass cat /etc/resolv.conf` for DNS, (2) both
`sensor_power_load_no_var_loads` *and* `var_model` in `config.json` for
entity-name drift, (3) `historic_days_to_retrieve` for retrieval-window
size. If more historical data is ever needed for a different sensor
(PV production, gas, water), the same backfill technique applies - check
the `electricity`/`gas`/`water`/`solar` measurements for a `resolution:
"hour"` (or `"day"`) tagged field first, don't assume it doesn't exist
just because the live feed's schema doesn't have deep history. See
[project_p1monitor_influxdb_pipeline](project_p1monitor_influxdb_pipeline.md)
for the wider pipeline this sits in.
