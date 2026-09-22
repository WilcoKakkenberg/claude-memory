---
name: project-p1monitor-3x25a-downgrade
description: Investigating downgrading grid connection from 3x35A to 3x25A using P1Monitor per-phase data - saves money yearly
metadata: 
  node_type: memory
  type: project
  originSessionId: 22d440e6-8902-457c-9856-1e481c1c0e0f
  modified: 2026-09-22T18:25:53.994Z
---

Wilco wants to find out if the electricity grid connection can be downgraded
from **3x35A to 3x25A** ("scheelt erg veel geld op jaarbasis").

**Analysis done 2026-09-22** on `04_faseinformatie.db` ->
`faseminmax_dag` table (daily per-phase min/max, 646 days,
2023-11-02 to 2025-08-08 - see [[project_p1monitor_influxdb_pipeline]] for
where this db lives, `/home/wilco/github/P1monitor/04_faseinformatie.db`):

- **Peak current per phase, 646 days:** never exceeded 29A on any phase.
  Only **1 day** (2024-04-08, L3=27A) exceeded 25A at all (0.2% of days).
  19 days (2.9%) exceeded 20A. This alone looks like a comfortable margin
  for 3x25A, especially since gG hoofdzekeringen tolerate brief overload
  above nameplate before tripping.
- **Clear structural phase imbalance:** L3 carries far more load than L1/L2
  (median peak-power ratio L3/L1 = 4.1x, p90 = 18.7x). L3 alone exceeded
  16A on 140/646 days (~22%); L1 never did. Rebalancing circuits across
  phases (move some load off L3 onto L1) would add further margin if ever
  needed, but isn't necessary based on the data so far.
- **Data quality gotchas found in this table** (filter before trusting any
  aggregate/max): two clear sensor glitches (`MAX_VERBR_L1_KW`=73.0 on
  2025-01-29 while `MAX_L1_A` that day was only 3A; `MAX_GELVR_L2_KW`=703.0
  on 2024-05-26) and sporadic (~monthly per phase) literal `0.0` /
  physically-impossible minimum voltage readings (e.g. `MIN_L1_V`=34.7V on
  2024-06-03) from P1-dongle telegram read glitches - not real electrical
  events. One genuine 3-phase voltage dip found: 2023-11-10, all phases to
  ~196V simultaneously (below EN50160 -10% margin), consistent across
  phases so likely a real grid-side event, not a sensor artifact.

**Critical gap - do not act on the above alone:** the P1 Monitor's
per-phase logging (`faseinformatie`/`faseminmax_dag`) stopped recording
after **2025-08-08** and stayed off for ~13.5 months. Wilco confirmed
2026-09-22 he just **re-enabled the logging** ("ik heb de logging weer
aangezet"). This means the analysis above has **zero coverage of the
period during/after the SolarEdge battery installation**
(see [[project_solaredge_battery_installation]], expected 2026-09-07 -
not yet independently confirmed installed in this conversation, per that
memory's own "don't assume, ask/verify" rule). Battery charge/discharge
current could push per-phase peaks meaningfully higher than this
historical picture shows.

**How to apply:** before recommending Wilco actually request the 3x25A
downgrade from the netbeheerder, get a fresh read of `faseminmax_dag`
covering at least a few weeks since the 2026-09-22 logging re-enable
(ideally spanning both a sunny/high-charge day and a dark/cold high-import
day), re-run the same >16/20/25/29A threshold analysis, and check it
against the actual max AC current spec of the SolarEdge inverter/battery
(don't guess the spec - ask or look it up from the installer docs). Only
then is the historical margin trustworthy for a downgrade decision.
