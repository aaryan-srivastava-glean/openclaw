---
title: "Webhook Event Lifecycle"
description: "How external events flow through OpenClaw — from HTTP request to agent action to delivered reply."
---

# Webhook Event Lifecycle

This page walks through what happens when an external system sends an event to OpenClaw. Each section is one step in the pipeline.

<Note>
OpenClaw normalizes **all** inputs — chat messages, webhooks, cron ticks, agent-to-agent calls — into the same shape: _a message arrives in a session_. The agent never knows what triggered it.
</Note>

---

## How Different Sources Enter the System

Not everything comes through the webhook endpoints. OpenClaw has three distinct ingestion paths depending on the source type:

### Chat channels (Slack, Telegram, WhatsApp, Discord, etc.)

These **bypass webhooks entirely**. Each channel has a dedicated adapter that maintains a persistent connection (WebSocket, long-poll, or platform SDK). Messages flow through the channel adapter directly into the agent session — no `/hooks` endpoint involved.

### External services via webhooks (GitHub, Stripe, PagerDuty, etc.)

These use the `/hooks` endpoints documented below. **OpenClaw does not auto-register webhooks with these services.** You configure the external service to POST to your gateway, then configure OpenClaw to recognize and route those payloads.

For example, to receive GitHub webhooks:
1. In GitHub repo settings, add a webhook pointing at `https://<your-gateway>/hooks/github`
2. Set the secret/token to match your OpenClaw `hooks.token`
3. In OpenClaw config, add a mapping that matches `path: "github"` and tells the gateway what to do with the payload

### Gmail (the only built-in preset)

Gmail is a special case — it uses Google Cloud Pub/Sub instead of direct webhooks. OpenClaw ships a `"gmail"` preset with built-in templates and a CLI wizard (`openclaw webhooks gmail setup`). All other services use the generic mapping system.

### Cron and heartbeats

Internal timers that fire on a schedule. Cron jobs run in isolated sessions (`cron:<jobId>`); heartbeats run in the main session every 30 minutes. Both enter the same agent execution path as webhooks.

### Agent-to-agent

One agent calls the `sessions_send` tool to message another agent. The target agent sees it as a normal message in its session.

---

## The Webhook Pipeline (Steps 1-8)

Everything below applies to the three `/hooks` endpoints. Chat channels, cron, and agent-to-agent use different entry points but converge at Step 5.

### 1. The HTTP Request Arrives

External systems send a `POST` to one of three endpoints on the gateway:

| Endpoint | Use case | What you send |
|----------|----------|---------------|
| `/hooks/wake` | Poke the main agent with a short message | `{ "text": "Check PRs", "mode": "now" }` |
| `/hooks/agent` | Run a specific agent in an isolated session | `{ "message": "...", "agentId": "reviewer" }` |
| `/hooks/<name>` | Route an arbitrary payload through a transform | Raw GitHub/Stripe/Gmail JSON |

All three require `Authorization: Bearer <token>` (or `X-OpenClaw-Token` header). Tokens in query parameters are rejected.

### 2. Authentication and Validation

Before touching the payload, the gateway checks:

1. **Token** — constant-time comparison, rate-limited to 20 failures per 60 s per IP.
2. **Payload size** — capped at 256 KB by default (`hooks.maxBodyBytes`).
3. **Agent ID** — if the caller specifies an `agentId`, it must appear in `hooks.allowedAgentIds`.
4. **Session key** — callers can only set a custom `sessionKey` when `allowRequestSessionKey: true` and the key matches `allowedSessionKeyPrefixes`.

Failures return standard HTTP codes: `401` (bad token), `429` (rate-limited), `400` (policy violation), `413` (too large).

### 3. Payload Transformation

Each endpoint normalizes its input differently:

**`/hooks/wake`** — Minimal path. The `text` field becomes a system event in the main session. If `mode: "now"`, a heartbeat fires immediately; if `"next-heartbeat"`, it waits for the next 30-minute tick.

**`/hooks/agent`** — The full-featured path. The payload controls routing, delivery, model, and timeout:

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

