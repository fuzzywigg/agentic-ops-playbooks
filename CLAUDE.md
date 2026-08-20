# CLAUDE.md — Agentic Ops Playbooks

This repo is operated as a living reference. The gateway that generated it also drafts follow-up playbooks autonomously, which are reviewed and merged by the human owner. This file tells Claude how to work with the repo.

## What belongs here

Two document types are admitted. Every file names its type in its version line.

**Incident playbooks** meet all three criteria:

1. **Real incident, not hypothetical.** It documents a defect that was observed in production, verified, and fixed — not a class of defects that could exist.
2. **Reusable defect class.** The failure pattern is general enough to recur in other agentic deployments. Idiosyncratic one-offs don't generalize.
3. **Ends with a test.** Every countermeasure must be verifiable. A control that has never fired is a hope, not a control.

**Doctrine documents** — operating doctrine, architecture descriptions, or pre-mortems of the running system. Admitted only when:

1. The version line names the type explicitly (e.g. `*Doctrine version: YYYY-MM-DD. Architecture description; failure modes anticipated, not yet observed.*`).
2. The document never uses incident narrative framing for events that were not observed. A verified failure mode of a tool this repo depends on may be documented without a production incident, provided the version line says so and the one-line test proves the behavior.
3. It still ends with a runnable one-line test.

When an anticipated failure mode fires in production, promote it: add the specimen — a timestamp, a log line, a tool output — and update the version line.

## Voice and style

- Direct, no hedges. "This breaks" not "this may cause issues."
- Evidence-first. State what was observed before what it means.
- Tables for structured comparisons, code blocks for exact commands and config.
- No vendor pitch framing. No "this powerful new feature" language.
- End every document with a one-line test the reader can run immediately.

## File structure

Each document is a standalone Markdown file at the root level. Name new files after the failure domain or defect class, never the countermeasure (`controls-that-lie.md` or `exit-codes-for-unattended-jobs.md`, not `fix-controls.md`). Existing filenames are frozen — this repo is published and cross-linked, and renames break inbound links; apply the naming preference to new files only.

Update `README.md` when adding a document — the table there is the index, and each row carries the document's type.

## Version lines

Each document ends with a version line, and the version line is the last line of the file:

```
*Playbook version: YYYY-MM-DD. [Brief provenance note.]*
*Doctrine version: YYYY-MM-DD. [Type and provenance note.]*
```

Update it when the document changes materially. The date is the date of the last meaningful edit, not the date of the original incident.

## What not to do

- Don't add hypothetical examples to incident playbooks, or present anticipated failure modes as observed anywhere. Anticipated modes are permitted only in documents labeled doctrine, and only labeled as anticipated.
- Don't cross-reference external vendor documentation as the primary source of truth — it drifts. Reference the behavior, not the URL.
- Don't clean up prose for its own sake without adding new evidence or correcting an error.
