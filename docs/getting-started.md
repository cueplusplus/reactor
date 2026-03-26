# Getting Started

## Prerequisites

- [Node.js](https://nodejs.org/) 18+
- [Supabase CLI](https://supabase.com/docs/guides/cli) (`npm install -g supabase`)
- [Trigger.dev CLI](https://trigger.dev/docs) (`npm install @trigger.dev/sdk`)
- A Supabase project (free tier works for development)
- A Trigger.dev account (free tier: 10 concurrent runs)
- An Anthropic API key and/or OpenAI API key

## Step 1: Set Up Supabase

### Create the project

```bash
# Login to Supabase
supabase login

# Link to your project (or create one at supabase.com)
supabase link --project-ref <your-project-ref>
```

### Run the database migrations

Create the core tables, Vault helpers, and RLS policies:

```bash
# Create migration
supabase migration new create_reactor_tables

# Paste the SQL from docs/data-model.md into the migration file
# Then apply:
supabase db push
```

Key SQL to include in the migration:

```sql
-- Enable Vault extension (if not already)
CREATE EXTENSION IF NOT EXISTS supabase_vault;

-- Create tables: events, reactions, executions, connections
-- (see docs/data-model.md for full schema)

-- Create Vault helper functions
CREATE OR REPLACE FUNCTION create_or_update_secret(
  secret_name text, secret_value text, secret_description text DEFAULT ''
) RETURNS void LANGUAGE plpgsql SECURITY DEFINER AS $$
DECLARE existing_id uuid;
BEGIN
  SELECT id INTO existing_id FROM vault.secrets WHERE name = secret_name;
  IF existing_id IS NOT NULL THEN
    UPDATE vault.secrets SET secret = secret_value, description = secret_description, updated_at = now() WHERE id = existing_id;
  ELSE
    INSERT INTO vault.secrets (name, secret, description) VALUES (secret_name, secret_value, secret_description);
  END IF;
END; $$;

CREATE OR REPLACE FUNCTION get_decrypted_secret(secret_name text)
RETURNS TABLE (decrypted_secret text) LANGUAGE sql SECURITY DEFINER AS $$
  SELECT decrypted_secret FROM vault.decrypted_secrets WHERE name = secret_name;
$$;

-- Database webhook: fires the evaluation Edge Function on new events
-- (configure via Supabase Dashboard > Database > Webhooks)
```

### Create the Database Webhook

In Supabase Dashboard:

1. Go to **Database > Webhooks**
2. Click **Create a new webhook**
3. Table: `events`
4. Events: `INSERT`
5. Type: **Supabase Edge Function**
6. Edge Function: `evaluate-event` (create this next)

## Step 2: Create Edge Functions

### Webhook receiver (GitHub example)

```bash
supabase functions new webhook-github
```

Edit `supabase/functions/webhook-github/index.ts` — see [Event Sources docs](./event-sources.md) for implementation.

### Event evaluator

```bash
supabase functions new evaluate-event
```

```typescript
// supabase/functions/evaluate-event/index.ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

Deno.serve(async (req) => {
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  const { record } = await req.json(); // Database webhook payload
  const event = record;

  // Find matching reactions
  const { data: reactions } = await supabase
    .from("reactions")
    .select("*")
    .eq("user_id", event.user_id)
    .eq("source", event.source)
    .eq("event_type", event.event_type)
    .eq("enabled", true);

  for (const reaction of reactions || []) {
    // Evaluate precondition
    if (!evaluatePrecondition(reaction.precondition, event)) continue;

    // Check cooldown
    if (reaction.cooldown_seconds > 0 && reaction.last_triggered_at) {
      const elapsed = (Date.now() - new Date(reaction.last_triggered_at).getTime()) / 1000;
      if (elapsed < reaction.cooldown_seconds) continue;
    }

    // Render prompt
    const renderedPrompt = renderTemplate(reaction.prompt_template, event);

    // Create execution record
    const { data: execution } = await supabase.from("executions").insert({
      user_id: event.user_id,
      event_id: event.id,
      reaction_id: reaction.id,
      status: "pending",
      agent_type: reaction.agent_type,
      agent_model: reaction.agent_model,
      rendered_prompt: renderedPrompt,
      system_prompt: reaction.system_prompt,
    }).select().single();

    // Dispatch to trigger.dev
    await fetch("https://api.trigger.dev/api/v1/tasks/run-agent/trigger", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${Deno.env.get("TRIGGER_SECRET_KEY")}`,
      },
      body: JSON.stringify({
        payload: {
          executionId: execution.id,
          userId: event.user_id,
          agentType: reaction.agent_type,
          agentModel: reaction.agent_model,
          renderedPrompt,
          systemPrompt: reaction.system_prompt,
          maxTokens: reaction.max_tokens,
        },
      }),
    });

    // Update cooldown
    await supabase
      .from("reactions")
      .update({ last_triggered_at: new Date().toISOString() })
      .eq("id", reaction.id);
  }

  // Mark event as processed
  await supabase
    .from("events")
    .update({ processed: true, processed_at: new Date().toISOString() })
    .eq("id", event.id);

  return new Response("OK");
});

