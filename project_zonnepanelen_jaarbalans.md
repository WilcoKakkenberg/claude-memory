---
name: project-zonnepanelen-jaarbalans
description: "Zonnepanelen jaarbalans-inloop + terugleverkosten-guardrail automation, deadline 11-11-2026 (einde salderingsregeling/vast contract)"
metadata: 
  node_type: memory
  type: project
  originSessionId: a6a3dbf0-49f6-4611-ab8f-3bc5aa57105c
  modified: 2026-08-27T19:45:18.144Z
---

Wilco's vaste energiecontract en de salderingsregeling lopen af op 11-11-2026. Tot dan wil hij twee dingen tegelijk bereiken via `packages/energie/zonnepanelen_zeroexport.yaml`, aangestuurd door het real-time bijsturen van `number.solaredge_modbus_active_power_limit`:

1. **Jaarbalans gelijktrekken** ("saldo-inloop"): hij heeft een historische achterstand (veel meer teruggeleverd dan verbruikt sinds 1 jan 2026), en wil die bewust inlopen door soms net iets meer te importeren dan puur zero-export zou vereisen — ook al kost dat geld per kWh (bewuste keuze, niet per ongeluk).
2. **Terugleverkosten-guardrail** (toegevoegd in v2.0.0, 27-08-2026): bruto teruglevering over het contractjaar (nulpunt 10-11-2025) mag de 2.000 kWh-schaalgrens niet overschrijden (~€32,52 per schaalsprong op de eindafrekening). Deze guardrail heeft voorrang boven de saldo-inloop-logica.

**Why:** Financieel gedreven, harde einddatum. De twee doelen gebruiken bewust verschillende nulpunten (saldo: 1 jan 2026; guardrail: 10 nov 2025) en kunnen tegenstrijdig zijn (guardrail blokkeert de ophoog-actie van de saldo-logica als de schaalgrens nadert).

**How to apply:**
- Deze automation is actief in ontwikkeling/tuning — verwacht wijzigingen en vraag bij twijfel naar de laatste staat op de server (`git pull`) voordat je ervan uitgaat dat een eerdere analyse nog klopt.
- Zie [[feedback_check_dependencies_before_rewrite]] — een eerdere herschrijving van dit bestand brak stilzwijgend beide doelen tegelijk door twee onderliggende sensoren te laten vallen; dat is inmiddels hersteld, maar dit bestand is duidelijk gevoelig voor dat soort regressies.
- De regeling reageert sinds 23/24-08-2026 ook direct (state-trigger + 15s debounce) op wijzigingen in `sensor.smartmeter_stroomverbruik`/`-stroomlevering`, niet alleen op de 2-minuten-heartbeat — nodig omdat grote verbruikers (wasmachine, droger, zwembadpomp) geen van allen in HA geïntegreerd zijn en dus niet los te herkennen zijn; de regeling reageert daarom alleen op het netto-effect op de meter.
- Live monitoren kan via `https://homeassistant.kakkenberg.com/dashboard-overview/heating` (SolarEdge Active Power Limit, Smart Meter netto Levering/Verbruik, Max. Productie Dag).
