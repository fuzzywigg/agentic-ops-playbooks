# Agentic Ops Playbooks

Operational doctrine from running a real, single-operator agent gateway in
production — model fleets, cron automation, exec policy, approval flows, and the
failure modes none of the tutorials mention.

**Everything here was learned the expensive way.** Each playbook documents defects
found live in a working deployment, verified, fixed, and re-verified. No
hypotheticals, no vendor pitch.

## Playbooks

| Doc | One-line thesis |
|---|---|
| [Controls That Lie](controls-that-lie.md) | The dominant failure mode of agent infrastructure is a control that reports a value it doesn't have — found 10+ times in one audit. |
| [Exit Codes for Unattended Jobs](exit-codes-for-unattended-jobs.md) | 0 = clean, 1 = findings, 2 = broken. Collapse the last two and you train yourself to ignore your own alerts. |
| [Model Lifecycle for Agent Fleets](model-lifecycle.md) | Capability is a gate, not a weight; free-model churn rots any static chain in ~24h — and a bi-weekly drift review keeps the registry itself honest. |
| [Exec Policy Shapes](exec-policy.md) | Reviewed automation beats both free-run and hard-deny — revised after independent external review (named-binary allowlists, accurate safeBins semantics, timeout-must-not-widen-authority). |

## The meta-point

The system that produced these docs drafts its own follow-ups on an overnight
"night shift" — queued briefs burned against otherwise-idle subscription compute,
with a morning digest. The docs about the machine are written by the machine,
reviewed by the human. That loop is the actual product.

## License

MIT. Use anything. Attribution appreciated.
