# CLAUDE.md — Agentic Ops Playbooks

This repo is operated as a living reference. The gateway that generated it also drafts follow-up playbooks autonomously, which are reviewed and merged by the human owner. This file tells Claude how to work with the repo.

## What belongs here

A playbook belongs here when it meets all three criteria:

1. **Real incident, not hypothetical.** It documents a defect that was observed in production, verified, and fixed — not a class of defects that could exist.
2. **Reusable defect class.** The failure pattern is general enough to recur in other agentic deployments. Idiosyncratic one-offs don't generalize.
3. **Ends with a test.** Every countermeasure must be verifiable. A control that has never fired is a hope, not a control.

## Voice and style

- Direct, no hedges. "This breaks" not "this may cause issues."
- Evidence-first. State what was observed before what it means.
- Tables for structured comparisons, code blocks for exact commands and config.
- No vendor pitch framing. No "this powerful new feature" language.
- End every playbook with a one-line test the reader can run immediately.

## File structure

Each playbook is a standalone Markdown file at the root level. Name files after the defect class, not the solution (`controls-that-lie.md` not `fix-controls.md`).

Update `README.md` when adding a playbook — the table there is the index.

## Version lines

Each playbook ends with a version line:

```
*Playbook version: YYYY-MM-DD. [Brief provenance note.]*
```

Update it when the playbook changes materially. The date is the date of the last meaningful edit, not the date of the original incident.

## What not to do

- Don't add hypothetical examples or "common patterns" that weren't observed in production.
- Don't cross-reference external vendor documentation as the primary source of truth — it drifts. Reference the behavior, not the URL.
- Don't clean up prose for its own sake without adding new evidence or correcting an error.