**`/hooks/<name>` (Mapped Hooks)** — For raw payloads from third-party webhooks. The gateway matches `<name>` against your configured hook mappings. You define mappings in your OpenClaw config:

```json5
// In ~/.openclaw/config.json5
{
  hooks: {
    mappings: [
      {
        id: "github-prs",
        match: { path: "github" },             // matches POST /hooks/github
        action: "agent",
        agentId: "security-reviewer",
        messageTemplate: "PR #{{payload.pull_request.number}}: {{payload.pull_request.title}}",
        sessionKey: "hook:github-pr-{{payload.pull_request.number}}",
        deliver: true,
        channel: "slack",
        to: "#security-reviews",
        transform: { module: "github-filter.js" }  // optional
      }
    ]
  }
}
```

The optional transform module (a JS/TS function in `~/.openclaw/hooks/transforms/`) can reshape the payload, skip it entirely (return `null` -> HTTP 204), or override any field. Templates use `{{payload.field}}` syntax to extract values from the raw webhook JSON.

### 4. Queue and Session Isolation

The transformed event enters a lane-aware queue. Two rules govern concurrency:

- **Same session -> serial.** Two webhooks targeting `hook:github-pr-456` run one after the other. No races.
- **Different sessions -> parallel.** `hook:github-pr-456` and `hook:github-pr-789` run concurrently.

Cron jobs run in their own lane, so scheduled work never blocks webhook-triggered work (and vice versa).

If a new event arrives while an agent turn is already running in the same session, the queue holds it until the current turn finishes, then delivers it as the next message.

### 5. Agent Execution

Every event type converges here — webhooks, chat messages, cron, and agent-to-agent calls all run through `runEmbeddedPiAgent()`. It:

1. **Acquires** the session write lock.
2. **Builds the system prompt** — OpenClaw defaults, the agent's `SOUL.md`, `AGENTS.md`, skills, memory context, and the webhook message itself.
3. **Calls the model** — streaming, with tool use enabled.
4. **Executes tool calls** — file ops, shell commands, messaging, web fetch, MCP servers, cron management, or cross-agent messaging.
5. **Enforces the timeout** — 600 s default, overridable per-webhook via `timeoutSeconds`.
6. **Handles context overflow** — compacts history or resets the session if needed.

The agent doesn't know it was triggered by a webhook. It sees a message in its session and acts on it using whatever tools are available.

### 6. Output Delivery

After the agent turn completes, the gateway decides where to send the result:

1. **Should it deliver?** — Only if `deliver: true` in the webhook payload.
2. **Where?** — The `channel` and `to` fields from the payload. `"last"` means the most recent conversation channel. If omitted, delivery is internal-only.
3. **How?** — Each channel (Slack, Telegram, Discord, WhatsApp, etc.) has an outbound adapter that normalizes the output to platform-specific formatting and size limits.

For isolated hook sessions, a summary is also posted back to the main session so the operator can see what happened without switching contexts.

If delivery fails, the message is queued for retry with exponential backoff (5 s, 25 s, 2 min, 10 min — up to 5 retries).

### 7. Lifecycle Events Fire

After the turn, internal hooks run synchronously in registration order:

| Event | When it fires |
|-------|---------------|
| `agent:start` | Agent run begins |
| `agent:end` | Agent run completes |
| `agent:error` | Agent run errors |
| `message:sent` | Outbound message delivered |

These are extension points — for example, an audit-logging hook can listen to `agent:end` to record every webhook-triggered run. Failures in hooks are logged but never block the pipeline.

See [Hooks](/automation/hooks) for the full event list and how to write custom handlers.

### 8. State Persisted

Finally, the gateway writes durable state:

- **Session transcript** — JSONL appended to `~/.openclaw/sessions/`.
- **Session metadata** — `sessions.json` updated.
- **Agent memory** — any `MEMORY.md` or `memory/*.md` changes persisted.

---

## End-to-End Example: GitHub PR -> Security Review -> Slack

This traces a single event through every step.

### What you set up (once)

