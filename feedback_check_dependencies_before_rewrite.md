---
name: feedback-check-dependencies-before-rewrite
description: "Before rewriting/replacing a HA package file, grep the whole repo for every entity_id it currently defines to make sure nothing else still depends on it"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: a6a3dbf0-49f6-4611-ab8f-3bc5aa57105c
  modified: 2026-08-27T19:44:59.356Z
---

When rewriting or restructuring an existing Home Assistant package file (not just adding to it), grep the repo for every `sensor.`/`input_number.`/etc. entity_id the OLD version defines before removing it, to confirm nothing else (including the same file's own new logic) still references it.

**Why:** A rewrite of `packages/energie/zonnepanelen_zeroexport.yaml` (2.0.0, "Renewed zonnepanelen_zeroexport", 2026-08-27) dropped the `input_number:` startpoints and the `template: sensor:` chain that computed `sensor.jaarbalans_achterstand` and `sensor.smartmeter_levering_totaal`, without replacing them — even though the new automation's own header still listed both as dependencies. Because Jinja's `states(...) | float(0)` silently falls back to 0 for a missing entity instead of erroring, both of the file's headline features (saldo-inloop import target, and the new terugleverkosten-guardrail) silently became permanent no-ops: no error anywhere, the automation just kept running a plain zero-export loop. This was live on the production HA server for a period before being caught by manual review. See [[project_zonnepanelen_jaarbalans]].

**How to apply:**
- Before deleting/replacing any `input_number`, `template: sensor:`, or similar helper in a package file, `grep -r` the entity_id (and its `sensor.`/`input_number.` form) across the whole repo — not just the file being edited — to find every consumer.
- Remember that HA templates degrade silently (`states('sensor.x') | float(0)` → 0, not an error) when an entity disappears — a missing dependency will not surface as a loud failure, so this can't be caught by "does it still load" alone; it has to be checked structurally.
- When restoring/fixing such a regression, also check for other things quietly dropped in the same rewrite (e.g. a second automation in the same file) — a rewrite that drops one dependency chain may have dropped more than one thing.
