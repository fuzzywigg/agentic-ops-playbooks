# Remote Control in Production Agent Gateways

*The session was configured. The dashboard would have shown it connected. It was silently dead — killed by a variable set two months earlier for CI hygiene.*

---

## The defect

Remote Control was enabled in the gateway's configuration. The feature had been tested. The docs said it worked. What nobody checked: `DISABLE_TELEMETRY` was set in the shell environment for an unrelated reason — it suppresses telemetry in CI pipelines, and someone had carried it over into the production session profile.

`DISABLE_TELEMETRY` disables the feature-flag evaluation that Remote Control availability depends on. Remote Control does not log a warning. It does not emit a failure. It simply does not activate. The session starts, looks normal, and Remote Control is absent.

This is the same defect class as [Controls That Lie](controls-that-lie.md) — a configuration that reports a value it doesn't have. The difference is that the kill comes from *outside* the feature, in variables set for reasons that have nothing to do with session management.

---

## The kill list

Five independent variables can disable Remote Control. Each is routinely present in production gateway deployments for entirely unrelated reasons:

| Variable | Why you have it | What it kills |
|---|---|---|
| `DISABLE_TELEMETRY` | CI hygiene, privacy posture | Feature-flag evaluation |
| `DO_NOT_TRACK` | Same | Feature-flag evaluation |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Airgapped or metered environments | Feature-flag evaluation |
| `DISABLE_GROWTHBOOK` | Testing or stability | Feature-flag evaluation |
| `ANTHROPIC_BASE_URL` | LLM gateway or proxy | Remote Control itself (v2.1.196+) |

The first four kill via feature-flag evaluation. The fifth is more direct: if `ANTHROPIC_BASE_URL` points anywhere other than `api.anthropic.com`, Remote Control is disabled at the architectural level — it communicates directly with Anthropic's infrastructure and cannot be proxied.

**The countermeasure is verification, not configuration.** Check for all five variables before deploying:

```sh
env | grep -E 'DISABLE|BASE_URL|DO_NOT_TRACK|GROWTHBOOK'
```

Then verify the capability itself: start a session, open `claude.ai/code` on a second device, confirm a session appears with a green dot. A configuration that has never produced a visible session is a story, not a configuration.

---

## LLM gateway deployments

If your deployment routes Claude Code traffic through an intermediary (`ANTHROPIC_BASE_URL` set), Remote Control is unavailable on that host. The conflict is architectural, not a misconfiguration — there is no workaround.

Options, stated plainly:

- **Unset `ANTHROPIC_BASE_URL` for sessions where Remote Control matters.** The session uses direct Anthropic access; your gateway does not see that traffic.
- **Accept the split.** Cron agents and gateway-routed sessions: no Remote Control. Interactive sessions: unset the variable, use direct access.
- **Document the split explicitly.** The dangerous version is an implicit split that nobody has written down. Someone assumes Remote Control is available, it isn't, they discover it during an incident rather than during setup.

---

## What it is

Remote Control connects `claude.ai/code` or the Claude mobile app to a Claude Code session running on your machine. The session stays local — filesystem, MCP servers, tools, and project configuration all remain on the host. The web and mobile interfaces are a window into that local session, not a cloud handoff.

Three invocation modes:

```bash
# Server mode — stays running, serves one or more remote sessions
claude remote-control

# Interactive mode — full terminal session, also reachable remotely
claude --remote-control

# From inside an existing session
/remote-control
```

Server mode is the right shape for unattended or headless deployments. Interactive mode is the right shape for handing off a session you started at your desk to your phone mid-task.

### Server mode flags

```bash
claude remote-control --name "Night Shift"          # visible session title
claude remote-control --spawn worktree              # isolated git worktree per session
claude remote-control --spawn session               # single session, reject additional
claude remote-control --capacity 8                  # cap concurrent sessions (default: 32)
claude remote-control --continue                    # resume last session from this directory
claude remote-control --session-id <id>             # resume one session by ID
```

`--spawn worktree` is the safest shape for unattended work — sessions that could conflict on the same files get isolated git worktrees automatically.

### Resume after restart

Sessions survive a Ctrl+C restart for approximately four hours. Within that window:

```bash
claude remote-control           # bring back all sessions from this directory
claude remote-control --continue    # bring back only the session the server started with
claude remote-control --session-id <id>   # bring back one session by ID
```

The session ID is the segment of the `claude.ai/code` URL between `/code/` and any `?`.

---

## Approval channel via phone

Remote Control and exec policy compose. When `askFallback` routes an unapproved command to a human, the approval can be sent from the Claude app on your phone through the active Remote Control session. This closes the reviewed-automation loop without requiring the operator to be at a desk.

This integration only exists when Remote Control is reachable. If `ANTHROPIC_BASE_URL` is set or any kill-list variable is present, the phone approval path is unavailable. For cron agents that depend on human-in-the-loop approval, Remote Control must be reachable from the same host — verify this during deployment, not during the first incident.

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
- [ ] `DISABLE_TELEMETRY`, `DO_NOT_TRACK`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DISABLE_GROWTHBOOK` all unset
- [ ] Workspace trust accepted from a project directory (not home)
- [ ] On Team/Enterprise: Owner has enabled Remote Control in admin settings
- [ ] **Verified: a session actually appears in the session list after startup**

The last item is the only one that cannot be faked by a setting.

---

## The one-line test

Start a Remote Control session. Open `claude.ai/code` on a second device. Confirm the session appears with a green dot. If it doesn't, run `env | grep -E 'DISABLE|BASE_URL|DO_NOT_TRACK|GROWTHBOOK'` and work through the kill list.

---

*Playbook version: 2026-08-19. Defect class first observed in a gateway deployment where `DISABLE_TELEMETRY` was inherited from CI environment config.*
