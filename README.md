# Agentic Ops Playbooks

Operational playbooks and doctrine from one production agent gateway (OpenClaw) and the Claude Code deployment beside it.

This repository is a **living index**, not a product. Every incident playbook documents a defect observed live, verified, and fixed. Doctrine documents state operating rules or pre-mortems for the same system — and say so in their version line. Nothing here is hypothetical filler.

Admission rules and editing voice live in [`CLAUDE.md`](CLAUDE.md). New proposals use the [playbook issue template](.github/ISSUE_TEMPLATE/playbook.yml).

---

The most dangerous thing in an agentic system is a control that believes it is working.

The incident playbooks each start from a real incident, name the defect class precisely, and end with a countermeasure that survived contact with reality. The doctrine documents state the operating rules and pre-mortems for the same system — and say so.

The system that produced these docs runs its own follow-up drafts on a scheduled night shift — burning otherwise-idle subscription compute, delivering a morning digest for review. The docs about the machine are written by the machine, reviewed by the human — that loop is what keeps them honest.

---

## The documents

| Document | Type | The defect class |
|---|---|---|
| [Controls That Lie](controls-that-lie.md) | incident | The dominant failure mode of agent infrastructure is a control that reports a value it doesn't have. Found 10+ times in one audit, independently, in components written months apart. |
| [Exit Codes for Unattended Jobs](exit-codes-for-unattended-jobs.md) | incident | `0 = clean · 1 = findings · 2 = broken`. Collapse the last two and you train the operator to stop reading red rows. Real crashes become invisible. |
| [Model Lifecycle for Agent Fleets](model-lifecycle.md) | incident | Capability is a gate, not a weight. Free-model churn rots any static chain in ~24h. A bi-weekly lineup drift review is the control that keeps the registry honest. |
| [Exec Policy Shapes](exec-policy.md) | doctrine | Reviewed automation beats both free-run and hard-deny. Named-binary allowlists, accurate `safeBins` semantics, `strictInlineEval`, and an `askFallback` that never widens authority on timeout. Grounded in the exec-bypass specimens from Controls That Lie. |
| [Remote Control in Production](remote-control.md) | incident | Five env vars — set for CI hygiene, privacy, testing, or gateway routing — disable Remote Control. Observed live: a session absent from the list, killed by a forgotten shell-profile variable. The defect class is a failure signal with no consumer. |
| [The Night Shift Loop](night-shift.md) | doctrine | A system that writes documentation about itself has specific anticipated failure modes: drafts that pass on style but fail on substance, a review backlog that goes unread, a context that cites its own unreviewed output as ground truth. |

---

## Why these exist

Most documentation describes what a system does when everything works. These playbooks describe what a system does when something is quietly wrong — and how to find out before a downstream job acts on a verdict it was never given.

The design principle behind all of them: **a control that has never fired is a hope, not a control.** Every document ends with a test that can be run.

---

## Checks

Pull requests run a thin docs workflow (`.github/workflows/docs.yml`) that verifies relative markdown links offline and asserts the known playbook markdown files exist and are non-empty. No app build — this repo is Markdown only.

Cloud Agents use `.cursor/environment.json` (no dependency install).

---

## License

MIT. Use anything. Attribution appreciated.
