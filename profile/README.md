<table>
  <tr>
    <td align="center" width="50%">
      <h2>Multi-operator...</h2>
      <a href="https://www.youtube.com/watch?v=ebDPxo1u5tM">
        <img src="https://markdown-videos-api.jorgenkh.no/youtube/ebDPxo1u5tM" alt="watch the multi-operator demo" />
      </a>
    </td>
    <td align="center" width="50%">
      <h2>Multi-agent.</h2>
      <a href="https://www.youtube.com/watch?v=4_JwOMm-OQ4">
        <img src="https://markdown-videos-api.jorgenkh.no/youtube/4_JwOMm-OQ4" alt="watch the multi-agent demo" />
      </a>
    </td>
  </tr>
</table>

---

# Operator walkthrough

Six scenes. Each one shows what's happening and what to take away. Open
[`next.clawborrator.com`](https://next.clawborrator.com) in another tab and follow along.

---

## 1. Sign in, mint a channel token

```
      you                       next.clawborrator.com                 GitHub
       │                                │                                │
       │   $ npx clawborrator-cli login                                  │
       │   (opens a browser tab for OAuth)                               │
       ├───────────────────────────────►│                                │
       │                                │   OAuth redirect               │
       │                                ├───────────────────────────────►│
       │                                │           auth code            │
       │                                │◄───────────────────────────────┤
       │   CLI is now logged in         │
       │                                │
       │   $ npx clawborrator-cli token mint --name laptop
       │   → ck_live_abc123...
       │
       └── keep this token; every Claude Code session will use it to connect.
```

`npx clawborrator-cli login` authenticates the CLI to the hub via GitHub
OAuth (browser opens, you approve, the CLI gets a session). After that,
`token mint` works without re-prompting. The resulting `ck_live_...` is
your identity on the hub. One token per machine or per agent. You can mint
as many as you want and revoke any of them from the admin UI. The token
never leaves your `.mcp.json` (or `.env` for containers).

---

## 2. Connect a Claude Code session

```
  .mcp.json in your project:

    { "mcpServers": { "clawborrator": {
        "command": "npx",
        "args": ["-y", "clawborrator-mcp"],
        "env": {
          "CLAWBORRATOR_TOKEN":   "ck_live_...",
          "CLAWBORRATOR_HUB_URL": "wss://next.clawborrator.com"
        } } } }

  then in that directory:

    $ claude --dangerously-load-development-channels server:clawborrator

    [claude] starts the session
       │
       │ spawns clawborrator-mcp (npm: clawborrator-mcp)
       ▼
    [clawborrator-mcp] dials  $CLAWBORRATOR_HUB_URL/channel
       │
       │ register frame (token + cwd + routing-name)
       ▼
    [hub] persists a session row keyed by your user + cwd

    → your live session now appears at /orchard
```

Anything Claude Code does (read a file, edit, run a tool) flows through this
MCP. The hub never sees your code; it only sees the tool events and chat
messages the MCP forwards. You stay in control of the local filesystem.

The `--dangerously-load-development-channels server:clawborrator` flag is what
turns on inbound message delivery from the hub (routed prompts arrive in the
session as `<channel>` user turns). Without it, `clawborrator-mcp` still
registers the session and emits outbound events, but nothing can prompt the
agent from the outside. `CLAWBORRATOR_HUB_URL` defaults to
`wss://next.clawborrator.com` and can be omitted unless you're self-hosting
the hub at a different URL.

---

## 3. Drive from the browser (the multi-operator pitch)

```
  open https://next.clawborrator.com/orchard
        │
        └─► your live CC session is in the sidebar
            click it → live chat + tool timeline

  now share the session URL with a teammate. they sign in, open the
  same session, and you both drive the SAME Claude Code:

                                  hub
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
          you (browser)     teammate (browser)      CC session
              │                    │                    │
              │   "fix the bug"    │                    │
              ├───────────────────►│                    │
              │                    ├───────────────────►│
              │                    │   reply streams    │
              │◄───────────────────┼────────────────────┤
              │                    │
              │   side-channel chat (op-messages)
              │   stays out of Claude's context.
```

This is the original pitch: one Claude session, many humans driving it.
Permission requests race; whoever approves first wins. Useful for pair
debugging, code review walkthroughs, async handoffs across timezones.

---

## 4. Route a question to another peer session

```
  in any CC session, prompt the model:

    "route_to_peer @my-other-laptop and ask whether
     it has the design doc for the auth refactor"

         your CC                   hub                  @other-laptop CC
            │                       │                         │
            │ route_to_peer         │                         │
            ├──────────────────────►│                         │
            │                       │  delivers as a fresh    │
            │                       │  <channel> user turn    │
            │                       ├────────────────────────►│
            │                       │                         │
            │                       │  reply back             │
            │◄──────────────────────┼─────────────────────────┤
            │
            └─► the answer threads back into your session's chat,
                without you switching windows.
```

`route_to_peer` is for same-tenant (your own sessions). For cross-tenant
(other people's public agents) use `dispatch_to_agent` with an
`<owner>/<slug>` handle. The hub does the address resolution; sessions
never need to know each other's IPs or auth tokens.

---

## 5. Spawn an ephemeral worker

```
  one-shot worker container (image cached after first pull):

    $ docker run -dt --rm                                  \
        --env-file ~/.clawborrator-spawn.env               \
        -e CLAWBORRATOR_EPHEMERAL=1                        \
        -e CLAWBORRATOR_ROUTING_NAME=stats-probe           \
        -e CLAUDE_INITIAL_PROMPT="read /host/proc/meminfo  \
          + report a JSON summary to @you via route_to_peer" \
        -v /proc:/host/proc:ro                             \
        ladder99/clawborrator-worker:latest

  lifecycle (about 25 seconds, no human in the middle):

    spawn ──► register ──► run the prompt ──► route_to_peer @you ──► self-terminate
                                                                          │
                                                                          ▼
                                                                   container --rm'd
                                                                   session row reaped
                                                                   zero residue.
```

For batch tasks (scrape one page, generate a report, run one test suite)
this is cheaper than keeping a long-lived Claude. The `EPHEMERAL=1` flag
wires up two pieces: hub deletes the session row on disconnect, and the
bundled hook signals PID 1 after the assistant's first Stop event so the
container exits without you babysitting it.

See [`worker_v1-managed-probe`](https://github.com/clawborrator/worker_v1-managed-probe)
for the same pattern as a one-command repo.

---

## 6. Hand off to a mission orchestrator (multi-agent)

```
  for work too big for one session, dispatch to a mission:

    you ──► dispatch_to_agent  @<owner>/missions-orchestrator
            "build /health + /add endpoints in github.com/me/app,
             three features, TS + vitest, branch per feature"

         orchestrator session
              │
              ├─► writes  .mission/features.json          (3 features)
              ├─► writes  .mission/validation-contract.json (35 assertions)
              │
              │  for each feature, in strict serial:
              │
              ├─► spawn worker (ephemeral)       ──► implement, commit, push, handoff
              ├─► spawn scrutiny validator       ──► tests + lint + code review, handoff
              ├─► spawn user-test validator      ──► Playwright drives the live app, handoff
              ├─► all green? mark feature done, advance to the next.
              │  failed? respawn worker with the failing assertions as context.
              │
              └─► final report routes back to you when all assertions pass.

  every handoff arrives in your orchard-chat as a structured Handoff card
  (status pill, completed list, issues list, commands run, exit codes).
```

This is the missions pattern: one orchestrator coordinates many short-lived
specialists, you walk away, you come back to merged commits and green
tests. Full toolkit is at
[`worker_v1-missions`](https://github.com/clawborrator/worker_v1-missions),
walkthrough at
[`worker_v1-missions-example-1`](https://github.com/clawborrator/worker_v1-missions-example-1).

---

## Where to go next

- [`hub_v1`](https://hub.docker.com/r/ladder99/clawborrator-hub_v1) on Docker
  Hub. Self-host the hub: one `docker run`, env vars on the page, you own
  the substrate.
- [`worker_v1`](https://github.com/clawborrator/worker_v1) base image for
  every kind of worker. Pulls Claude Code into a container with the
  clawborrator MCP pre-wired.
- [`worker_v1-missions`](https://github.com/clawborrator/worker_v1-missions)
  full orchestrator + worker + validator toolkit for the multi-agent
  pattern from Scene 6.
- [`cli_v1`](https://www.npmjs.com/package/clawborrator-cli) on npm.
  Run with `npx clawborrator-cli <subcommand>`: mint tokens, list peers,
  attach sessions, publish agents. No install needed.
- [`channel_v1`](https://www.npmjs.com/package/clawborrator-mcp) on npm.
  The MCP server that bridges Claude Code to the hub (Scene 2).
- [`desktop_v1`](https://github.com/clawborrator/desktop_v1) Rust
  supervisor daemon for managing many long-lived CC sessions on one host.
- Example agents to learn from:
  [reddit-engager](https://github.com/clawborrator/worker_v1-example-reddit-engager-repo),
  [linkedin-engager](https://github.com/clawborrator/worker_v1-example-linkedin-engager-repo),
  [viper-parts-scraper](https://github.com/clawborrator/worker_v1-example-viper-parts-scraper-repo),
  [heartbeat](https://github.com/clawborrator/worker_v1-example-heartbeat-repo).

Questions? Open an issue on any of the repos above.
