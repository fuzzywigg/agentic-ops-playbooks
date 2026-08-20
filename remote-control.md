# Remote Control in Production Agent Gateways

*Pre-mortem. How a Remote Control session dies in an unattended deployment: the error goes to a terminal nobody watches, and the only remotely visible symptom is an absence. No production incident is claimed — every behavior below is documented Claude Code behavior, confirmable by the one-line test.*

---

## The defect class

Five environment variables disable Remote Control. None of them exists for session management — they serve CI hygiene, privacy posture, testing, and gateway routing — so nothing about setting one looks related to Remote Control.

The failure is not silent at the source. Claude Code emits a named error: `claude remote-control` checks eligibility and errors out at startup rather than serving a dead session, and an interactive session shows a failure notification shortly after launch. Since v2.1.154 the message — "Remote Control requires feature-flag evaluation" — names the exact variable Claude Code found; before that it was the generic "Remote Control is not yet enabled for your account." `claude doctor` shows which individual eligibility check failed.

The defect class is **a failure signal with no consumer**. In an unattended deployment the error lands on stderr of a host nobody is watching, the launcher swallows it or restarts, and the only symptom visible from anywhere else is an absence: the session never appears at claude.ai/code. The signal existed; the pipeline had no consumer for it. This is a cousin of [Controls That Lie](controls-that-lie.md) — not a control reporting a value it doesn't have, but a correct report delivered where nobody reads.

---

## The kill list

| Variable | Why you have it | What it kills | Provenance |
|---|---|---|---|
| `DISABLE_TELEMETRY` | CI hygiene, privacy posture | Feature-flag evaluation | documented behavior |
| `DO_NOT_TRACK` | Same | Feature-flag evaluation | documented behavior |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Airgapped or metered environments | Feature-flag evaluation | documented behavior |
| `DISABLE_GROWTHBOOK` | Testing or stability | Feature-flag evaluation | documented behavior |
| `ANTHROPIC_BASE_URL` | LLM gateway or proxy | Remote Control itself (v2.1.196+) | documented behavior |

None of these rows comes from an incident in this deployment; each is documented Claude Code behavior. Confirm any of them with the one-line test.

One more location the table can't show: these variables also kill Remote Control when set in the `env` block of any `settings.json` file — managed, user (`~/.claude/settings.json`), project, or local — where a shell `env | grep` will never see them.

Fast first pass before deploying:

```sh
env | grep -E 'DISABLE|BASE_URL|DO_NOT_TRACK|GROWTHBOOK'
```

Then run the one-line test at the bottom of this playbook. A configuration that has never produced a visible session is a story, not a configuration.

---

## LLM gateway deployments

If your deployment routes Claude Code traffic through an intermediary (`ANTHROPIC_BASE_URL` set to a host other than `api.anthropic.com`), Remote Control is unavailable on that host as of v2.1.196. Remote Control pairs the local session with claude.ai through Anthropic's own backend; with a non-Anthropic base URL there is no backend to pair with.

Options, stated plainly:

- **Unset `ANTHROPIC_BASE_URL` for sessions where Remote Control matters.** The session uses direct Anthropic access; your gateway does not see that traffic.
- **Accept the split.** Cron agents and gateway-routed sessions: no Remote Control. Interactive sessions: unset the variable, use direct access.
- **Document the split explicitly.** The dangerous version is an implicit split nobody has written down — someone assumes Remote Control is available, it isn't, and they discover it during an incident rather than during setup.

---

## The mechanism

The session runs on the host — filesystem, MCP servers, tools, and project configuration stay local, and execution never moves to the cloud. `claude.ai/code` and the Claude app are a window into that session, paired through `api.anthropic.com` and gated on feature-flag evaluation — which is exactly what the kill list severs. (The session transcript — messages, responses, tool activity — is stored on Anthropic servers while connected; only execution and filesystem access provably stay local.)

Three ways in:

```bash
# Server mode — stays running, serves one or more remote sessions
claude remote-control

# Interactive mode — full terminal session, also reachable remotely
claude --remote-control

# From inside an existing session
/remote-control
```

Server mode for headless fleets; interactive mode for handing a desk session to your phone mid-task.

