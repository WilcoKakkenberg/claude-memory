---
name: new-branch-for-local-changes
description: "All local changes in the HomeAssistant repo go in a new branch, never on master or an unrelated existing branch"
metadata:
  node_type: memory
  type: feedback
  originSessionId: f8684980-f7fb-45b9-a092-e7e9d95571ea
  modified: 2026-10-15T16:17:21.939Z
---

Make every set of local changes in a new branch, not directly on master and not on whatever feature branch happens to be checked out.

**Why:** master is what runs live on Home Assistant (homeassistant.kakkenberg.home) — anything on master goes to production. The user stated this rule on 2026-10-15 ("alle lokale wijzigingen in een nieuwe branch") after I edited files on the already checked-out `feature/emhass-melding-overdag-html` branch for an unrelated fix (frost warning / KNMI d2tmin).

Never push, merge or commit to master — the user does that themselves on HA.

The master on HA is always leading: HA pushes it to origin/master (the "auto: <datum> 03:00" and "<datum> - <maand> Push" commits). Treat origin/master as the source of truth; local master only follows it, and on any difference or conflict origin/master wins. Base new branches on a fresh origin/master.

**How to apply:** Before the first edit, create a new branch from up-to-date master (`git fetch` first) with a descriptive name (e.g. `fix/<onderwerp>` or `feature/<onderwerp>`). Production HA runs master; feature branches only reach HA after merge to master.
