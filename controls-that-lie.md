# Controls That Report a Value They Don't Have

*Field notes from auditing a production agent gateway. Every example below was found
live, verified, and fixed — none are hypothetical.*

## The defect class

An agentic system accumulates controls: allowlists, gates, probes, state markers,
health checks. The most dangerous failure mode isn't a missing control — it's a
control that **reports a value it doesn't have**. The dashboard says gated; nothing
is gated. The config says restricted; the restriction is never consulted. Each
instance is small. Together they train the operator to trust a fiction.

In one two-week audit of a single-owner agent deployment, this exact class appeared
**more than ten times**, independently, in components written months apart. It is not
a bug; it is an attractor.

## Specimens

**The exec policy that didn't apply to the main agent.** The gateway enforced a
binary allowlist with approval escalation — for subagents and cron jobs. The default
agent's shell tool was a separate code path with no approval logic at all, and it
silently ignored the deny-list too. Every audit of "what can the agent execute"
read the policy that governed the *delegated* paths and concluded the system was
locked down. The main path had no lock. (Resolution: the fast path was kept — but
as a *documented owner decision* with its consequences written down, instead of an
invisible hole.)

**Two config files, and the reassuring one was wrong.** Requested policy lived in
the main config (`allowlist`); the host approvals file said `full`. The merge took
the stricter value, so enforcement was correct — but a human auditing the approvals
file alone concluded exec was unrestricted. Two sources of truth disagreed, and
which one lied depended on which file you opened first.

**The discovery job that mutated what it observed.** A daily model-registry scan
called the platform's `models scan` — which, it turned out, *rewrote the production
fallback chain* as a side effect. The scan's own probe data showed a model failing
tool-routing on the same morning the scan wrote that model into the live chain.
A monitoring job was the system's biggest unaudited writer. (Fix: snapshot the
config before the scan, restore byte-for-byte after, alert if they differ.)

**Markers that said "done" days after the work stopped.** Daily jobs were gated on
state markers — but sandboxed job runtimes couldn't write to the marker path, so
markers silently froze while jobs kept reporting green. Gating logic read stale
markers and made real scheduling decisions on them. (Fix: move marker-writing into
a shell wrapper that runs the agent turn — the component that *can* write owns the
write.)

**The checker blind to exactly where breakage lands.** A source-health check
scanned only files tracked by git. Freshly generated, not-yet-committed files —
the place new corruption actually appears — were invisible. A skill landed with
two of three modules unparseable and the checker reported the repo healthy.

**"Findings" indistinguishable from "crashed."** Audit scripts exited 1 when they
found something — by design. The scheduler treated any nonzero exit as failure. So
"the audit worked and found problems" and "the audit crashed" produced the identical
red row, and both trained the operator to stop reading red rows. (Fix below — this
one deserves its own convention.)

**The gate script with a quoting bug.** A pre-run gate parsed its state file with
an unquoted `jq` filter that the shell split into three arguments. The gate returned
"run" unconditionally, forever. It had also never been wired to its job. A control
can be dead twice at once.

**Success that only meant "the model replied."** A subagent's outcome record read
`{"status":"ok"}` for a run whose only command was denied by policy. Status meant
"a response was produced," not "the task was done." Anything consuming that status
inherited the lie.

## Why this class recurs

1. **Enforcement and configuration live in different components.** The config is
   what audits read; enforcement is what runtime does. Nothing forces them to agree.
2. **Controls are written for the path the author was using.** The exec gate covered
   the paths the author tested (subagents), not the path most traffic used (main).
3. **Side effects hide in verbs that sound read-only.** `scan`, `list`, `check` —
   one of these rewrote production config.
4. **Nobody tests the control, only the feature.** Features fail loudly. Controls
   fail silently, because their failure mode *is* silence.

## The countermeasures that actually worked

- **Verify the enforcement, not the setting.** After any policy change, attempt the
  forbidden thing and require the denial. A control that has never fired is a hope.
- **Snapshot-and-diff around anything that "only reads."** If a job shouldn't write
  config, prove it: hash before, hash after, alert on drift.
- **Three-state exits, everywhere: 0 = clean, 1 = findings, 2 = harness failure.**
  Then map exit 1 to *success-with-report* at the scheduler boundary. Red must mean
  broken, or red means nothing.
- **The writer owns the marker.** Whatever component can reliably write state is the
  component that must write it. Never let a sandboxed runtime "promise" a write.
- **Document the deliberate holes.** A fast path with no gate can be a legitimate
  choice — as an owner decision with consequences written down. The crime is the
  undocumented hole wearing a gate costume.
- **When two sources of truth exist, make one generated from the other** — or add a
  check that fails loudly when they disagree.

## The one-line test

For every control in your system, ask: *if this stopped working tonight, what would
tell me?* If the answer is "nothing," you don't have a control. You have a story.

*→ see also: [Remote Control in Production](remote-control.md) — a concrete instance of this class where the kill comes from an env var set for unrelated reasons, with no visible failure signal.*

---

*Playbook version: 2026-08-19. Based on a two-week audit of a single-owner agent deployment, 10+ instances of this defect class found independently.*
