# Model Lifecycle for Agent Fleets

*A practical playbook for teams running multi-model cron fleets across free and metered API lanes.*

---

## The problem

When you run dozens of automated agent tasks daily against a pool of language models, two things happen that static configuration cannot handle: models vanish without notice, and the models that are available today are not the ones that will work tomorrow.

Free lanes on aggregator platforms can turn over within 24 hours. A model that passed last week's test may silently stop routing tools this week. The naive response is to hand-curate a fallback chain and update it when something breaks. This document argues for a different approach: a continuous discovery and validation pipeline that keeps the registry current, and a set of principles that prevent the pipeline itself from making things worse.

---

## Principle 1 — Capability is a gate, not a weight

This is the rule everything else follows from.

In a weighted scoring system, a model that is free, fast, and mostly-capable will outscore a model that is metered, slower, and fully capable. The problem is that "mostly" on tool routing is not a graceful degradation — it is a silent failure. A model that shoves shell commands into a messaging tool, or hallucinates tool names, does not produce lower-quality output; it produces garbage that can look like success until a downstream system acts on it.

The evidence for this is concrete. In a real production run, a free model was selected on cost grounds over a metered incumbent. The free model consumed roughly five times the tokens, ran sixty times slower, and produced no usable artifact. Any composite score with meaningful weight on price would have made the same choice.

**A model must pass eligibility before it is scored at all.** A model that cannot route tools correctly is not a low-scoring candidate for a tool-routing role — it is ineligible at any price. Cost, latency, and context length only rank survivors.

---

## Principle 2 — No single probe is a verdict

The instinct after a bad production result is to build a test that catches it and trust that test. This instinct is wrong for model evaluation because models on aggregated free lanes do not behave like stable software.

Consider a real example: three independent checks were run against the same model's tool-routing capability.

| Source | Result |
|---|---|
| Provider-reported metadata | supports tools |
| Native scanner probe | FAILED |
| Direct API battery, two attempts | PASS — correct tool call, correct arguments |

All three can be true simultaneously. Aggregators route requests to different backends across calls. Free lanes rate-limit unpredictably. A model that passes on the first call, fails on the second, and passes on the third is not misbehaving — it is telling you that one sample is insufficient to characterize it.

**Verdicts must be three-state: PASS / FAIL / INCONCLUSIVE.**

A transport error (HTTP 429, 502, 503, 504, timeout) is not a capability finding — it is a fact about load, not about the model. Treating rate-limit responses as FAIL would retire popular models precisely because they are popular. The prior verdict stands until real signal arrives.

A FAIL verdict requires that the model actually responded incorrectly, not that it failed to respond at all. A PASS requires consistency across multiple samples, not a single clean call. INCONCLUSIVE is an honest answer and a better one than a confident wrong verdict.

Provider-reported metadata (`supportsTools` flags and similar) is useful as a cheap pre-filter but is never used as the deciding signal for any gate. Metadata misreports its own capability often enough to make it unreliable as a sole source.

---

## Principle 3 — Never grant a role on absence of evidence

A related trap: inferring capability from the absence of a failure. If a model replies to a prompt without crashing, it has demonstrated that it can produce text. It has not demonstrated that it can route tools, produce valid structured output, respect state contracts, or operate correctly inside a multi-turn agent loop.

Granting roles based on reachability alone will admit music generation models and safety classifiers into candidate pools for scheduling crons. When a probe tests exactly two things — tool routing and image understanding — chat capability is unknown. The registry says "unknown," not "yes." A fabricated verdict is worse than a missing one because it leads to silent failures in production.

---

## The pipeline

```
discover → smoke → pressure → trial → promote
```

**Discover:** Daily, deterministic, zero token cost. Enumerate what exists on configured providers today. Flag new arrivals and record last-seen timestamps. This is the cheapest possible defense against building on infrastructure that disappears.

**Smoke:** A fast probe — does the model respond, and can it emit a correctly-formed tool call at all? This is a filter, not a verdict. Models that hard-fail here are INELIGIBLE; models that return transport errors are INCONCLUSIVE.

**Pressure:** A capability battery run against the kinds of tasks the fleet actually does. The checks that matter for cron work: tool routing under a realistic tool surface, state-gate compliance (honoring a "this work is done, stop" contract), and structured output validity. A model that fails any of these three is ineligible for cron work regardless of price or speed.

The battery runs against the provider API directly, not through the agent harness. This makes failures attributable to the model rather than to local wiring, and it prevents a probe from mutating live agent state. The cost: the battery cannot see failures that only appear inside the full agent loop.

