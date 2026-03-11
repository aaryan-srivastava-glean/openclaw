---
title: "Webhook Event Lifecycle"
description: "How external events flow through OpenClaw — from HTTP request to agent action to delivered reply."
---

# Webhook Event Lifecycle

This page walks through what happens when an external system (GitHub, Stripe, Gmail, etc.) sends an event to OpenClaw. Each section is one step in the pipeline.

<Note>
OpenClaw normalizes **all** inputs — chat messages, webhooks, cron ticks, agent-to-agent calls — into the same shape: _a message arrives in a session_. The agent never knows what triggered it.
</Note>

---

## 1. The HTTP Request Arrives

External systems send a `POST` to one of three endpoints on the gateway:

| Endpoint | Use case | What you send |
|----------|----------|---------------|
| `/hooks/wake` | Poke the main agent with a short message | `{ "text": "Check PRs", "mode": "now" }` |
| `/hooks/agent` | Run a specific agent in an isolated session | `{ "message": "...", "agentId": "reviewer" }` |
| `/hooks/<name>` | Route an arbitrary payload through a transform | Raw GitHub/Stripe/Gmail JSON |

All three require `Authorization: Bearer <token>` (or `X-OpenClaw-Token` header). Tokens in query parameters are rejected.

---

## 2. Authentication and Validation

Before touching the payload, the gateway checks:

1. **Token** — constant-time comparison, rate-limited to 20 failures per 60 s per IP.
2. **Payload size** — capped at 256 KB by default (`hooks.maxBodyBytes`).
3. **Agent ID** — if the caller specifies an `agentId`, it must appear in `hooks.allowedAgentIds`.
4. **Session key** — callers can only set a custom `sessionKey` when `allowRequestSessionKey: true` and the key matches `allowedSessionKeyPrefixes`.

Failures return standard HTTP codes: `401` (bad token), `429` (rate-limited), `400` (policy violation), `413` (too large).

---

## 3. Payload Transformation

Each endpoint normalizes its input differently:

### `/hooks/wake`

Minimal path. The `text` field becomes a system event in the main session. If `mode: "now"`, a heartbeat fires immediately; if `"next-heartbeat"`, it waits for the next 30-minute tick.

### `/hooks/agent`

The full-featured path. The payload controls routing, delivery, model, and timeout:

```json
{
  "message": "New PR opened: fix auth bypass in login.ts",
  "agentId": "security-reviewer",
  "sessionKey": "hook:github-pr-456",
  "wakeMode": "now",
  "deliver": true,
  "channel": "slack",
  "to": "#security-reviews",
  "model": "anthropic/claude-sonnet-4-20250514",
  "timeoutSeconds": 300
}
```

The gateway resolves a session key (custom or generated `hook:<uuid>`), validates the agent policy, then dispatches the turn asynchronously — the HTTP response returns a `runId` immediately.

### `/hooks/<name>` (Mapped Hooks)

For raw payloads from third-party webhooks. The gateway matches the `<name>` against configured hook mappings:

```json
{
  "id": "stripe-events",
  "match": { "path": "stripe" },
  "action": "agent",
  "message": "New Stripe event: {{event.type}}",
  "sessionKey": "hook:stripe:{{event.id}}",
  "transform": { "module": "./transforms/stripe.js" }
}
```

The transform module (a JS/TS function in `~/.openclaw/hooks/transforms/`) can reshape the payload, skip it entirely (return `null` → HTTP 204), or override any field. The result feeds into the same agent dispatch path as `/hooks/agent`.

---

## 4. Queue and Session Isolation

The transformed event enters a lane-aware queue. Two rules govern concurrency:

- **Same session → serial.** Two webhooks targeting `hook:github-pr-456` run one after the other. No races.
- **Different sessions → parallel.** `hook:github-pr-456` and `hook:github-pr-789` run concurrently.

Cron jobs run in their own lane, so scheduled work never blocks webhook-triggered work (and vice versa).

If a new event arrives while an agent turn is already running in the same session, the queue holds it until the current turn finishes, then delivers it as the next message.

---

## 5. Agent Execution

Every event type converges on the same function: `runEmbeddedPiAgent()`. This is the core agent loop. It:

