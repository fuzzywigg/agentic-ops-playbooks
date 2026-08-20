# Remote Control in Production Agent Gateways

*Field notes on deploying Claude Code Remote Control alongside an LLM gateway, cron fleet, and exec policy stack. The failure modes are specific and silent.*

---

## What it is

Remote Control connects `claude.ai/code` or the Claude mobile app to a Claude Code session running on your machine. The session stays local — your filesystem, MCP servers, tools, and project configuration all remain on the host. The web and mobile interfaces are a window into that local session, not a cloud handoff.

Three invocation modes:

```bash
# Server mode — stays running, serves one or more remote sessions
claude remote-control

# Interactive mode — full terminal session, also reachable remotely
claude --remote-control

# From inside an existing session
/remote-control
```

Server mode is the right shape for unattended or headless deployments. Interactive mode is the right shape when you want to hand off a session you started at your desk to your phone mid-task.

---

## The kill list

Remote Control can be silenced by four independent environment variables, each of which is routinely set in production deployments for entirely unrelated reasons:

| Variable | Why you have it | What it kills |
|---|---|---|
| `DISABLE_TELEMETRY` | Privacy posture, CI hygiene | Feature-flag evaluation |
| `DO_NOT_TRACK` | Same | Feature-flag evaluation |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Airgapped or metered environments | Feature-flag evaluation |
| `DISABLE_GROWTHBOOK` | Testing or stability | Feature-flag evaluation |
| `ANTHROPIC_BASE_URL` | LLM gateway or proxy | Remote Control itself (as of v2.1.196) |

The first four disable the feature-flag evaluation that Remote Control availability depends on. The fifth is more direct: if `ANTHROPIC_BASE_URL` points anywhere other than `api.anthropic.com`, Remote Control is disabled, full stop.

This is the same defect class as [Controls That Lie](controls-that-lie.md) — a configuration that looks correct but silently removes a capability. The difference is that here the kill comes from *outside* the feature, in variables set for reasons that have nothing to do with session management.

**The countermeasure is the same:** verify the capability, not the setting. After deploying a Remote Control configuration, test that a session actually appears in the session list. A config that has never produced a visible session is a story, not a configuration.

---

## LLM gateway deployments

If you run an LLM gateway or proxy — routing Claude Code traffic through an intermediary before it reaches Anthropic — `ANTHROPIC_BASE_URL` is set to your gateway's endpoint. As of Claude Code v2.1.196, this disables Remote Control entirely.

The conflict is architectural: Remote Control communicates directly with Anthropic's infrastructure and cannot be proxied through a gateway. There is currently no supported way to use both simultaneously.

Options:

- **Unset `ANTHROPIC_BASE_URL` for sessions where Remote Control matters.** The session uses direct Anthropic access. Your gateway does not see that traffic.
- **Accept the constraint.** For unattended cron work, Remote Control isn't needed — the approval channel via exec policy handles human-in-the-loop requirements. Remote Control is most valuable for interactive sessions you want to continue on another device.
- **Document the split explicitly.** Cron agents and gateway-routed sessions: no Remote Control. Interactive dev sessions: unset the variable, use direct access. The crime is an implicit split that nobody has written down.

---

## Server mode for unattended work

`claude remote-control` in server mode stays running in your terminal, waiting for remote connections. It is the right shape when you want multiple concurrent sessions reachable from a single host:

```bash
# One-per-dir sessions (default)
claude remote-control --spawn same-dir

# Isolated git worktree per session
claude remote-control --spawn worktree

# Exactly one session, reject additional connections
claude remote-control --spawn session

# Named session, visible in the remote session list
claude remote-control --name "Night Shift"

# Cap concurrent sessions (default: 32)
claude remote-control --capacity 8
```

For an unattended host running cron automation, `--spawn worktree` is the safest shape. Sessions that could conflict on the same files get isolated git worktrees automatically.

### Resume after restart

When you stop the server with Ctrl+C, sessions stop responding remotely but are not archived. Resume them within four hours:

```bash
# Bring back all sessions from this directory
claude remote-control

# Bring back only the session the server started with
claude remote-control --continue

# Bring back one specific session by ID
claude remote-control --session-id <id>
```

The session ID is the segment of the `claude.ai/code` session URL between `/code/` and any `?`.

After four hours, the sessions are gone. Design for it: if continuity matters, use `--name` so the session is easy to find and reconnect manually.

---

## Approval channel via phone

Remote Control and exec policy are designed to work together. When `askFallback` routes an unapproved command to a human, the approval can be sent from the Claude app on your phone through the active Remote Control session. This closes the loop on the reviewed automation pattern without requiring the operator to be at a desk.

The session must be active for this to work. If `ANTHROPIC_BASE_URL` is set or any kill-list variable is present, the phone approval path does not exist. Plan accordingly: cron agents that depend on human-in-the-loop approval need Remote Control reachable from the same host.

---

## Workspace trust

Remote Control requires workspace trust. Run `claude` in your project directory at least once and accept the trust dialog before starting a Remote Control session.

One specific gotcha: Claude Code's startup trust dialog never saves trust for your home directory. If you `cd ~` and run `claude remote-control`, trust fails. Always start Remote Control from a project directory.

---

## Auto-connect

To activate Remote Control automatically on every interactive session:

```json
// ~/.claude/settings.json
{
  "remoteControlAtStartup": true
}
```

Or via `/config` inside Claude Code → **Enable Remote Control for all sessions**.

Project-level settings can *disable* auto-connect for a specific repo (set `false`) but cannot enable it (a `true` in `.claude/settings.json` is ignored for project/local scope, to prevent a checked-in file from enrolling everyone who opens the repository).

---

## Requirements checklist

Before relying on Remote Control in any deployment:

- [ ] Subscription is Pro, Max, Team, or Enterprise (API keys not supported)
- [ ] Authenticated via `/login` — not API key
- [ ] Not running on Bedrock, GCP Agent Platform, or Microsoft Foundry
- [ ] `ANTHROPIC_BASE_URL` is unset or points to `api.anthropic.com`
- [ ] `DISABLE_TELEMETRY`, `DO_NOT_TRACK`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DISABLE_GROWTHBOOK` are all unset
- [ ] Workspace trust accepted from a project directory (not home)
- [ ] On Team/Enterprise: Owner has enabled the Remote Control toggle in admin settings
- [ ] Verified: a session actually appears in the session list after startup

The last item is the only one that can't be faked by a setting.

---

## The one-line test

Start a Remote Control session. Open `claude.ai/code` on a second device. Confirm the session appears with a green dot.

If it doesn't appear, work through the kill list. The most common culprits in a gateway deployment are `ANTHROPIC_BASE_URL` and `DISABLE_TELEMETRY`. Check `env | grep -E 'DISABLE|BASE_URL|DO_NOT_TRACK|GROWTHBOOK'` before assuming the feature is broken.

---

*Playbook version: 2026-08-19. Based on Remote Control v2.1.234+ documentation and production gateway deployment patterns.*