The rest of this section is behavior reference for the feature the kill list disables — kept per this repo's "reference the behavior, not the URL" rule, not as incident evidence.

### Server mode flags

```bash
claude remote-control --name "Night Shift"   # visible session title
claude remote-control --spawn worktree       # isolated git worktree per session
claude remote-control --spawn session        # single session, reject additional
claude remote-control --capacity 8           # cap concurrent sessions (default: 32)
```

`--spawn worktree` is the safest shape for unattended work — sessions that could conflict on the same files get isolated git worktrees automatically.

One caveat the docs attach that matters for exactly this use case: during a network outage, server mode gives up after roughly 10 minutes and the process exits, while interactive mode retries for as long as the outage lasts. An unattended server-mode deployment therefore needs a process supervisor (systemd unit, restart loop) — and the ~4-hour resume window below bounds how long a dead server can sit before its sessions are unrecoverable.

### Resume after restart

Sessions survive a Ctrl+C restart for about four hours — provided no other `claude remote-control` was running in the same directory and the server wasn't started with `--no-create-session-in-dir` (which archives its sessions on stop). Within that window:

```bash
claude remote-control                     # bring back every session the server was serving
claude remote-control --continue          # bring back the session the last server in this directory started with; exits when it ends
claude remote-control --session-id <id>   # bring back one session by ID; exits when it ends
```

Both `--continue` and `--session-id` require Claude Code v2.1.200 or later; earlier versions reject them as unknown arguments. The session ID is the segment of the `claude.ai/code` URL between `/code/` and any `?`.

---

## Approvals from your phone

What the docs support: Claude Code's own permission prompts can be answered from a connected device, and after several permission prompts in a session an **Approve tool calls from your phone** notification shows the session URL. That covers Claude Code's tool calls — not your gateway's exec policy.

An agent gateway's human approval channel (Stage 3 in [exec-policy.md](exec-policy.md)) is a separate system with its own transport. If a deployment chooses to wire that channel through a Remote Control session, that is deployment-specific wiring, not a built-in integration — and it inherits the kill list: any of the five variables takes the phone path down with it. Do not design an approval flow around an integration this page cannot promise.

---

## Workspace trust

Run `claude` in your project directory at least once and accept the trust dialog before starting a Remote Control session. Claude Code's startup trust dialog never saves trust for the home directory — starting `claude remote-control` from `~` fails the trust check. Always start from a project directory.

---

## Auto-connect

```json
// ~/.claude/settings.json
{ "remoteControlAtStartup": true }
```

Or via `/config` → **Enable Remote Control for all sessions**.

Project-level `.claude/settings.json` can disable auto-connect for a specific repo (set `false`) but cannot enable it — a checked-in `true` is ignored, so a repo can't silently enroll everyone who opens it.

---

## Requirements before deploying

- [ ] Subscription is Pro, Max, Team, or Enterprise (API keys not supported)
- [ ] Authenticated via `/login`, not API key
- [ ] Not on Bedrock, GCP Agent Platform, or Microsoft Foundry
- [ ] `ANTHROPIC_BASE_URL` unset or pointing to `api.anthropic.com`
- [ ] `DISABLE_TELEMETRY`, `DO_NOT_TRACK`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DISABLE_GROWTHBOOK` all unset — in the shell **and** in every `settings.json` `env` block
- [ ] Workspace trust accepted from a project directory (not home)
- [ ] On Team/Enterprise: Owner has enabled Remote Control in admin settings
- [ ] Server mode: a process supervisor restarts it after network outages (it exits after ~10 minutes offline)
- [ ] **Verified: a session actually appears in the session list after startup**

The last item is the only one that cannot be faked by a setting.

---

## The one-line test

Start a Remote Control session. Open `claude.ai/code` on a second device. Confirm the session appears with a green dot.

If it doesn't: read the startup output or the failure notification — since v2.1.154 it names the exact variable that killed eligibility — or run `claude doctor` to see which check failed. `claude doctor` evaluates the effective environment, including variables injected from `settings.json` `env` blocks that a shell grep never shows.

---

*Doctrine version: 2026-08-19. Behavior verified against the Remote Control documentation (v2.1.234+); no production incident is claimed. Rewritten after an adversarial review found the original incident framing constructed and its silent-failure claim false.*