function evaluatePrecondition(precondition: any, event: any): boolean {
  if (!precondition) return true;
  if (precondition.and) return precondition.and.every((s: any) => evaluatePrecondition(s, event));
  if (precondition.or) return precondition.or.some((s: any) => evaluatePrecondition(s, event));
  if (precondition.not) return !evaluatePrecondition(precondition.not, event);

  const actual = precondition.field.split(".").reduce((o: any, k: string) => o?.[k], event);
  switch (precondition.op) {
    case "eq": return actual === precondition.value;
    case "neq": return actual !== precondition.value;
    case "contains": return typeof actual === "string" && actual.includes(precondition.value);
    case "starts_with": return typeof actual === "string" && actual.startsWith(precondition.value);
    case "in": return Array.isArray(precondition.value) && precondition.value.includes(actual);
    case "exists": return actual != null;
    case "gt": return actual > precondition.value;
    case "gte": return actual >= precondition.value;
    case "lt": return actual < precondition.value;
    case "lte": return actual <= precondition.value;
    default: return false;
  }
}

function renderTemplate(template: string, event: any): string {
  return template.replace(/\{\{([^}]+)\}\}/g, (_, path) => {
    const val = path.trim().split(".").reduce((o: any, k: string) => o?.[k], event);
    if (val == null) return "";
    return typeof val === "object" ? JSON.stringify(val, null, 2) : String(val);
  });
}
```

### Deploy Edge Functions

```bash
# Set secrets
supabase secrets set TRIGGER_SECRET_KEY=<your-trigger-dev-secret-key>
supabase secrets set GITHUB_WEBHOOK_SECRET=<your-github-webhook-secret>

# Deploy
supabase functions deploy webhook-github
supabase functions deploy evaluate-event
```

## Step 3: Set Up Trigger.dev

### Initialize

```bash
npm init -y
npm install @trigger.dev/sdk @anthropic-ai/sdk @supabase/supabase-js
npx trigger.dev@latest init
```

### Create the agent task

Create `src/trigger/run-agent.ts` — see [Trigger.dev docs](./trigger-dev.md) for the full implementation.

### Set environment variables

In the Trigger.dev dashboard, set:

- `SUPABASE_URL` = your Supabase project URL
- `SUPABASE_SERVICE_ROLE_KEY` = your service role key

### Deploy

```bash
# Development (runs locally)
npx trigger.dev@latest dev

# Production
npx trigger.dev@latest deploy
```

## Step 4: Store Your API Keys

### Via Supabase SQL Editor (quickest for personal use)

```sql
-- Store your Anthropic API key
SELECT vault.create_secret(
  'sk-ant-api03-YOUR-KEY-HERE',
  'anthropic_key_YOUR-USER-UUID',
  'My Anthropic API key'
);

-- Store your OpenAI API key (if using Codex)
SELECT vault.create_secret(
  'sk-YOUR-OPENAI-KEY-HERE',
  'openai_key_YOUR-USER-UUID',
  'My OpenAI API key'
);
```

Find your user UUID: `SELECT id FROM auth.users WHERE email = 'you@example.com';`

## Step 5: Configure a Webhook Source

### GitHub

1. Go to your repo > Settings > Webhooks > Add webhook
2. Payload URL: `https://<project-ref>.supabase.co/functions/v1/webhook-github`
3. Content type: `application/json`
4. Secret: the same value you set as `GITHUB_WEBHOOK_SECRET`
5. Select events: Push, Pull requests, Issues, Issue comments

### Slack

1. Create a Slack App at https://api.slack.com/apps
2. Enable Event Subscriptions
3. Request URL: `https://<project-ref>.supabase.co/functions/v1/webhook-slack`
4. Subscribe to: `message.channels`, `app_mention`

## Step 6: Create Your First Reaction

```sql
INSERT INTO reactions (user_id, name, source, event_type, precondition, agent_type, agent_model, prompt_template)
VALUES (
  'YOUR-USER-UUID',
  'Review PRs to main',
  'github',
  'pull_request.opened',
  '{"field": "payload.pull_request.base.ref", "op": "eq", "value": "main"}',
  'claude',
  'claude-sonnet-4-6',
  'Review this PR for bugs and security issues.

Title: {{payload.pull_request.title}}
Author: {{payload.pull_request.user.login}}
Description: {{payload.pull_request.body}}
Changed files: {{payload.pull_request.changed_files}}

Provide a concise, structured review.'
);
```

## Step 7: Test It

1. Open a PR to `main` on your GitHub repo
2. GitHub sends a webhook to your Edge Function
3. Check the `events` table — you should see the event
4. Check the `executions` table — you should see a pending/running execution
5. Check the Trigger.dev dashboard — you should see the agent task running
6. When complete, the `executions` table will have the agent's output

## Monitoring

| What | Where |
|------|-------|
| Incoming events | `events` table in Supabase |
| Execution status | `executions` table in Supabase |
| Agent task runs | Trigger.dev dashboard |
| Edge Function logs | Supabase Dashboard > Edge Functions > Logs |
| Errors | `executions.error` column + Trigger.dev failed runs |

## Cost Breakdown (Personal Use)

| Component | Free Tier | Paid |
|-----------|-----------|------|
| Supabase | 500 MB DB, 500K Edge Function invocations | $25/mo Pro |
| Trigger.dev | 10 concurrent runs, $5 compute | $50/mo Pro |
| Anthropic API | Pay per token | ~$1-3/day typical |
| **Total** | **$0 + API tokens** | **~$100/mo** |
