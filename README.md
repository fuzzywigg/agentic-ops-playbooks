# Agentic Ops Playbooks

> Operational doctrine from a real, single-operator agent gateway in production.  
> Every defect documented here was found live, verified, and fixed. None are hypothetical.

---

The most dangerous thing in an agentic system is a control that believes it is working.

These playbooks were distilled from auditing and operating a production Claude Code gateway — model fleets, scheduled automation, exec policy, approval flows, and the failure modes the tutorials don't cover. Each one starts from a real incident, names the defect class precisely, and ends with a countermeasure that survived contact with reality.

The system that produced these docs runs its own follow-up drafts on a scheduled night shift — burning otherwise-idle subscription compute, delivering a morning digest for review. The docs about the machine are written by the machine, reviewed by the human. That loop is the actual product.

---

## Playbooks

| Playbook | The defect class |
|---|---|
| [Controls That Lie](controls-that-lie.md) | The dominant failure mode of agent infrastructure is a control that reports a value it doesn't have. Found 10+ times in one audit, independently, in components written months apart. |
| [Exit Codes for Unattended Jobs](exit-codes-for-unattended-jobs.md) | `0 = clean · 1 = findings · 2 = broken`. Collapse the last two and you train the operator to stop reading red rows. Real crashes become invisible. |
| [Model Lifecycle for Agent Fleets](model-lifecycle.md) | Capability is a gate, not a weight. Free-model churn rots any static chain in ~24h. A bi-weekly lineup drift review is the control that keeps the registry honest. |
| [Exec Policy Shapes](exec-policy.md) | Reviewed automation beats both free-run and hard-deny. Named-binary allowlists, accurate `safeBins` semantics, `strictInlineEval`, and an `askFallback` that never widens authority on timeout. |
| [Remote Control in Production](remote-control.md) | Remote Control can be silently killed by five independent env vars — each of which is routinely set in production gateway deployments. Knowing the kill list before you deploy is cheaper than diagnosing it after. |
| [The Night Shift Loop](night-shift.md) | A system that writes documentation about itself has specific failure modes: drafts that pass on style but fail on substance, a review backlog that goes unread, a context that cites its own unreviewed output as ground truth. |

---

## Why these exist

Most documentation describes what a system does when everything works. These playbooks describe what a system does when something is quietly wrong — and how to find out before a downstream job acts on a verdict it was never given.

The design principle behind all of them: **a control that has never fired is a hope, not a control.** Every playbook ends with a test that can be run.

---

## License

MIT. Use anything. Attribution appreciated.
