# Exec Policy Shapes for Agent Gateways

*How to configure command execution policy so that automation is genuinely reviewed and unattended jobs degrade gracefully.*

---

## The decision space

When an AI agent can run shell commands, you have three broad options:

1. **Full freerun** — anything runs, no review. Fast, simple, and dangerous at scale.
2. **Hard deny** — nothing runs without explicit per-call approval. Safe, but jobs waiting on approval do not hang indefinitely — approvals can be async, expire, or terminate as denied.
3. **Reviewed automation** — an allowlist of trusted binaries runs freely; misses go through a review pipeline before reaching a human; unattended jobs have a defined fallback when the human is unavailable.

This document argues for the third option and explains the specific levers that make it work.

---

## The four-stage approval flow

A production exec policy for an agent gateway has four stages, each answering a different question. These stack with additional controls — tool policy, sandbox boundaries and workspace access, gateway-versus-node host selection, elevated mode, per-agent policy, host-local enforcement, and channel authorization including approver identity. The four stages describe what happens to a single command request:

**Stage 1: The allowlist.** What can run without any review? The answer should be: named binaries for installed, trusted tooling — not directory globs. `~/.nvm/versions/node/v24.0.0/bin/*` implicitly trusts every current and future executable in a user-controlled directory, including package-manager shims and any compromised install script that drops a binary there. Prefer explicit named entries:

- `~/.nvm/versions/node/v24.0.0/bin/node`
- `~/.nvm/versions/node/v24.0.0/bin/npm`
- `~/.local/bin/rg`

For high-capability tools where the binary is trusted but argument forms matter, add `argPattern` constraints alongside the path entry. A broad path-only entry trusts every argument form the binary accepts.

Anything not matched goes to Stage 2.

**Stage 2: Model auto-review.** Before a missed command reaches a human, a reviewer model inspects it. Use a reliable, low-latency reviewer with isolated quota. The key security property is not the subscription lane — it is that auto-review decisions are single-use and bound to an enforceable execution plan.

Heredocs, shell expansions, and wrapper quoting that cannot be reduced to a verifiable plan are not approved by the reviewer; they fall to Stage 3. Treat reviewer timeout, provider failure, or malformed output as **escalation**, never as approval. The reviewer is a filter, not a rubber stamp.

**Stage 3: Human approval channel.** Unresolved misses — commands that auto-review cannot approve — are forwarded to an approval channel. The human sees the exact command, approves or denies, and the decision is logged. Note: an approved script form binds a file snapshot; if the file drifts before execution, the binding detects it and denies.

**Stage 4: Fallback on timeout.** The approval channel may not have a human watching. This is `askFallback` and it is the most consequential setting in the whole policy. Three options:

- `askFallback: full` — on timeout, run anyway. The approval gate becomes advisory. Never use this for unattended work.
- `askFallback: deny` — on timeout, block. The job terminates with an observable denial.
- `askFallback: allowlist` — on timeout, re-evaluate against the allowlist. A command that missed the allowlist originally will normally be **denied** by the fallback. Timeout is terminal for novel commands.

**The right default for unattended work is `allowlist` or `deny`.** On timeout the gateway must not widen authority. `allowlist` re-evaluates deterministic trust; `deny` terminates cleanly. Cron must treat a denial as an expected, observable failure — not as a condition to retry interactively. Design for it: preflight cron commands against the effective policy before deploying; alert on approval-pending, timeout, and denial events; use dedicated cron agents with minimal allowlists and fixed wrapper scripts rather than broad interpreter rules.

---

## safeBins: what it is and what it is not

Most gateway frameworks include a built-in vetted binary list called `safeBins` — a constrained stdin-only fast path with argument validation, denied flags, literal-token handling, and trusted-directory resolution.

In recent OpenClaw versions the defaults are: `cut`, `uniq`, `head`, `tail`, `tr`, `wc`. `grep` and `sort` are opt-in and restricted. `jq`, `awk`, and `sed` are denied — they cannot be reliably constrained to stdin-only transformation and are too programmable.

`safeBins` is not a general binary trust list. It is never the right place for interpreters, runtimes, package managers, or programmable tools. A binary in `safeBins` runs on the fast path; it does not pass through the allowlist review chain.

---

## strictInlineEval: the interpreter and command-carrier problem

Allowlisting `python3` or `node` solves one problem and creates another. If the interpreter binary is on the allowlist, then `python3 somefile.py` runs freely — but so does `python3 -c "import os; os.system('...')"`. The allowlist cannot distinguish them.

`strictInlineEval: true` closes this gap. Coverage extends to:

- Inline execution flags (`-c`, `-e`) on interpreter binaries
- Command carriers that embed code in argument strings: `awk`, `sed`, `make`, `find -exec`, `xargs`

When a command cannot be reduced to an enforceable plan — because of unsupported heredocs, expansions, or carrier forms — it falls to a human rather than being auto-approved or auto-denied.

Always enable `strictInlineEval`. The cost is a small number of additional review events for legitimate one-liners.

---

## Interpreters do not belong in safeBins

Interpreters (`bash`, `sh`, `python3`, `node`) are categorically different from data-transformation tools. They can turn any string passed to them into arbitrary execution. Placing them in `safeBins` bypasses the allowlist review chain entirely. OpenClaw warns on unprofiled interpreter entries in `safeBins` — that warning is correct and should be acted on.

The right treatment: place interpreters in the allowlist with strict path entries and `argPattern` constraints where needed. The allowlist still applies `strictInlineEval`; `safeBins` does not.

---

## Ungated fast paths: document, do not deny

Every production deployment has paths where review is deliberately skipped. The goal is not to eliminate these but to name them explicitly so audits and incident response are accurate.

In some deployments a primary interactive shell — `bash` invoked via a dedicated shell tool — runs outside the exec approval chain entirely. **This is deployment-specific behavior, not universal OpenClaw behavior.** The exec policy as described above applies to subagents, cron runners, and remote nodes; whether it applies to the primary session's direct shell depends on how the deployment is wired.

Actual bypass classes to document explicitly:

- Effective mode `full` (skips approval stages for that scope)
- Authorized session overrides granted by the operator
- Elevated mode when both requested and host policy permits
- Tool paths that use dedicated tools rather than the exec mechanism
- External plugins with separate authorization contracts

Distinguish "shell binary routed through exec" from "dedicated shell tool with its own authorization." A security control that appears to exist but does not is more dangerous than an acknowledged gap.

---

## Inspecting the effective policy

Do not rely on hand-editing configuration files or counting which files exist. The effective policy is the **merged result** of gateway config, host approvals, and any per-agent overrides — and in recent versions, approvals live in the shared state database, not in a standalone file. Inspect via CLI:

```
openclaw approvals get --gateway
openclaw exec-policy show
```

When multiple policy sources conflict, the stricter setting wins. An agent scope absent from `exec-policy show` output is not necessarily misconfigured — verify whether the inherited result matches the expected effective policy. Treat an omitted scope as a finding only when the CLI cannot demonstrate the expected inherited result. Node-managed policies are inspected at the node.

`mode: auto` expands to `{security: allowlist, ask: on-miss, autoReview: true}`. Do not set `mode` alongside explicit `security`/`ask` fields — the schema rejects the combination.

---

## Validation matrix

Run these checks after any exec-policy modification:

| Test | Expected result |
|------|----------------|
| Allowlisted low-risk binary | Runs without prompt |
| Non-allowlisted binary | Review / prompt / deny |
| Interpreter with script file via intended path rule | Runs only if path and arg pattern match |
| Interpreter with `-c`/`-e` | Goes to reviewer or requires explicit approval |
| Safe-bin tool with stdin input | Runs on fast path |
| Safe-bin tool with file operand | Fails closed (file operands not in stdin-only contract) |
| Pipeline with one untrusted segment | Entire command needs approval |
| Approval timeout | Per `askFallback` — allowlist re-evaluates, deny terminates |
| Reviewer outage or malformed response | Escalates to human; never auto-approves |
| Approved script file modified before execution | Binding detects drift, denies |
| Cron command denied | Terminates observably; alert fires |

Every pipeline segment must satisfy allowlist or safe-bin rules. Shell redirections are not supported in allowlist mode. Approved script forms bind a file snapshot, not a path.

---

## What exec policy does not cover

The allowlist gates *binaries*, not *intent*. `git push --force` and `git status` are indistinguishable at the allowlist layer. Destructive-intent gating belongs in the review and policy layers above.

Human gating for high-stakes actions — credential entry, financial transactions, production database migrations, domain transfers, git history rewrites — is a separate design concern that exec policy cannot substitute for.

---

## The policy in one sentence

Name trusted binaries explicitly with `argPattern` where arguments matter, keep interpreters out of `safeBins`, auto-review misses with a reliable reviewer where failure always escalates, forward unresolved cases to a human, fall back to allowlist-or-deny (never full-run) on timeout, enable `strictInlineEval` to cover command carriers, inspect the effective merged policy via CLI rather than counting files, and document every deliberate bypass rather than representing it as gated.

---

*Playbook version: v2, 2026-08-03. Incorporates external review corrections applied 2026-08-02.*