1. **Acquires** the session write lock.
2. **Builds the system prompt** — OpenClaw defaults, the agent's `SOUL.md`, `AGENTS.md`, skills, memory context, and the webhook message itself.
3. **Calls the model** — streaming, with tool use enabled.
4. **Executes tool calls** — file ops, shell commands, messaging, web fetch, MCP servers, cron management, or cross-agent messaging.
5. **Enforces the timeout** — 600 s default, overridable per-webhook via `timeoutSeconds`.
6. **Handles context overflow** — compacts history or resets the session if needed.

The agent doesn't know it was triggered by a webhook. It sees a message in its session and acts on it using whatever tools are available.

---

## 6. Output Delivery

After the agent turn completes, the gateway decides where to send the result:

1. **Should it deliver?** — Only if `deliver: true` in the webhook payload.
2. **Where?** — The `channel` and `to` fields from the payload. `"last"` means the most recent conversation channel. If omitted, delivery is internal-only.
3. **How?** — Each channel (Slack, Telegram, Discord, WhatsApp, etc.) has an outbound adapter that normalizes the output to platform-specific formatting and size limits.

For isolated hook sessions, a summary is also posted back to the main session so the operator can see what happened without switching contexts.

---

## 7. Lifecycle Events Fire

After the turn, internal hooks run synchronously in registration order:

| Event | When it fires |
|-------|---------------|
| `agent:start` | Agent run begins |
| `agent:end` | Agent run completes |
| `agent:error` | Agent run errors |
| `message:sent` | Outbound message delivered |

These are extension points — for example, an audit-logging hook can listen to `agent:end` to record every webhook-triggered run. Failures in hooks are logged but never block the pipeline.

See [Hooks](/automation/hooks) for the full event list and how to write custom handlers.

---

## 8. State Persisted

Finally, the gateway writes durable state:

- **Session transcript** — JSONL appended to `~/.openclaw/sessions/`.
- **Session metadata** — `sessions.json` updated.
- **Agent memory** — any `MEMORY.md` or `memory/*.md` changes persisted.
- **Delivery queue** — if delivery failed, the message is queued for retry with exponential backoff (5 s → 25 s → 2 min → 10 min, up to 5 retries).

---

## Putting It Together

Here's the full pipeline for a GitHub webhook that triggers a security review and posts results to Slack:

```
GitHub sends POST /hooks/github
  │
  ▼
Gateway authenticates token                          ← Step 2
  │
  ▼
Hook mapping matches path "github"                   ← Step 3
Transform extracts PR title, diff URL, author
Produces: { message: "Review PR #456: ...",
            agentId: "security-reviewer",
            sessionKey: "hook:github-pr-456",
            deliver: true, channel: "slack",
            to: "#security-reviews" }
  │
  ▼
Queue checks: is hook:github-pr-456 busy?            ← Step 4
  No → run immediately
  Yes → wait for current turn to finish
  │
  ▼
runEmbeddedPiAgent() starts                          ← Step 5
Agent reads PR diff, checks for vulnerabilities,
writes review comment via GitHub tool
  │
  ▼
Agent turn completes                                  ← Step 6
deliver=true + channel=slack + to=#security-reviews
→ Slack adapter posts formatted review summary
→ Summary also posted to main session
  │
  ▼
agent:end hook fires → audit log written              ← Step 7
Session transcript + memory persisted                 ← Step 8
```

---

## Quick Reference: All Input Types

| Input | Entry point | Session | Concurrent with others? |
|-------|-------------|---------|------------------------|
| Chat message | Channel adapter | Main or bound session | Serial within session |
| `/hooks/wake` | HTTP POST | Main session | Serial within session |
| `/hooks/agent` | HTTP POST | `hook:<key>` (isolated) | Yes, across sessions |
| `/hooks/<name>` | HTTP POST | Per mapping config | Yes, across sessions |
| Cron job | Timer | `cron:<jobId>` (isolated) | Yes (separate lane) |
| Heartbeat | Timer (30 min) | Main session | Serial within session |
| Agent-to-agent | `sessions_send` tool | Target agent's session | Serial within session |

---

## Recommended Starter Config

```json5
{
  hooks: {
    enabled: true,
    token: "${OPENCLAW_HOOKS_TOKEN}",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false,
    allowedSessionKeyPrefixes: ["hook:"],
    allowedAgentIds: ["hooks", "main"],
    path: "/hooks"
  }
}
```

Start with `allowRequestSessionKey: false` (all webhooks share one session). Enable it when you need per-event isolation (e.g., one session per PR).
