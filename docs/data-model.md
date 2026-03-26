# Data Model

## Overview

The Supabase (Postgres) database contains four core tables and uses Supabase Vault for secret storage. All tables are scoped by `user_id` with Row Level Security (RLS) for multi-user support.

## Entity Relationship

```
connections (1) ──── (N) reactions
     │                      │
     │                      │
  user_id                user_id
     │                      │
     │                      │
     └──── events (1) ──── (N) executions
              │                    │
              └── reaction_id ─────┘
```

## Tables

### events

Stores every incoming event from external sources.

```sql
CREATE TABLE events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id),

  -- Event identification
  source text NOT NULL,          -- 'github', 'slack', 'linear', 'jira'
  event_type text NOT NULL,      -- 'push', 'pull_request.opened', 'message', 'issue.created'
  source_event_id text,          -- Deduplication key from the source (e.g., GitHub delivery ID)

  -- Payload
  payload jsonb NOT NULL,        -- Raw webhook payload
  metadata jsonb DEFAULT '{}',   -- Extracted/normalized fields for fast querying

  -- Processing state
  processed boolean DEFAULT false,
  processed_at timestamptz,

  -- Timestamps
  created_at timestamptz DEFAULT now(),

  -- Indexes
  CONSTRAINT events_source_event_id_unique UNIQUE (source, source_event_id)
);

-- Indexes for common queries
CREATE INDEX idx_events_user_source ON events (user_id, source, event_type);
CREATE INDEX idx_events_unprocessed ON events (user_id, processed) WHERE processed = false;
CREATE INDEX idx_events_created ON events (created_at DESC);

-- RLS
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view their own events"
  ON events FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Service role can insert events"
  ON events FOR INSERT WITH CHECK (true);  -- Edge Functions use service role
```

### reactions

User-defined rules that map events to agent actions.

```sql
CREATE TABLE reactions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id),

  -- Matching criteria
  name text NOT NULL,             -- Human-readable name: "Review PRs to main"
  source text NOT NULL,           -- Which source to match: 'github'
  event_type text NOT NULL,       -- Which event type: 'pull_request.opened'

  -- Precondition (evaluated against event payload)
  precondition jsonb,             -- Rule-based: {"field": "payload.pull_request.base.ref", "op": "eq", "value": "main"}
                                  -- Or null for "always match"

  -- Agent configuration
  agent_type text NOT NULL DEFAULT 'claude',  -- 'claude' | 'codex'
  agent_model text NOT NULL DEFAULT 'claude-sonnet-4-6',
  prompt_template text NOT NULL,  -- "Review this PR:\n\nTitle: {{payload.pull_request.title}}\nDiff URL: {{payload.pull_request.diff_url}}"
  system_prompt text,             -- Optional system prompt for the agent
  tools jsonb DEFAULT '[]',       -- Allowed tools: ["Read", "Bash", "WebFetch"]
  max_tokens integer DEFAULT 4096,

  -- Controls
  enabled boolean DEFAULT true,
  cooldown_seconds integer DEFAULT 0,  -- Minimum time between executions for same source
  last_triggered_at timestamptz,

  -- Timestamps
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

CREATE INDEX idx_reactions_match ON reactions (user_id, source, event_type, enabled);

ALTER TABLE reactions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users manage their own reactions"
  ON reactions FOR ALL USING (auth.uid() = user_id);
```

### executions

Tracks every agent run with full input/output history.

```sql
CREATE TABLE executions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id),
  event_id uuid NOT NULL REFERENCES events(id),
  reaction_id uuid NOT NULL REFERENCES reactions(id),

  -- Agent run details
  status text NOT NULL DEFAULT 'pending',  -- 'pending', 'running', 'completed', 'failed', 'cancelled'
  agent_type text NOT NULL,
  agent_model text NOT NULL,

  -- Input/Output
  rendered_prompt text NOT NULL,   -- The fully rendered prompt sent to the agent
  system_prompt text,
  agent_output text,               -- Agent's final response
  tool_calls jsonb DEFAULT '[]',   -- Log of tool calls made during execution

  -- Trigger.dev integration
  trigger_run_id text,             -- Trigger.dev run ID for linking to their dashboard
  trigger_task_id text,

  -- Timing
  started_at timestamptz,
  completed_at timestamptz,

  -- Errors
  error text,
  retry_count integer DEFAULT 0,

  -- Cost tracking
  input_tokens integer,
  output_tokens integer,
  estimated_cost_usd numeric(10, 6),

  -- Timestamps
  created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_executions_user_status ON executions (user_id, status);
CREATE INDEX idx_executions_event ON executions (event_id);
CREATE INDEX idx_executions_created ON executions (created_at DESC);

ALTER TABLE executions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view their own executions"
  ON executions FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Service role can manage executions"
  ON executions FOR ALL USING (true);  -- trigger.dev tasks use service role
```