**Trial:** Shadow-run a real task through the real agent harness, against a sandboxed workspace. This is the stage that catches what the pressure battery cannot see. The gap is real and large: a model that passes every isolated probe at every tool-surface size may still fail to complete a multi-turn cron task, because single-turn probes cannot observe how a model handles tool results, context accumulation, or the need to keep routing correctly after the first call.

**Promotion must never run off the battery alone.** The battery is a cheap filter. Only the trial can say whether a model actually works in-loop.

---

## Autonomy boundaries

Some operations are safe to run autonomously; others require explicit human ratification. The distinction is whether the failure mode is visible.

**Autonomous (no approval needed):**

- Discovery, registry maintenance, smoke and pressure probing, scoring, ranking
- Retiring a model that has 404'd for multiple consecutive scans — waiting for human ratification of a model that no longer exists is strictly worse than acting
- Demoting a model whose tool-routing status has regressed from passing to failing — this is a silent, dangerous transition; auto-shelve and report
- Reordering the fallback chain among already-eligible models — reversible, low blast radius

**Requires human ratification (and the gate fails closed):**

- Promoting a candidate to primary — every interactive session changes behavior at once
- Changing the model on a named scheduled job or agent — silent swaps make failures very hard to attribute later

Ratification works as follows: a one-time token is sent via an approval channel, the system polls for the reply, and the promotion proceeds only on an explicit `APPROVE <token>` response that post-dates the request. Silence is not consent. A notification failure is not consent. An old approval is not consent. Every other condition results in no change.

---

## Discovery jobs must not mutate the config they observe

This deserves its own statement because the failure mode is subtle.

A discovery or registry scan that modifies the configuration it is reading creates a feedback loop: the next scan sees its own previous writes, not ground truth. The job that notices a model has vanished should set an alert exit code and stop — it should not attempt to remove the model from the config, reorder fallback chains, or write any state that the next promotion decision will read.

Mutation belongs in promotion, which requires the full pipeline plus human ratification. Discovery is read-only.

*→ see also: [Controls That Lie](controls-that-lie.md) — the specimen "the discovery job that mutated what it observed" is this incident; snapshot-and-diff around anything that "only reads" is how you prove discovery stayed read-only.*

---

## Operational reality: free-model churn

Free lanes on aggregated providers change fast. Of 15 free models available at a given scan, at least one may be flagged for retirement within 24 hours. Of 18 tracked models in an early registry, only 13 demonstrated any capability at all, and two actively misreported their own.

A registry that tracks when each model was first seen and when it was last seen is the cheapest possible hedge against this. The scarce thing is not access to models — it is a current, evidence-backed answer to which models can actually do the work. Static configuration answers this once. A live registry answers it every day.

---

## Summary

| Principle | Wrong | Right |
|---|---|---|
| Scoring | Weighted composite including price | Gate first, then score |
| Probing | Trust any single measurement | Three-state verdict: PASS/FAIL/INCONCLUSIVE |
| Role assignment | Reachability implies capability | Only probed capabilities are asserted |
| Promotion | Battery pass is sufficient | Battery + trial + human ratification |
| Discovery | Read and write | Read-only; mutation is a separate stage |
| Transport errors | Mark as FAIL | Mark as INCONCLUSIVE |

The goal is not a large model pool. It is a current, tested, honest answer to which models can do the specific work in the fleet — and a system that maintains that answer automatically as the landscape changes.

---

## The cadence that keeps it honest (owner review addendum)

A registry that runs but is never read reverts to being a story. The failure mode isn't
the scan breaking — it's *drift going unexamined*: models quietly stop being freshened
into the lineup and nobody asks why. So the operating doctrine includes a **bi-weekly
lineup drift review** — a scheduled, delivered report answering three questions:

1. What entered or left the eligible set since last review, and why?
2. Is anything in a live chain that the evidence no longer supports?
3. Has the *rate* of change shifted — churn accelerating or, more suspiciously, stopping?

A registry whose eligible set hasn't changed in a month is more likely broken than
stable. The review exists to catch exactly that.

---

## The one-line test

Pull the registry's last-seen timestamp for every model in the live fallback chain.
Any timestamp older than one discovery-scan interval means discovery is dead and the
chain is running on stale evidence. An eligible set unchanged for a month is more
likely a broken pipeline than a stable landscape.

---

*Playbook version: 2026-08-19. Based on production evidence collected 2026-07-26 through 2026-07-28; owner-review addendum (bi-weekly drift review) folded above the version line and closing test added 2026-08-19.*
