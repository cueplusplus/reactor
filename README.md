# Reactor

Event-driven AI agent platform. Listen to external events (GitHub, Slack, Linear, etc.), evaluate them against user-defined rules, and run AI agents (Claude or Codex) when preconditions are met.

## How It Works

```
External Event → Webhook → Supabase → Evaluate Rules → Trigger.dev → AI Agent → Results
```

1. **Events arrive** via webhooks from GitHub, Slack, Linear, Jira, or any service
2. **Stored** in Supabase (Postgres) as durable event records
3. **Matched** against user-defined reactions (source + event type)
4. **Preconditions checked** — rule-based JSON predicates on the event payload
5. **Agent dispatched** via Trigger.dev — no timeouts, retries, full observability
6. **Results stored** — execution history with full input/output audit trail

## Stack

| Layer | Technology |
|-------|-----------|
| Event ingestion | Supabase Edge Functions |
| Database | Supabase (Postgres + RLS + Vault) |
| Agent execution | Trigger.dev v4 |
| AI models | Claude (Anthropic) / Codex (OpenAI) |

## Example: Auto-review PRs to main

```json
{
  "name": "Review PRs to main",
  "source": "github",
  "event_type": "pull_request.opened",
  "precondition": {
    "field": "payload.pull_request.base.ref",
    "op": "eq",
    "value": "main"
  },
  "agent_type": "claude",
  "agent_model": "claude-sonnet-4-6",
  "prompt_template": "Review this PR:\n\nTitle: {{payload.pull_request.title}}\nAuthor: {{payload.pull_request.user.login}}\n\nFocus on bugs and security issues."
}
```

## Multi-User

Each user brings their own API keys (Anthropic, OpenAI). Keys are stored encrypted in Supabase Vault. No shared billing, no shared rate limits.

## Documentation

- [Architecture](docs/architecture.md) — system overview and component diagram
- [Data Model](docs/data-model.md) — Supabase schema, RLS, Vault usage
- [Event Sources](docs/event-sources.md) — supported sources and webhook setup
- [Trigger.dev Integration](docs/trigger-dev.md) — agent task implementation
- [Reactions](docs/reactions.md) — preconditions, prompt templates, examples
- [Authentication](docs/authentication.md) — API keys, multi-user, security
- [Getting Started](docs/getting-started.md) — step-by-step setup guide

## Quick Start

See [Getting Started](docs/getting-started.md) for the full guide.

```bash
# 1. Set up Supabase tables
supabase db push

# 2. Deploy Edge Functions
supabase functions deploy webhook-github
supabase functions deploy evaluate-event

# 3. Deploy Trigger.dev tasks
npx trigger.dev@latest deploy

# 4. Store your API key
# (via Supabase SQL editor — see Getting Started)

# 5. Point GitHub/Slack webhooks to your Edge Functions

# 6. Create a reaction and watch it work
```
