---
name: feedback-git-remote-leading
description: "Git workflow rule for the HomeAssistant repo - remote is always authoritative, local clone must never get ahead"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 0230a3a0-73fc-46d9-a6c8-56fda3f0df17
  modified: 2026-08-23T09:37:12.049Z
---

The remote repository (`origin/master` on GitHub) is always leading/authoritative for this Home Assistant config repo. The local clone this agent works in must never be ahead of the remote, and this agent must not introduce/deploy changes onto the live HA server or push them to GitHub itself.

This local checkout is the user's shared space for *developing* automation/config changes together with the agent (editing YAML, designing logic, reviewing diffs) - that collaborative editing is welcome and expected. But rolling those changes out is a separate, manual step the user reserves for themselves: they copy/introduce the change onto the HA server and check the result there, then push from the HA server to GitHub using their own scripts (e.g. `gitupdate.sh`). The agent should not shortcut that by committing/pushing from this clone.

**Why:** Confirmed twice in the same session - first after this agent auto-merged a divergent history locally (merge commits + a fix commit) and pushed directly to origin, the user clarified local should never lead; then the user further clarified that they specifically want to be the one who introduces changes onto the HA server and verifies them there, i.e. deployment is a user action, not an agent action.

**How to apply:**
- Editing files together in this checkout (discussing, writing, revising YAML) is fine and is the whole point of the directory.
- Prefer `git pull` / `git fetch` to stay in sync; avoid creating local commits unless the user explicitly asks for a commit.
- Never push, and never suggest/imply the agent will apply a change to the live HA server - that's the user's job. If a fix is needed, hand the diff/explanation to the user rather than committing+pushing it.
- If a merge conflict or a bad auto-merge is discovered (e.g. dangling trigger references, dropped YAML blocks), report the problem clearly and let the user decide how/where to resolve it.
- If local ever ends up ahead of `origin/master`, flag it and ask before pushing, rather than assuming it's fine to publish.
- Reconfirmed explicitly (2026-08-23): "push nooit wijzigingen" - never push, full stop, not even if asked to push "this time." Local commits on a branch are fine when the user asks for a commit; pushing is categorically off-limits unless/until the user changes this rule.
