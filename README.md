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
| Model Lifecycle for Agent Fleets | *(coming — drafted nightly by the system these docs describe)* Capability is a gate, not a weight; free-model churn rots any static chain in ~24h. |
| Exec Policy Shapes | *(coming)* Reviewed automation beats both free-run and hard-deny; where interpreters belong and where they never do. |

## The meta-point

The system that produced these docs drafts its own follow-ups on an overnight
"night shift" — queued briefs burned against otherwise-idle subscription compute,
with a morning digest. The docs about the machine are written by the machine,
reviewed by the human. That loop is the actual product.

## License

MIT. Use anything. Attribution appreciated.
