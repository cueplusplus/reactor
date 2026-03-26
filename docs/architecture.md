# Architecture

## Overview

Reactor is an event-driven AI agent platform. It listens to external events (GitHub commits, PR comments, Slack messages, etc.), evaluates them against user-defined rules, checks preconditions, and runs AI agents (Claude or Codex) with prompts defined for that event type.

## System Diagram

```
External Sources (GitHub, Slack, Linear, Jira, etc.)
        │
        │  Webhooks (HTTPS POST)
        ▼
┌──────────────────────────┐
│   Supabase Edge Functions│  Webhook receivers (one per source)
│   /functions/v1/webhook-*│  - Validate webhook signatures
│                          │  - Normalize payload
│                          │  - INSERT into `events` table
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│   Supabase Database      │  Postgres
│   `events` table         │  - Stores all incoming events
│                          │  - RLS scoped by user_id
└──────────┬───────────────┘
           │
           │  Database Webhook (pg_net) on INSERT
           ▼
┌──────────────────────────┐
│   Evaluation Edge Func   │  /functions/v1/evaluate-event
│                          │  - Query matching `reactions`
│                          │  - Evaluate preconditions
│                          │  - Render prompt templates
│                          │  - Dispatch to trigger.dev
└──────────┬───────────────┘
           │
           │  tasks.trigger() via SDK
           ▼
┌──────────────────────────┐
│   Trigger.dev            │  Background task execution
│                          │  - No execution timeouts
│                          │  - CRIU checkpoint/restore
│                          │  - Retries + queues
│                          │  - Observability (OpenTelemetry)
│                          │
│   ┌────────────────────┐ │
│   │  Agent Task        │ │
│   │  - Load user's key │ │
│   │  - Claude API call │ │
│   │  - OR Codex SDK    │ │
│   │  - Tool execution  │ │
│   │  - Multi-step loop │ │
│   └────────┬───────────┘ │
└────────────┼─────────────┘
             │
             │  Write results back
             ▼
┌──────────────────────────┐
│   Supabase Database      │
│   `executions` table     │  - Status, output, errors
│                          │  - Linked to event + reaction
│   Supabase Realtime ────────► Frontend dashboard (optional)
└──────────────────────────┘
```

## Component Responsibilities

### Supabase Edge Functions (Webhook Receivers)

One Edge Function per external source. Each function:

1. Receives the HTTPS POST from the external service
2. Validates the webhook signature (e.g., GitHub uses HMAC-SHA256, Slack uses signing secrets)
3. Normalizes the payload into a common event format
4. Inserts into the `events` table with source, event type, and raw payload

Edge Functions run on Deno with 256 MB memory and a 400s wall-clock limit on paid plans. They are stateless and scale automatically.

### Supabase Database (Event Store)

Postgres serves as the durable event store. Key tables:

- `events` — all incoming events with raw payloads
- `reactions` — user-defined rules mapping events to agent actions
- `executions` — agent run history with inputs, outputs, status
- `connections` — user credentials stored via Supabase Vault

Database Webhooks (pg_net) fire on INSERT to the `events` table, triggering the evaluation pipeline.

### Evaluation Pipeline

A dedicated Edge Function (`evaluate-event`) handles the core logic:

1. Query `reactions` table for rules matching the event's source and event_type
2. For each matching reaction, evaluate the precondition against the event payload
3. If the precondition is met, render the prompt template with data from the event payload
4. Create an `executions` record with status `pending`
5. Call `tasks.trigger()` to dispatch to trigger.dev

### Trigger.dev (Agent Execution)

Trigger.dev v4 runs the AI agent tasks. Key properties:

- **No execution timeouts** — agent loops can run indefinitely
- **Checkpoint/restore** — process state is saved during waits (API calls, etc.), no charge for idle time
- **Retries** — configurable retry policies with exponential backoff
- **Concurrency control** — queues with configurable limits
- **Observability** — OpenTelemetry traces for every run

Each task:
1. Retrieves the user's API key from Supabase Vault
2. Initializes the appropriate AI client (Anthropic SDK or Codex SDK)
3. Runs the agent with the rendered prompt and configured tools
4. Writes results back to the `executions` table
5. Optionally triggers follow-up actions (post a GitHub comment, send a Slack message, etc.)

### Supabase Realtime (Optional Dashboard)

Supabase Realtime pushes database changes over WebSockets. A frontend dashboard can subscribe to changes on `events` and `executions` tables to show live status updates.

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Event ingestion | Supabase Edge Functions (Deno) | Webhook receivers |
| Event store | Supabase (Postgres) | Durable storage, RLS |
| Event routing | Supabase Database Webhooks (pg_net) | Trigger evaluation on INSERT |
| Evaluation | Supabase Edge Functions | Precondition checks, prompt rendering |
| Agent execution | Trigger.dev v4 (TypeScript) | Reliable background task execution |
| AI models | Claude API (Anthropic SDK) / Codex SDK | Agent intelligence |
| Secret storage | Supabase Vault | Encrypted API keys and tokens |
| Auth | Supabase Auth | User management, RLS |
| Realtime UI | Supabase Realtime | Live dashboard updates |

## Design Decisions

### Why Supabase Edge Functions for webhook reception (not trigger.dev)?

Trigger.dev v4 removed built-in webhook triggers. It expects your application layer to handle webhook reception and call `tasks.trigger()`. Supabase Edge Functions are a natural fit — they're HTTP endpoints that can validate webhooks and write to the database, all within the same Supabase project.

### Why trigger.dev for agent execution (not Edge Functions)?

Edge Functions have a 2-second CPU time limit per request (400s wall-clock). AI agent loops can be CPU-intensive and long-running. Trigger.dev has no timeouts, supports checkpoint/restore, and provides retries and observability — purpose-built for this workload.

### Why separate event storage from evaluation?

Decoupling ingestion from evaluation means:
- Events are never lost even if evaluation fails
- You can replay events through updated rules
- You can add new reactions retroactively
- Audit trail of everything that happened

### Why not Supabase Queues (pgmq)?

Supabase Queues provide guaranteed delivery and are a good alternative to Database Webhooks for mission-critical flows. The architecture can be upgraded to use pgmq if at-most-once delivery from pg_net proves insufficient. For a personal-use platform, pg_net is simpler and sufficient.
