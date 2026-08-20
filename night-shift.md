# The Night Shift Loop

*Architecture and failure modes of a system that writes documentation about itself.*

---

## The setup

The gateway that produced this repo also drafts its own follow-up playbooks. The pattern: otherwise-idle subscription compute runs a scheduled agent turn overnight, burns against a primed context of recent production logs and prior playbooks, produces a draft, and delivers a morning digest for human review. Drafts that pass review get merged. The docs about the machine are written by the machine, reviewed by the human.

This is not a novel idea — but the failure modes are specific, and most of them are invisible until the loop has been running long enough to drift.

---

## The architecture

```
cron trigger
  → agent turn (primed context: recent logs + existing playbooks + style guide)
    → draft playbook or revision
      → delivery channel (morning digest)
        → human review
          → merge or discard
```

Each component has a failure mode. The dangerous ones are the ones that look like success.

---

## Failure modes

**The draft that passes on style but fails on substance.**  
The loop has read the existing playbooks long enough to reproduce their voice. It knows the structure: defect class, specimens, countermeasures, one-line test. It can produce a document that looks like a playbook — and documents a hypothetical rather than a real incident. The style gate passes. The substance gate doesn't exist.

*Countermeasure:* The review protocol must include a question that cannot be answered by style alone: *what was the exact symptom that was observed?* A draft that cannot name a specific observation — a timestamp, a tool output, a dashboard reading, a failed command — is rejected regardless of how well it's written.

**The morning digest that goes unread.**  
If the digest arrives at a predictable time and nothing in it has ever been urgent, the operator learns to handle it later, then later becomes never. The loop keeps running. The drafts accumulate. The feedback signal from human review disappears. Weeks later, the loop is producing documents that have drifted from the real operating conditions because nobody told it they'd changed.

*Countermeasure:* The digest must report the rate of unreviewed drafts. If more than N drafts are queued without review, the loop pauses and sends a different message: *the review backlog has grown — the loop cannot stay honest without the feedback.* The loop knows it cannot self-verify.

**The context that references its own previous drafts as ground truth.**  
When the agent turn is primed with existing playbooks, it treats them as evidence of what the system does. If a draft was wrong and got merged without correction, the next turn builds on the error. Playbooks cite each other. The error propagates.

*Countermeasure:* The primed context distinguishes between *merged and human-reviewed* playbooks and *recent machine drafts*. Only the former count as evidence. The loop does not cite its own unreviewed output.

**The loop that rewrites instead of discovering.**  
A loop that runs long enough against a stable corpus starts producing revisions rather than new observations — rephrasing existing playbooks, reorganizing sections, adding examples. This is not useful, but it looks like output. It also slowly degrades the originals.

*Countermeasure:* Revisions to existing playbooks require an explicit flag in the draft header and a stated reason that references new evidence. A revision without new evidence is discarded at the delivery stage, before it reaches the morning digest.

**The cron that silently stops.**  
The scheduled turn fails — auth token expired, quota exhausted, harness error — and emits nothing. No draft, no error, no digest. The operator assumes the loop is running because nothing has broken visibly. This is the [exit-codes](exit-codes-for-unattended-jobs.md) defect class: "no output" and "ran successfully with nothing to report" are indistinguishable.

*Countermeasure:* The loop has a heartbeat separate from its output. Even if no draft is produced, the morning digest includes a line: *loop ran, no draft generated, exit 0.* Silence is not a valid delivery. A loop that doesn't check in is considered stopped.

---

## The review protocol

Human review is not editorial review. The question is not "is this well-written?" The questions are:

1. **Did this actually happen?** Can I point to a log, a tool output, a conversation, a dashboard reading that supports the specific observation the draft makes?
2. **Is the defect class real and reusable?** Would I recognize this failure in a different deployment?
3. **Is the countermeasure testable?** Can I run the one-line test right now and get a meaningful result?
4. **Does anything in this draft contradict known current operating conditions?** (The loop's context may be days stale.)

If any answer is "no" or "I'm not sure," the draft is discarded. The cost of a bad playbook is higher than the cost of a missing one — a bad playbook trains the operator to trust a fiction.

---

## What the loop cannot do

The loop cannot discover failures it hasn't been told about. It works from logs, tool outputs, and prior playbooks. If a failure leaves no log trace and was never surfaced in conversation, the loop cannot produce a playbook for it. The human review process is also the primary discovery channel — the reviewer is expected to bring new incidents to the loop, not just validate drafts.

The loop also cannot self-correct a drifted defect class. If the system changes and a playbook becomes outdated, the loop will continue citing it as evidence until a human marks it outdated. The bi-weekly drift review that keeps the model registry honest ([model-lifecycle.md](model-lifecycle.md)) applies here too: a corpus that hasn't been challenged recently is more likely drifted than stable.

---

## The one-line test

Check the delivery timestamp of the last morning digest. If it's more than 48 hours ago, the loop is stopped or the delivery channel is broken. Investigate before assuming it ran.

---

*Playbook version: 2026-08-19. Architecture description of the operating system for this repo. Updated as the loop itself evolves.*