### connections

Stores references to external service configurations and links to Vault secrets.

```sql
CREATE TABLE connections (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id),

  -- Provider info
  provider text NOT NULL,          -- 'github', 'slack', 'anthropic', 'openai'
  label text,                      -- User-friendly label: "My GitHub App"

  -- Vault references (actual secrets stored in Supabase Vault)
  api_key_secret_id uuid,          -- vault.secrets.id for API key
  webhook_secret_id uuid,          -- vault.secrets.id for webhook signing secret

  -- Connection metadata
  scopes jsonb DEFAULT '[]',       -- OAuth scopes or permissions
  config jsonb DEFAULT '{}',       -- Provider-specific config (e.g., GitHub org, Slack workspace)

  -- Status
  active boolean DEFAULT true,
  last_verified_at timestamptz,

  -- Timestamps
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),

  CONSTRAINT connections_user_provider_unique UNIQUE (user_id, provider, label)
);

ALTER TABLE connections ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users manage their own connections"
  ON connections FOR ALL USING (auth.uid() = user_id);
```

## Supabase Vault Usage

API keys and webhook secrets are stored in Supabase Vault (encrypted at rest), not in the `connections` table directly. The `connections` table only holds references (UUIDs) to Vault entries.

```sql
-- Storing a secret
SELECT vault.create_secret(
  'sk-ant-api03-xxxxx',                    -- the actual secret
  'anthropic_key_<user_id>',               -- name (unique identifier)
  'Anthropic API key for user <user_id>'   -- description
);

-- Retrieving a secret (from Edge Functions / trigger.dev tasks using service role)
SELECT decrypted_secret
FROM vault.decrypted_secrets
WHERE name = 'anthropic_key_<user_id>';
```

## Precondition Schema

Preconditions are stored as JSON in the `reactions.precondition` column. The evaluation engine supports:

### Simple field comparison

```json
{
  "field": "payload.action",
  "op": "eq",
  "value": "opened"
}
```

### Nested field access (dot notation)

```json
{
  "field": "payload.pull_request.base.ref",
  "op": "eq",
  "value": "main"
}
```

### Supported operators

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equals | `{"field": "payload.action", "op": "eq", "value": "opened"}` |
| `neq` | Not equals | `{"field": "payload.action", "op": "neq", "value": "closed"}` |
| `contains` | String contains | `{"field": "payload.comment.body", "op": "contains", "value": "deploy"}` |
| `starts_with` | String starts with | `{"field": "payload.ref", "op": "starts_with", "value": "refs/heads/"}` |
| `in` | Value in array | `{"field": "payload.action", "op": "in", "value": ["opened", "reopened"]}` |
| `exists` | Field exists | `{"field": "payload.pull_request.merged", "op": "exists"}` |
| `gt`, `gte`, `lt`, `lte` | Numeric comparison | `{"field": "payload.pull_request.additions", "op": "gt", "value": 100}` |

### Compound conditions (AND / OR)

```json
{
  "and": [
    {"field": "payload.action", "op": "eq", "value": "opened"},
    {"field": "payload.pull_request.base.ref", "op": "eq", "value": "main"},
    {
      "or": [
        {"field": "payload.pull_request.additions", "op": "gt", "value": 100},
        {"field": "payload.pull_request.title", "op": "contains", "value": "[review]"}
      ]
    }
  ]
}
```

### Null precondition

If `precondition` is null, the reaction always matches (no filtering).

## Prompt Template Syntax

Prompt templates use Mustache-style `{{}}` interpolation with dot-notation access into the event payload:

```
Review this pull request:

Title: {{payload.pull_request.title}}
Author: {{payload.pull_request.user.login}}
Description: {{payload.pull_request.body}}
Diff URL: {{payload.pull_request.diff_url}}
Files changed: {{payload.pull_request.changed_files}}

Focus on security issues and potential bugs.
```

Available variables:
- `{{source}}` — event source (github, slack, etc.)
- `{{event_type}}` — event type (push, message, etc.)
- `{{payload.*}}` — any field from the raw webhook payload
- `{{metadata.*}}` — any field from the normalized metadata