**In GitHub:** Add a webhook in your repo settings:
- URL: `https://your-gateway.example.com/hooks/github`
- Content type: `application/json`
- Secret: your OpenClaw hooks token
- Events: Pull requests

**In OpenClaw config** (`~/.openclaw/config.json5`):
```json5
{
  hooks: {
    enabled: true,
    token: "${OPENCLAW_HOOKS_TOKEN}",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:"],
    allowedAgentIds: ["security-reviewer"],
    mappings: [
      {
        id: "github-prs",
        match: { path: "github" },
        action: "agent",
        agentId: "security-reviewer",
        messageTemplate: "Review PR #{{payload.pull_request.number}}: {{payload.pull_request.title}}\n\nDiff: {{payload.pull_request.diff_url}}\nAuthor: {{payload.pull_request.user.login}}",
        sessionKey: "hook:github-pr-{{payload.pull_request.number}}",
        deliver: true,
        channel: "slack",
        to: "#security-reviews",
        timeoutSeconds: 300,
        transform: { module: "github-filter.js" }
      }
    ]
  }
}
```

**Optional transform** (`~/.openclaw/hooks/transforms/github-filter.js`):
```js
export default async (ctx) => {
  // Only process opened PRs, skip everything else
  if (ctx.payload.action !== "opened") return null;
  // Let the mapping handle the rest
  return {};
};
```

### What happens when a PR is opened

```
1. ARRIVE     GitHub POSTs raw JSON to /hooks/github
              ──────────────────────────────────────

2. AUTH       Gateway checks Bearer token ✓
              Payload under 256 KB ✓
              ──────────────────────────────────────

3. TRANSFORM  Path "github" matches mapping "github-prs"
              Transform runs: action=opened → continue
              Templates resolve:
                message  = "Review PR #456: Fix auth bypass\n\n..."
                session  = "hook:github-pr-456"
                agentId  = "security-reviewer"
              ──────────────────────────────────────

4. QUEUE      Session "hook:github-pr-456" is idle → run now
              (If PR #456 already had a review in progress,
               this would wait for it to finish first)
              ──────────────────────────────────────

5. EXECUTE    runEmbeddedPiAgent() starts for security-reviewer
              Agent sees: "Review PR #456: Fix auth bypass..."
              Agent uses tools: fetches diff, reads code, checks
              for vulnerabilities, writes a review summary
              ──────────────────────────────────────

6. DELIVER    deliver=true → send to slack #security-reviews
              Slack adapter formats the review as a message
              Summary also posted to main session
              ──────────────────────────────────────

7. HOOKS      agent:end fires → audit log records the run
              message:sent fires → delivery confirmed
              ──────────────────────────────────────

8. PERSIST    Session transcript saved to disk
              Agent memory updated if anything was learned
```

### What the agent sees

Just a message in its session:

> Review PR #456: Fix auth bypass
>
> Diff: https://github.com/example/repo/pull/456.diff
> Author: octocat

It has no idea this came from a webhook. It responds using its tools, and the gateway handles delivery.

---

## Quick Reference: All Input Types

| Source | How it enters | Who registers it | Session | Webhook pipeline? |
|--------|--------------|-------------------|---------|-------------------|
| **Chat** (Slack, Telegram, etc.) | Channel adapter (persistent connection) | OpenClaw channel setup | Main or bound | No — direct to agent |
| **Webhook** (GitHub, Stripe, etc.) | `POST /hooks/<name>` | You, in the external service | `hook:<key>` (isolated) | Yes (Steps 1-8) |
| **Gmail** | `POST /hooks/gmail` via Pub/Sub | `openclaw webhooks gmail setup` | `hook:gmail:<id>` | Yes, with built-in preset |
| **Wake** | `POST /hooks/wake` | You, via curl/script | Main session | Yes (simplified) |
| **Cron** | Internal timer | `openclaw cron` or agent tool | `cron:<jobId>` (isolated) | No — enters at Step 5 |
| **Heartbeat** | Internal timer (30 min) | Always on | Main session | No — enters at Step 5 |
| **Agent-to-agent** | `sessions_send` tool | Another agent | Target agent's session | No — enters at Step 5 |

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
