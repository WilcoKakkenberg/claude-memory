---
name: feedback-git-commit-push-permission
description: Commit/push to any git repo (not just HomeAssistant) requires explicit per-instance user permission, every time
metadata:
  type: feedback
  originSessionId: 196e6bb1-90c5-4c0f-8017-4aae7c626445
  modified: 2026-08-29T21:32:04.190Z
---

Never commit or push to a git repository unless the user explicitly grants
permission for that specific instance. This applies broadly, not just to
the HomeAssistant repo (see [[feedback_git_remote_leading]] for that
repo's stricter "never push at all" rule).

**Why:** After approving a commit+push to the MkDocs docs repo
(`C:\Users\Wilco\Documents\github\MkDocs`) with "ja commit en push", the
user immediately added "dit mag alleen als ik toestemming geef" - making
explicit that this was a per-instance approval, not a standing
authorization to commit/push that repo going forward without asking
again.

**How to apply:**
- Treat every commit/push as requiring a fresh "yes" in the current
  conversation, even for a repo already committed/pushed to earlier in
  the same session.
- Editing/staging files, showing diffs, and drafting commit messages is
  fine to do proactively - only the actual `git commit`/`git push`
  execution needs the explicit go-ahead.
- If asked to "keep documentation updated" as an ongoing task, still stop
  and ask before each commit/push rather than treating one earlier
  approval as blanket permission for future ones.
