## Context

Reactor is an event-driven AI agent platform. It listens to external events (GitHub commits, PR comments, Slack messages, etc.), evaluates them against user-defined rules, checks preconditions, and runs AI agents (Claude or Codex) when conditions are met. Results are posted back to the source (GitHub comments, Slack replies, etc.). This spec defines everything needed to build it.

---

# Reactor -- Comprehensive Implementation Specification

## Table of Contents

1. Project Setup and Directory Structure
2. Database Layer
3. Shared Libraries
4. Edge Functions (Supabase)
5. Trigger.dev Tasks
6. Output Actions System
7. Next.js Dashboard (shadcn/ui)
8. Testing Strategy
9. Deployment

---

## 1. Project Setup and Directory Structure

### 1.1 Tooling

- **Monorepo**: Turborepo 2.8.x
- **Package Manager**: pnpm (with `workspace:*` protocol)
- **Frontend**: Next.js 16.x (Turbopack default)
- **UI Components**: shadcn/ui (new-york style, Tailwind v4)
- **Language**: TypeScript 5.7+

### 1.2 Repository Layout

```
/Users/criss/work/cue++/reactor/
├── README.md
├── docs/                             # existing documentation
├── package.json                      # root package.json
├── pnpm-workspace.yaml               # pnpm workspace config
├── turbo.json                        # Turborepo pipeline config
├── tsconfig.json                     # root tsconfig (shared base)
├── .env.example                      # env var template
├── .gitignore
│
├── apps/
│   └── web/                          # Next.js 16 dashboard
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       ├── components.json           # shadcn config for app
│       ├── .env.local.example
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx
│       │   │   ├── page.tsx              # redirect to /dashboard
│       │   │   ├── globals.css           # Tailwind v4 base styles
│       │   │   ├── (auth)/
│       │   │   │   ├── login/page.tsx
│       │   │   │   ├── signup/page.tsx
│       │   │   │   └── callback/route.ts     # Supabase Auth callback
│       │   │   └── (app)/
│       │   │       ├── layout.tsx            # authenticated layout with sidebar
│       │   │       ├── dashboard/page.tsx     # overview
│       │   │       ├── events/
│       │   │       │   ├── page.tsx           # list with filters
│       │   │       │   └── [id]/page.tsx      # event detail
│       │   │       ├── reactions/
│       │   │       │   ├── page.tsx           # list
│       │   │       │   ├── new/page.tsx       # create
│       │   │       │   └── [id]/
│       │   │       │       ├── page.tsx       # edit
│       │   │       │       └── executions/page.tsx
│       │   │       ├── executions/
│       │   │       │   ├── page.tsx           # list with realtime
│       │   │       │   └── [id]/page.tsx      # detail
│       │   │       ├── connections/
│       │   │       │   └── page.tsx           # API keys + webhooks
│       │   │       └── settings/
│       │   │           └── page.tsx
│       │   ├── components/               # app-specific composed components
│       │   │   ├── event-list.tsx
│       │   │   ├── event-detail.tsx
│       │   │   ├── reaction-form.tsx
│       │   │   ├── reaction-list.tsx
│       │   │   ├── precondition-builder.tsx
│       │   │   ├── prompt-editor.tsx
│       │   │   ├── output-action-config.tsx
│       │   │   ├── execution-list.tsx
│       │   │   ├── execution-detail.tsx
│       │   │   ├── connection-form.tsx
│       │   │   ├── connection-list.tsx
│       │   │   ├── status-badge.tsx
│       │   │   ├── json-viewer.tsx
│       │   │   ├── sidebar.tsx
│       │   │   └── header.tsx
│       │   ├── lib/
│       │   │   ├── supabase/
│       │   │   │   ├── client.ts         # browser client
│       │   │   │   ├── server.ts         # server client
│       │   │   │   └── middleware.ts
│       │   │   └── hooks/
│       │   │       ├── use-realtime.ts
│       │   │       ├── use-events.ts
│       │   │       ├── use-reactions.ts
│       │   │       ├── use-executions.ts
│       │   │       └── use-connections.ts
│       │   └── middleware.ts             # Next.js auth middleware
│       └── public/
│
├── packages/
│   ├── ui/                           # shared shadcn/ui components
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── components.json           # shadcn config for ui package
│   │   └── src/
│   │       ├── components/ui/        # shadcn primitives (button, card, dialog, etc.)
│   │       ├── lib/
│   │       │   └── utils.ts          # cn() utility
│   │       └── styles/
│   │           └── globals.css       # shared Tailwind theme
│   │
│   ├── shared/                       # shared TypeScript library
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts              # barrel export
│   │       ├── types.ts              # all entity interfaces + enums
│   │       ├── preconditions.ts      # precondition evaluation engine
│   │       ├── templates.ts          # prompt template renderer
│   │       ├── validators/
│   │       │   ├── index.ts
│   │       │   ├── github.ts         # GitHub HMAC-SHA256
│   │       │   ├── slack.ts          # Slack signing secret
│   │       │   ├── linear.ts         # Linear HMAC-SHA256
│   │       │   └── generic.ts        # Bearer token
│   │       ├── supabase.ts           # Supabase client factory
│   │       ├── output-actions.ts     # output action types + helpers
│   │       ├── cost.ts               # token cost estimation
│   │       └── errors.ts             # shared error types
│   │
│   ├── typescript-config/            # shared tsconfig presets
│   │   ├── package.json
│   │   ├── base.json
│   │   ├── nextjs.json
│   │   └── library.json
│   │
│   └── eslint-config/                # shared ESLint config
│       ├── package.json
│       └── base.js
│
├── supabase/
│   ├── config.toml                   # Supabase local dev config
│   ├── seed.sql
│   ├── migrations/
│   │   ├── 00001_create_reactor_tables.sql
│   │   ├── 00002_create_vault_helpers.sql
│   │   └── 00003_create_webhook_user_lookup.sql
│   └── functions/
│       ├── deno.json
│       ├── _shared/
│       │   ├── supabase-client.ts
│       │   ├── cors.ts
│       │   └── respond.ts
│       ├── webhook-github/index.ts
│       ├── webhook-slack/index.ts
│       ├── webhook-linear/index.ts
│       ├── webhook-generic/index.ts
│       ├── evaluate-event/index.ts
│       └── store-secret/index.ts
│
└── apps/
    └── trigger/                      # trigger.dev tasks
        ├── package.json
        ├── tsconfig.json
        ├── trigger.config.ts
        └── src/
            ├── queues.ts
            ├── tasks/
            │   ├── run-agent.ts
            │   ├── run-agent-with-tools.ts
            │   └── execute-output-action.ts
            └── lib/
                ├── vault.ts
                ├── claude.ts
                ├── codex.ts
                └── output/
                    ├── github.ts
                    ├── slack.ts
                    ├── linear.ts
                    └── webhook.ts
```

### 1.3 pnpm-workspace.yaml

**File**: `/Users/criss/work/cue++/reactor/pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### 1.4 turbo.json

**File**: `/Users/criss/work/cue++/reactor/turbo.json`

```json
{
  "$schema": "https://turborepo.dev/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true,
      "interactive": true
    },
    "lint": {
      "outputs": [],
      "dependsOn": ["^build"]
    },
    "test": {
      "outputs": ["coverage/**"],
      "dependsOn": ["build"]
    },
    "typecheck": {
      "outputs": [],
      "dependsOn": ["^build"]
    }
  }
}
```

### 1.5 Root package.json

**File**: `/Users/criss/work/cue++/reactor/package.json`

```json
{
  "name": "reactor",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "dev:web": "turbo dev --filter=@reactor/web",
    "dev:trigger": "turbo dev --filter=@reactor/trigger",
    "dev:supabase": "supabase start",
    "lint": "turbo lint",
    "test": "turbo test",
    "typecheck": "turbo typecheck",
    "deploy:functions": "supabase functions deploy --project-ref $SUPABASE_PROJECT_REF",
    "deploy:trigger": "cd apps/trigger && npx trigger.dev@latest deploy",
    "db:push": "supabase db push",
    "db:reset": "supabase db reset"
  },
  "devDependencies": {
    "turbo": "^2.8.0",
    "typescript": "^5.7.0"
  },
  "packageManager": "pnpm@9.15.0"
}
```

### 1.6 Root tsconfig.json

**File**: `/Users/criss/work/cue++/reactor/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "resolveJsonModule": true,
    "isolatedModules": true
  }
}
```

### 1.7 packages/shared/package.json

**File**: `/Users/criss/work/cue++/reactor/packages/shared/package.json`

```json
{
  "name": "@reactor/shared",
  "version": "0.1.0",
  "private": true,
  "exports": {
    ".": "./src/index.ts",
    "./types": "./src/types.ts",
    "./preconditions": "./src/preconditions.ts",
    "./templates": "./src/templates.ts",
    "./validators": "./src/validators/index.ts",
    "./cost": "./src/cost.ts"
  },
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "vitest": "^3.0.0"
  }
}
```

The shared package has zero runtime dependencies. It exports pure functions and type definitions via the `exports` map. Consumed by trigger tasks and the dashboard via `workspace:*`. Edge Functions inline the needed functions since Deno cannot consume pnpm workspaces directly (see section 4).

### 1.8 packages/ui/package.json

**File**: `/Users/criss/work/cue++/reactor/packages/ui/package.json`

```json
{
  "name": "@reactor/ui",
  "version": "0.1.0",
  "private": true,
  "exports": {
    "./components/*": "./src/components/*.tsx",
    "./lib/*": "./src/lib/*.ts",
    "./styles/*": "./src/styles/*.css"
  },
  "scripts": {
    "build": "tsc",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^3.0.0",
    "lucide-react": "^0.468.0",
    "@radix-ui/react-dialog": "^1.1.0",
    "@radix-ui/react-dropdown-menu": "^2.1.0",
    "@radix-ui/react-select": "^2.1.0",
    "@radix-ui/react-tabs": "^1.1.0",
    "@radix-ui/react-toast": "^1.2.0",
    "@radix-ui/react-label": "^2.1.0",
    "@radix-ui/react-switch": "^1.1.0",
    "@radix-ui/react-separator": "^1.1.0",
    "@radix-ui/react-tooltip": "^1.1.0",
    "@radix-ui/react-popover": "^1.1.0",
    "@radix-ui/react-slot": "^1.1.0"
  },
  "peerDependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "@types/react": "^19.0.0"
  }
}
```

**shadcn/ui setup**: Initialize in the ui package:
```bash
cd packages/ui && pnpm dlx shadcn@latest init
```

Then add components:
```bash
pnpm dlx shadcn@latest add button card dialog dropdown-menu input label select separator switch table tabs textarea toast tooltip badge popover
```

### 1.9 packages/ui/components.json

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/styles/globals.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "iconLibrary": "lucide",
  "aliases": {
    "components": "@reactor/ui/components",
    "utils": "@reactor/ui/lib/utils",
    "ui": "@reactor/ui/components/ui",
    "lib": "@reactor/ui/lib",
    "hooks": "@reactor/ui/hooks"
  }
}
```

### 1.10 apps/trigger/package.json

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/package.json`

```json
{
  "name": "@reactor/trigger",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "npx trigger.dev@latest dev",
    "build": "tsc",
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@trigger.dev/sdk": "^4.0.0",
    "@anthropic-ai/sdk": "^0.39.0",
    "@openai/codex-sdk": "^0.116.0",
    "@supabase/supabase-js": "^2.49.0",
    "@reactor/shared": "workspace:*"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "vitest": "^3.0.0"
  }
}
```

### 1.11 apps/web/package.json

**File**: `/Users/criss/work/cue++/reactor/apps/web/package.json`

```json
{
  "name": "@reactor/web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "start": "next start",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "next": "^16.2.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@supabase/supabase-js": "^2.49.0",
    "@supabase/ssr": "^0.6.0",
    "@reactor/ui": "workspace:*",
    "@reactor/shared": "workspace:*",
    "react-hook-form": "^7.54.0",
    "zod": "^3.24.0",
    "@hookform/resolvers": "^3.10.0",
    "tailwindcss": "^4.0.0",
    "@tailwindcss/postcss": "^4.0.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0"
  }
}
```

### 1.12 apps/web/components.json

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "iconLibrary": "lucide",
  "aliases": {
    "components": "@/components",
    "utils": "@reactor/ui/lib/utils",
    "ui": "@reactor/ui/components/ui",
    "lib": "@/lib",
    "hooks": "@/lib/hooks"
  }
}
```

### 1.13 Environment Variables

**File**: `/Users/criss/work/cue++/reactor/.env.example`

```bash
# Supabase
SUPABASE_URL=https://YOUR_PROJECT_REF.supabase.co
SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...
SUPABASE_PROJECT_REF=YOUR_PROJECT_REF

# Trigger.dev
TRIGGER_SECRET_KEY=tr_dev_...
TRIGGER_API_URL=https://api.trigger.dev

# Dashboard (Next.js)
NEXT_PUBLIC_SUPABASE_URL=https://YOUR_PROJECT_REF.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...

# Webhook secrets (stored in Vault for production, env vars for local dev)
GITHUB_WEBHOOK_SECRET=whsec_...
SLACK_SIGNING_SECRET=...
LINEAR_WEBHOOK_SECRET=...
```

---

## 2. Database Layer

### 2.1 Migration 1: Core Tables

**File**: `/Users/criss/work/cue++/reactor/supabase/migrations/00001_create_reactor_tables.sql`

This is the foundational migration. It creates all four tables, their indexes, and RLS policies. The `output_actions` column on `reactions` (which the existing docs do not include) is added here as a first-class column.

```sql
-- ============================================================
-- Migration: 00001_create_reactor_tables
-- Creates: events, reactions, executions, connections
-- ============================================================

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "supabase_vault" SCHEMA vault;

-- -----------------------------------------------------------
-- EVENTS
-- -----------------------------------------------------------
CREATE TABLE public.events (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  source          text NOT NULL,
  event_type      text NOT NULL,
  source_event_id text,

  payload   jsonb NOT NULL,
  metadata  jsonb DEFAULT '{}'::jsonb,

  processed    boolean DEFAULT false,
  processed_at timestamptz,

  created_at timestamptz DEFAULT now(),

  CONSTRAINT events_source_event_id_unique
    UNIQUE NULLS NOT DISTINCT (source, source_event_id)
);

CREATE INDEX idx_events_user_source
  ON public.events (user_id, source, event_type);
CREATE INDEX idx_events_unprocessed
  ON public.events (user_id, processed) WHERE processed = false;
CREATE INDEX idx_events_created
  ON public.events (created_at DESC);
CREATE INDEX idx_events_source_event_id
  ON public.events (source, source_event_id);

ALTER TABLE public.events ENABLE ROW LEVEL SECURITY;

CREATE POLICY "events_select_own"
  ON public.events FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "events_insert_service"
  ON public.events FOR INSERT
  WITH CHECK (true);

CREATE POLICY "events_update_service"
  ON public.events FOR UPDATE
  USING (true);

-- -----------------------------------------------------------
-- REACTIONS
-- -----------------------------------------------------------
CREATE TABLE public.reactions (
  id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  name       text NOT NULL,
  source     text NOT NULL,
  event_type text NOT NULL,

  precondition jsonb,

  agent_type     text NOT NULL DEFAULT 'claude'
    CHECK (agent_type IN ('claude', 'codex')),
  agent_model    text NOT NULL DEFAULT 'claude-sonnet-4-6',
  prompt_template text NOT NULL,
  system_prompt  text,
  tools          jsonb DEFAULT '[]'::jsonb,
  max_tokens     integer DEFAULT 4096,

  -- Output actions: what to do with the agent's result
  output_actions jsonb DEFAULT '[]'::jsonb,

  enabled          boolean DEFAULT true,
  cooldown_seconds integer DEFAULT 0,
  last_triggered_at timestamptz,

  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

CREATE INDEX idx_reactions_match
  ON public.reactions (user_id, source, event_type, enabled);

ALTER TABLE public.reactions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "reactions_all_own"
  ON public.reactions FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION public.set_updated_at()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;

CREATE TRIGGER reactions_updated_at
  BEFORE UPDATE ON public.reactions
  FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();

-- -----------------------------------------------------------
-- EXECUTIONS
-- -----------------------------------------------------------
CREATE TABLE public.executions (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  event_id    uuid NOT NULL REFERENCES public.events(id) ON DELETE CASCADE,
  reaction_id uuid NOT NULL REFERENCES public.reactions(id) ON DELETE CASCADE,

  status      text NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
  agent_type  text NOT NULL,
  agent_model text NOT NULL,

  rendered_prompt text NOT NULL,
  system_prompt   text,
  agent_output    text,
  tool_calls      jsonb DEFAULT '[]'::jsonb,

  trigger_run_id  text,
  trigger_task_id text,

  -- Output action results
  output_results jsonb DEFAULT '[]'::jsonb,

  started_at   timestamptz,
  completed_at timestamptz,

  error       text,
  retry_count integer DEFAULT 0,

  input_tokens       integer,
  output_tokens      integer,
  estimated_cost_usd numeric(10, 6),

  created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_executions_user_status
  ON public.executions (user_id, status);
CREATE INDEX idx_executions_event
  ON public.executions (event_id);
CREATE INDEX idx_executions_reaction
  ON public.executions (reaction_id);
CREATE INDEX idx_executions_created
  ON public.executions (created_at DESC);
CREATE INDEX idx_executions_trigger_run
  ON public.executions (trigger_run_id);

ALTER TABLE public.executions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "executions_select_own"
  ON public.executions FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "executions_insert_service"
  ON public.executions FOR INSERT
  WITH CHECK (true);

CREATE POLICY "executions_update_service"
  ON public.executions FOR UPDATE
  USING (true);

-- -----------------------------------------------------------
-- CONNECTIONS
-- -----------------------------------------------------------
CREATE TABLE public.connections (
  id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  provider text NOT NULL,
  label    text,

  api_key_secret_id    uuid,
  webhook_secret_id    uuid,

  scopes jsonb DEFAULT '[]'::jsonb,
  config jsonb DEFAULT '{}'::jsonb,

  active           boolean DEFAULT true,
  last_verified_at timestamptz,

  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),

  CONSTRAINT connections_user_provider_label_unique
    UNIQUE (user_id, provider, label)
);

ALTER TABLE public.connections ENABLE ROW LEVEL SECURITY;

CREATE POLICY "connections_all_own"
  ON public.connections FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

CREATE TRIGGER connections_updated_at
  BEFORE UPDATE ON public.connections
  FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();

-- -----------------------------------------------------------
-- Enable Realtime for execution status updates
-- -----------------------------------------------------------
ALTER PUBLICATION supabase_realtime ADD TABLE public.executions;
ALTER PUBLICATION supabase_realtime ADD TABLE public.events;
```

**Key design decisions in this migration**:

- `events.source_event_id` uses `UNIQUE NULLS NOT DISTINCT` so that null values do not bypass the unique constraint when source_event_id is unknown.
- `executions.output_results` (new) stores the results of output action execution (success/failure per action).
- `reactions.output_actions` (new) stores an array of output action configurations.
- The `status` column on executions uses a CHECK constraint rather than an enum to make it easier to add states later without a migration.
- Realtime is enabled on `executions` and `events` for the dashboard.
- ON DELETE CASCADE on `user_id` foreign keys ensures clean user deletion.
- The `set_updated_at()` trigger function is shared between reactions and connections.

### 2.2 Migration 2: Vault Helper Functions

**File**: `/Users/criss/work/cue++/reactor/supabase/migrations/00002_create_vault_helpers.sql`

```sql
-- ============================================================
-- Migration: 00002_create_vault_helpers
-- Postgres functions for Vault secret management
-- ============================================================

-- Create or update a secret in Vault
-- SECURITY DEFINER: runs with the definer's permissions (superuser)
-- so that Edge Functions and Trigger.dev tasks can call it via service role
CREATE OR REPLACE FUNCTION public.create_or_update_secret(
  secret_name text,
  secret_value text,
  secret_description text DEFAULT ''
)
RETURNS uuid
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$
DECLARE
  existing_id uuid;
  result_id uuid;
BEGIN
  SELECT id INTO existing_id
  FROM vault.secrets
  WHERE name = secret_name;

  IF existing_id IS NOT NULL THEN
    UPDATE vault.secrets
    SET secret = secret_value,
        description = secret_description,
        updated_at = now()
    WHERE id = existing_id;
    result_id := existing_id;
  ELSE
    INSERT INTO vault.secrets (name, secret, description)
    VALUES (secret_name, secret_value, secret_description)
    RETURNING id INTO result_id;
  END IF;

  RETURN result_id;
END;
$$;

-- Retrieve a decrypted secret by name
CREATE OR REPLACE FUNCTION public.get_decrypted_secret(secret_name text)
RETURNS TABLE (decrypted_secret text)
LANGUAGE sql
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT decrypted_secret
  FROM vault.decrypted_secrets
  WHERE name = secret_name;
$$;

-- Delete a secret by name
CREATE OR REPLACE FUNCTION public.delete_secret(secret_name text)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$
BEGIN
  DELETE FROM vault.secrets WHERE name = secret_name;
END;
$$;

-- Check if a secret exists (without decrypting)
CREATE OR REPLACE FUNCTION public.secret_exists(secret_name text)
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT EXISTS (SELECT 1 FROM vault.secrets WHERE name = secret_name);
$$;
```

**Notes**:
- `SET search_path = ''` is a security best practice for `SECURITY DEFINER` functions, preventing search path injection.
- `create_or_update_secret` returns the secret UUID so the `connections` table can store it as `api_key_secret_id` or `webhook_secret_id`.

### 2.3 Migration 3: Webhook User Lookup Helper

**File**: `/Users/criss/work/cue++/reactor/supabase/migrations/00003_create_webhook_user_lookup.sql`

When a webhook arrives from GitHub/Slack/etc., the Edge Function needs to determine which user owns that webhook. This function looks up the user based on the connection's webhook_secret_id or provider+config match.

```sql
-- ============================================================
-- Migration: 00003_create_webhook_user_lookup
-- Helper to resolve user_id from webhook context
-- ============================================================

-- Look up user by provider and a config field match
-- Example: find the user whose GitHub connection has repo "owner/repo"
CREATE OR REPLACE FUNCTION public.get_user_for_webhook(
  p_provider text,
  p_config_key text DEFAULT NULL,
  p_config_value text DEFAULT NULL
)
RETURNS uuid
LANGUAGE sql
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT user_id
  FROM public.connections
  WHERE provider = p_provider
    AND active = true
    AND (
      p_config_key IS NULL
      OR config ->> p_config_key = p_config_value
    )
  LIMIT 1;
$$;

-- For single-user setups: get the first (only) user
-- This is a convenience for personal deployments
CREATE OR REPLACE FUNCTION public.get_default_user()
RETURNS uuid
LANGUAGE sql
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT id FROM auth.users LIMIT 1;
$$;
```

### 2.4 Database Webhook Configuration

The database webhook is configured via the Supabase Dashboard (not SQL), but the specification must be precise:

- **Table**: `public.events`
- **Events**: `INSERT`
- **Type**: Supabase Edge Function
- **Function**: `evaluate-event`
- **HTTP Headers**: `Content-Type: application/json`
- **Timeout**: 5000ms

The payload Supabase sends to the Edge Function has this shape (for INSERT):

```typescript
{
  type: "INSERT",
  table: "events",
  schema: "public",
  record: {
    id: "uuid",
    user_id: "uuid",
    source: "github",
    event_type: "pull_request.opened",
    source_event_id: "...",
    payload: { /* raw webhook */ },
    metadata: { /* extracted fields */ },
    processed: false,
    processed_at: null,
    created_at: "2026-03-26T..."
  },
  old_record: null
}
```

### 2.5 Seed Data

**File**: `/Users/criss/work/cue++/reactor/supabase/seed.sql`

```sql
-- ============================================================
-- Seed data for local development
-- Creates a test user and sample reaction
-- ============================================================

-- Note: In local development, create a user via Supabase Auth UI
-- or via the dashboard at http://localhost:54323
-- Then insert seed data referencing that user's ID

-- Sample reaction (replace USER_UUID after creating test user)
-- INSERT INTO public.reactions (user_id, name, source, event_type, precondition, agent_type, agent_model, prompt_template, system_prompt, output_actions)
-- VALUES (
--   'USER_UUID_HERE',
--   'Review PRs to main',
--   'github',
--   'pull_request.opened',
--   '{"and": [{"field": "payload.pull_request.base.ref", "op": "eq", "value": "main"}, {"field": "payload.pull_request.draft", "op": "eq", "value": false}]}'::jsonb,
--   'claude',
--   'claude-sonnet-4-6',
--   E'Review this pull request:\n\nTitle: {{payload.pull_request.title}}\nAuthor: {{payload.pull_request.user.login}}\nDescription: {{payload.pull_request.body}}\nFiles changed: {{payload.pull_request.changed_files}}\n\nFocus on bugs and security issues.',
--   'You are a senior code reviewer. Be concise and focus on substantive issues.',
--   '[{"type": "github_pr_comment", "config": {"repo": "{{metadata.repo}}", "pr_number": "{{metadata.pr_number}}"}}]'::jsonb
-- );

-- Sample event for testing the evaluation pipeline
-- INSERT INTO public.events (user_id, source, event_type, source_event_id, payload, metadata)
-- VALUES (
--   'USER_UUID_HERE',
--   'github',
--   'pull_request.opened',
--   'test-delivery-001',
--   '{"action": "opened", "pull_request": {"title": "Test PR", "body": "Testing", "base": {"ref": "main"}, "head": {"ref": "feature/test"}, "user": {"login": "testuser"}, "draft": false, "changed_files": 3, "additions": 50, "deletions": 10, "diff_url": "https://github.com/test/repo/pull/1.diff", "number": 1}, "repository": {"full_name": "test/repo"}, "sender": {"login": "testuser"}}'::jsonb,
--   '{"repo": "test/repo", "sender": "testuser", "action": "opened", "pr_number": 1}'::jsonb
-- );
```

---

## 3. Shared Libraries

### 3.1 Type Definitions

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/types.ts`

This file defines all entity interfaces, enums, and contracts used across the entire system. These types are the single source of truth.

```typescript
// ============================================================
// Core Entity Types
// ============================================================

export type EventSource = "github" | "slack" | "linear" | "jira" | "generic";

export type AgentType = "claude" | "codex";

export type ExecutionStatus =
  | "pending"
  | "running"
  | "completed"
  | "failed"
  | "cancelled";

export type ConnectionProvider =
  | "github"
  | "slack"
  | "linear"
  | "jira"
  | "anthropic"
  | "openai";

// ============================================================
// Database Row Types
// ============================================================

export interface EventRow {
  id: string;
  user_id: string;
  source: EventSource;
  event_type: string;
  source_event_id: string | null;
  payload: Record<string, unknown>;
  metadata: Record<string, unknown>;
  processed: boolean;
  processed_at: string | null;
  created_at: string;
}

export interface ReactionRow {
  id: string;
  user_id: string;
  name: string;
  source: EventSource;
  event_type: string;
  precondition: PreconditionNode | null;
  agent_type: AgentType;
  agent_model: string;
  prompt_template: string;
  system_prompt: string | null;
  tools: ToolDefinition[];
  max_tokens: number;
  output_actions: OutputActionConfig[];
  enabled: boolean;
  cooldown_seconds: number;
  last_triggered_at: string | null;
  created_at: string;
  updated_at: string;
}

export interface ExecutionRow {
  id: string;
  user_id: string;
  event_id: string;
  reaction_id: string;
  status: ExecutionStatus;
  agent_type: AgentType;
  agent_model: string;
  rendered_prompt: string;
  system_prompt: string | null;
  agent_output: string | null;
  tool_calls: ToolCallRecord[];
  trigger_run_id: string | null;
  trigger_task_id: string | null;
  output_results: OutputActionResult[];
  started_at: string | null;
  completed_at: string | null;
  error: string | null;
  retry_count: number;
  input_tokens: number | null;
  output_tokens: number | null;
  estimated_cost_usd: number | null;
  created_at: string;
}

export interface ConnectionRow {
  id: string;
  user_id: string;
  provider: ConnectionProvider;
  label: string | null;
  api_key_secret_id: string | null;
  webhook_secret_id: string | null;
  scopes: string[];
  config: Record<string, unknown>;
  active: boolean;
  last_verified_at: string | null;
  created_at: string;
  updated_at: string;
}

// ============================================================
// Precondition Types
// ============================================================

export type ComparisonOp =
  | "eq" | "neq"
  | "gt" | "gte" | "lt" | "lte"
  | "contains" | "starts_with" | "ends_with"
  | "in" | "exists" | "regex";

export interface LeafPrecondition {
  field: string;
  op: ComparisonOp;
  value?: unknown;
}

export interface AndPrecondition {
  and: PreconditionNode[];
}

export interface OrPrecondition {
  or: PreconditionNode[];
}

export interface NotPrecondition {
  not: PreconditionNode;
}

export type PreconditionNode =
  | LeafPrecondition
  | AndPrecondition
  | OrPrecondition
  | NotPrecondition;

// ============================================================
// Tool Types
// ============================================================

export interface ToolDefinition {
  name: string;
  description: string;
  input_schema: Record<string, unknown>;
}

export interface ToolCallRecord {
  name: string;
  input: Record<string, unknown>;
  output: string;
  timestamp: string;
}

// ============================================================
// Output Action Types
// ============================================================

export type OutputActionType =
  | "github_pr_comment"
  | "github_issue_comment"
  | "github_add_labels"
  | "slack_reply"
  | "slack_post"
  | "linear_comment"
  | "webhook_callback";

export interface OutputActionConfig {
  type: OutputActionType;
  config: Record<string, string>;
  // config values can contain {{template}} variables
  // that are rendered with the event + execution context
}

export interface OutputActionResult {
  type: OutputActionType;
  success: boolean;
  response?: unknown;
  error?: string;
  executed_at: string;
}

// ============================================================
// Task Payloads (Trigger.dev)
// ============================================================

export interface RunAgentPayload {
  executionId: string;
  userId: string;
  agentType: AgentType;
  agentModel: string;
  renderedPrompt: string;
  systemPrompt?: string;
  tools?: ToolDefinition[];
  maxTokens?: number;
  outputActions?: OutputActionConfig[];
  eventPayload?: Record<string, unknown>;
  eventMetadata?: Record<string, unknown>;
}

export interface ExecuteOutputActionPayload {
  executionId: string;
  userId: string;
  action: OutputActionConfig;
  agentOutput: string;
  eventPayload: Record<string, unknown>;
  eventMetadata: Record<string, unknown>;
  eventSource: EventSource;
  eventType: string;
}

// ============================================================
// Webhook Payload (Database Webhook -> evaluate-event)
// ============================================================

export interface DatabaseWebhookPayload {
  type: "INSERT" | "UPDATE" | "DELETE";
  table: string;
  schema: string;
  record: EventRow;
  old_record: EventRow | null;
}

// ============================================================
// Template Rendering Context
// ============================================================

export interface TemplateContext {
  source: string;
  event_type: string;
  payload: Record<string, unknown>;
  metadata: Record<string, unknown>;
  // Available in output action template rendering:
  agent_output?: string;
  execution_id?: string;
}
```

### 3.2 Precondition Evaluation Engine

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/preconditions.ts`

```typescript
import type { PreconditionNode, LeafPrecondition, ComparisonOp } from "./types";

/**
 * Evaluates a precondition tree against an event object.
 * Returns true if the precondition matches, false otherwise.
 * A null/undefined precondition always matches (no filtering).
 *
 * @param precondition - The precondition tree (or null)
 * @param event - The event object with { source, event_type, payload, metadata }
 * @returns boolean
 */
export function evaluatePrecondition(
  precondition: PreconditionNode | null | undefined,
  event: Record<string, unknown>
): boolean {
  if (precondition === null || precondition === undefined) {
    return true;
  }

  // AND compound
  if ("and" in precondition) {
    return precondition.and.every((sub) => evaluatePrecondition(sub, event));
  }

  // OR compound
  if ("or" in precondition) {
    return precondition.or.some((sub) => evaluatePrecondition(sub, event));
  }

  // NOT compound
  if ("not" in precondition) {
    return !evaluatePrecondition(precondition.not, event);
  }

  // Leaf node: field comparison
  return evaluateLeaf(precondition as LeafPrecondition, event);
}

function evaluateLeaf(
  leaf: LeafPrecondition,
  event: Record<string, unknown>
): boolean {
  const actual = getNestedValue(event, leaf.field);
  const { op, value } = leaf;

  switch (op) {
    case "eq":
      return actual === value;
    case "neq":
      return actual !== value;
    case "gt":
      return typeof actual === "number" && typeof value === "number" && actual > value;
    case "gte":
      return typeof actual === "number" && typeof value === "number" && actual >= value;
    case "lt":
      return typeof actual === "number" && typeof value === "number" && actual < value;
    case "lte":
      return typeof actual === "number" && typeof value === "number" && actual <= value;
    case "contains":
      return typeof actual === "string" && typeof value === "string" && actual.includes(value);
    case "starts_with":
      return typeof actual === "string" && typeof value === "string" && actual.startsWith(value);
    case "ends_with":
      return typeof actual === "string" && typeof value === "string" && actual.endsWith(value);
    case "in":
      return Array.isArray(value) && value.includes(actual);
    case "exists":
      return actual !== undefined && actual !== null;
    case "regex":
      if (typeof actual !== "string" || typeof value !== "string") return false;
      try {
        return new RegExp(value).test(actual);
      } catch {
        return false; // invalid regex
      }
    default:
      return false;
  }
}

/**
 * Resolves a dot-notation path against an object.
 * Example: getNestedValue({a: {b: {c: 1}}}, "a.b.c") => 1
 */
export function getNestedValue(obj: unknown, path: string): unknown {
  return path.split(".").reduce<unknown>((current, key) => {
    if (current === null || current === undefined) return undefined;
    if (typeof current !== "object") return undefined;
    return (current as Record<string, unknown>)[key];
  }, obj);
}
```

**Acceptance criteria**:
- `evaluatePrecondition(null, event)` returns `true`.
- Compound `and`, `or`, `not` work recursively to arbitrary depth.
- All 12 operators (`eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains`, `starts_with`, `ends_with`, `in`, `exists`, `regex`) work correctly.
- Invalid regex strings in `regex` op return `false` rather than throwing.
- Dot-notation traversal handles missing intermediate keys by returning `undefined`.

### 3.3 Prompt Template Renderer

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/templates.ts`

```typescript
import { getNestedValue } from "./preconditions";
import type { TemplateContext } from "./types";

/**
 * Renders a Mustache-style template with {{path}} interpolation.
 * Supports dot-notation access into the context object.
 *
 * - Missing values render as empty string ""
 * - Object values are JSON-stringified with 2-space indent
 * - Array values are JSON-stringified with 2-space indent
 * - All other values are coerced to string via String()
 *
 * @param template - The template string with {{path}} placeholders
 * @param context - The data object to resolve paths against
 * @returns The rendered string
 */
export function renderTemplate(
  template: string,
  context: TemplateContext
): string {
  return template.replace(/\{\{([^}]+)\}\}/g, (_match, rawPath: string) => {
    const path = rawPath.trim();
    const value = getNestedValue(context, path);

    if (value === undefined || value === null) {
      return "";
    }
    if (typeof value === "object") {
      return JSON.stringify(value, null, 2);
    }
    return String(value);
  });
}

/**
 * Extracts all template variable paths from a template string.
 * Useful for the UI to show which variables are available.
 */
export function extractTemplatePaths(template: string): string[] {
  const paths: string[] = [];
  const regex = /\{\{([^}]+)\}\}/g;
  let match;
  while ((match = regex.exec(template)) !== null) {
    paths.push(match[1].trim());
  }
  return [...new Set(paths)];
}
```

### 3.4 Webhook Signature Validators

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/validators/github.ts`

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";

/**
 * Validates a GitHub webhook signature (HMAC-SHA256).
 * @param body - Raw request body as string
 * @param signature - Value of X-Hub-Signature-256 header
 * @param secret - The webhook secret
 * @returns true if valid
 */
export function validateGitHubSignature(
  body: string,
  signature: string | null,
  secret: string
): boolean {
  if (!signature) return false;

  const expected =
    "sha256=" + createHmac("sha256", secret).update(body).digest("hex");

  if (signature.length !== expected.length) return false;

  return timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}
```

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/validators/slack.ts`

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";

/**
 * Validates a Slack webhook signature.
 * Slack uses: v0=HMAC-SHA256(signing_secret, "v0:{timestamp}:{body}")
 *
 * @param body - Raw request body as string
 * @param timestamp - Value of X-Slack-Request-Timestamp header
 * @param signature - Value of X-Slack-Signature header
 * @param signingSecret - The Slack app signing secret
 * @returns true if valid
 */
export function validateSlackSignature(
  body: string,
  timestamp: string | null,
  signature: string | null,
  signingSecret: string
): boolean {
  if (!timestamp || !signature) return false;

  // Reject requests older than 5 minutes to prevent replay attacks
  const ts = parseInt(timestamp, 10);
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - ts) > 300) return false;

  const basestring = `v0:${timestamp}:${body}`;
  const expected =
    "v0=" + createHmac("sha256", signingSecret).update(basestring).digest("hex");

  if (signature.length !== expected.length) return false;

  return timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}
```

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/validators/linear.ts`

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";

/**
 * Validates a Linear webhook signature (HMAC-SHA256).
 * @param body - Raw request body as string
 * @param signature - Value of Linear-Signature header
 * @param secret - The webhook secret
 * @returns true if valid
 */
export function validateLinearSignature(
  body: string,
  signature: string | null,
  secret: string
): boolean {
  if (!signature) return false;

  const expected = createHmac("sha256", secret).update(body).digest("hex");

  if (signature.length !== expected.length) return false;

  return timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}
```

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/validators/generic.ts`

```typescript
/**
 * Validates a Bearer token from the Authorization header.
 * @param authHeader - Value of the Authorization header
 * @param expectedToken - The expected token
 * @returns true if valid
 */
export function validateBearerToken(
  authHeader: string | null,
  expectedToken: string
): boolean {
  if (!authHeader) return false;
  const token = authHeader.replace(/^Bearer\s+/i, "");
  return token === expectedToken;
}
```

**Note on Deno compatibility**: The Edge Functions run on Deno, which has `node:crypto` available via the `node:` prefix. The `createHmac` and `timingSafeEqual` imports work in Deno. However, since the shared package is compiled to ESM for npm workspace consumption (by Trigger.dev), and the Edge Functions need to copy these functions inline (or import from a URL), the validators will be duplicated in `supabase/functions/_shared/validators.ts` as a Deno-compatible version. The shared package version serves as the canonical reference and is used by Trigger.dev tasks.

### 3.5 Supabase Client Factory

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/supabase.ts`

```typescript
import { createClient, type SupabaseClient } from "@supabase/supabase-js";

/**
 * Creates a Supabase client with the service role key.
 * Used by Trigger.dev tasks that need to bypass RLS.
 */
export function createServiceClient(): SupabaseClient {
  const url = process.env.SUPABASE_URL;
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY;

  if (!url || !key) {
    throw new Error(
      "Missing SUPABASE_URL or SUPABASE_SERVICE_ROLE_KEY environment variables"
    );
  }

  return createClient(url, key, {
    auth: { persistSession: false, autoRefreshToken: false },
  });
}
```

### 3.6 Output Action Type Helpers

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/output-actions.ts`

```typescript
import type { OutputActionType, OutputActionConfig } from "./types";

/**
 * Maps output action types to the provider whose API key is needed.
 */
export const OUTPUT_ACTION_PROVIDER: Record<OutputActionType, string> = {
  github_pr_comment: "github",
  github_issue_comment: "github",
  github_add_labels: "github",
  slack_reply: "slack",
  slack_post: "slack",
  linear_comment: "linear",
  webhook_callback: "generic",
};

/**
 * Schema of required config keys per output action type.
 */
export const OUTPUT_ACTION_REQUIRED_CONFIG: Record<OutputActionType, string[]> = {
  github_pr_comment: ["repo", "pr_number"],
  github_issue_comment: ["repo", "issue_number"],
  github_add_labels: ["repo", "issue_number", "labels"],
  slack_reply: ["channel", "thread_ts"],
  slack_post: ["channel"],
  linear_comment: ["issue_id"],
  webhook_callback: ["url"],
};

/**
 * Validates that an output action config has all required fields.
 */
export function validateOutputActionConfig(
  action: OutputActionConfig
): string[] {
  const required = OUTPUT_ACTION_REQUIRED_CONFIG[action.type];
  if (!required) return [`Unknown action type: ${action.type}`];

  const missing = required.filter((key) => !action.config[key]);
  return missing.map((key) => `Missing required config key: ${key}`);
}
```

### 3.7 Cost Estimation

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/cost.ts`

```typescript
/**
 * Estimated cost per million tokens for common models.
 * These are approximate and should be updated as pricing changes.
 */
const COST_PER_MILLION_TOKENS: Record<
  string,
  { input: number; output: number }
> = {
  "claude-sonnet-4-6": { input: 3.0, output: 15.0 },
  "claude-haiku-4-5": { input: 0.80, output: 4.0 },
  "claude-opus-4-6": { input: 15.0, output: 75.0 },
  "gpt-5.3-codex": { input: 2.0, output: 10.0 },
};

export function estimateCost(
  model: string,
  inputTokens: number,
  outputTokens: number
): number {
  const pricing = COST_PER_MILLION_TOKENS[model];
  if (!pricing) return 0;

  return (
    (inputTokens / 1_000_000) * pricing.input +
    (outputTokens / 1_000_000) * pricing.output
  );
}
```

### 3.8 Barrel Export

**File**: `/Users/criss/work/cue++/reactor/packages/shared/src/index.ts`

```typescript
export * from "./types";
export * from "./preconditions";
export * from "./templates";
export * from "./output-actions";
export * from "./cost";
export * from "./supabase";
export * from "./errors";
export { validateGitHubSignature } from "./validators/github";
export { validateSlackSignature } from "./validators/slack";
export { validateLinearSignature } from "./validators/linear";
export { validateBearerToken } from "./validators/generic";
```

---

## 3.A Integration Framework (Extensible Source System)

The integration framework makes it easy to add new event sources via a **provider registry** pattern. Each provider implements a standard interface. The system supports multiple integration methods: webhooks, OAuth-based subscriptions, polling, and WebSocket listeners.

### 3.A.1 Integration Method Types

| Method | Description | Examples |
|--------|-------------|---------|
| `webhook` | External service POSTs to our endpoint | GitHub Webhooks, Slack Events API, Linear Webhooks, Stripe |
| `oauth_webhook` | OAuth app that auto-registers webhooks on user's behalf | GitHub App (auto-subscribes repos), Slack App |
| `polling` | Periodically check an API for new data | Jira (if no webhook access), RSS feeds, email via IMAP |
| `websocket` | Persistent connection to a streaming API | Slack RTM, Discord Gateway |

### 3.A.2 Provider Registry (Database)

Add a `providers` table to the data model to make the system self-describing:

**Migration**: `00004_create_providers.sql`

```sql
CREATE TABLE public.providers (
  id text PRIMARY KEY,               -- 'github', 'slack', 'linear', etc.
  name text NOT NULL,                 -- 'GitHub'
  icon text,                          -- Lucide icon name: 'github'
  description text,

  -- Integration method
  integration_method text NOT NULL DEFAULT 'webhook'
    CHECK (integration_method IN ('webhook', 'oauth_webhook', 'polling', 'websocket')),

  -- Auth requirements for this provider
  auth_type text NOT NULL DEFAULT 'webhook_secret'
    CHECK (auth_type IN ('webhook_secret', 'api_key', 'oauth2', 'none')),

  -- OAuth config (for oauth2 auth_type)
  oauth_config jsonb,
  -- {
  --   "authorize_url": "https://github.com/login/oauth/authorize",
  --   "token_url": "https://github.com/login/oauth/access_token",
  --   "scopes": ["repo", "read:org"],
  --   "client_id_env": "GITHUB_OAUTH_CLIENT_ID",
  --   "client_secret_env": "GITHUB_OAUTH_CLIENT_SECRET"
  -- }

  -- Webhook config
  webhook_config jsonb,
  -- {
  --   "signature_header": "X-Hub-Signature-256",
  --   "signature_algorithm": "hmac-sha256",
  --   "event_header": "X-GitHub-Event",
  --   "delivery_header": "X-GitHub-Delivery"
  -- }

  -- Polling config (for polling method)
  polling_config jsonb,
  -- {
  --   "default_interval_seconds": 300,
  --   "min_interval_seconds": 60
  -- }

  -- Known event types for this provider (for UI autocomplete)
  event_types jsonb DEFAULT '[]'::jsonb,
  -- ["push", "pull_request.opened", "pull_request.closed", "issue.opened", ...]

  -- Known payload fields (for precondition builder autocomplete)
  payload_schema jsonb DEFAULT '{}'::jsonb,
  -- { "push": {"ref": "string", "commits": "array", ...} }

  -- Whether this provider is built-in or user-defined
  built_in boolean DEFAULT false,

  created_at timestamptz DEFAULT now()
);

-- Seed built-in providers
INSERT INTO public.providers (id, name, icon, integration_method, auth_type, webhook_config, event_types, built_in) VALUES
('github', 'GitHub', 'github', 'webhook', 'webhook_secret',
 '{"signature_header": "X-Hub-Signature-256", "signature_algorithm": "hmac-sha256", "event_header": "X-GitHub-Event", "delivery_header": "X-GitHub-Delivery"}'::jsonb,
 '["push", "pull_request.opened", "pull_request.closed", "pull_request.merged", "pull_request_review", "issue_comment.created", "issue.opened", "check_run.completed", "release.published", "workflow_run.completed"]'::jsonb,
 true),

('slack', 'Slack', 'message-square', 'webhook', 'webhook_secret',
 '{"signature_header": "X-Slack-Signature", "signature_algorithm": "slack-v0", "event_header": null, "delivery_header": null}'::jsonb,
 '["message", "app_mention", "reaction_added", "channel_created", "member_joined_channel"]'::jsonb,
 true),

('linear', 'Linear', 'layout-list', 'webhook', 'webhook_secret',
 '{"signature_header": "Linear-Signature", "signature_algorithm": "hmac-sha256", "event_header": null, "delivery_header": null}'::jsonb,
 '["issue.created", "issue.updated", "comment.created", "project.updated"]'::jsonb,
 true),

('jira', 'Jira', 'ticket', 'webhook', 'webhook_secret',
 '{"signature_header": null, "signature_algorithm": "none", "event_header": "X-Atlassian-Webhook-Event", "delivery_header": null}'::jsonb,
 '["jira:issue_created", "jira:issue_updated", "comment_created", "sprint_started"]'::jsonb,
 true),

('generic', 'Generic Webhook', 'webhook', 'webhook', 'api_key',
 '{"signature_header": null, "signature_algorithm": "bearer", "event_header": null, "delivery_header": null}'::jsonb,
 '[]'::jsonb,
 true);
```

### 3.A.3 Provider Interface (TypeScript)

**File**: `packages/shared/src/providers.ts`

```typescript
import type { EventSource } from "./types";

// ============================================================
// Integration methods
// ============================================================

export type IntegrationMethod = "webhook" | "oauth_webhook" | "polling" | "websocket";
export type AuthType = "webhook_secret" | "api_key" | "oauth2" | "none";

// ============================================================
// Provider definition (mirrors the DB providers table)
// ============================================================

export interface ProviderDefinition {
  id: string;
  name: string;
  icon: string;
  description?: string;
  integrationMethod: IntegrationMethod;
  authType: AuthType;
  oauthConfig?: OAuthConfig;
  webhookConfig?: WebhookConfig;
  pollingConfig?: PollingConfig;
  eventTypes: string[];
  payloadSchema: Record<string, Record<string, string>>;
  builtIn: boolean;
}

export interface OAuthConfig {
  authorizeUrl: string;
  tokenUrl: string;
  scopes: string[];
  clientIdEnv: string;
  clientSecretEnv: string;
  // Optional: webhook auto-registration endpoint
  webhookRegistrationUrl?: string;
}

export interface WebhookConfig {
  signatureHeader: string | null;
  signatureAlgorithm: "hmac-sha256" | "slack-v0" | "bearer" | "none";
  eventHeader: string | null;
  deliveryHeader: string | null;
}

export interface PollingConfig {
  defaultIntervalSeconds: number;
  minIntervalSeconds: number;
}

// ============================================================
// Webhook handler interface (each source implements this)
// ============================================================

export interface WebhookHandler {
  /** Validate the incoming request (signature, token, etc.) */
  validate(req: Request, secret: string): Promise<boolean>;

  /** Extract the event type from the request */
  extractEventType(req: Request, payload: Record<string, unknown>): string;

  /** Extract a deduplication ID from the request */
  extractDeliveryId(req: Request): string | null;

  /** Normalize the payload into a standard metadata format */
  extractMetadata(payload: Record<string, unknown>, eventType: string): Record<string, unknown>;

  /** Handle any source-specific quirks (e.g., Slack url_verification) */
  handleSpecialCases?(req: Request, payload: Record<string, unknown>): Response | null;
}

// ============================================================
// Polling handler interface (for polling-based sources)
// ============================================================

export interface PollingHandler {
  /** Fetch new events since the last poll */
  poll(config: Record<string, unknown>, since: Date): Promise<PolledEvent[]>;
}

export interface PolledEvent {
  eventType: string;
  sourceEventId: string;
  payload: Record<string, unknown>;
  metadata: Record<string, unknown>;
  timestamp: Date;
}
```

### 3.A.4 Built-in Webhook Handlers

Each source gets a handler file that implements the `WebhookHandler` interface:

**File**: `packages/shared/src/handlers/github.ts`
**File**: `packages/shared/src/handlers/slack.ts`
**File**: `packages/shared/src/handlers/linear.ts`
**File**: `packages/shared/src/handlers/generic.ts`

These are pure functions (no I/O) that handle validation, event type extraction, and metadata normalization.

### 3.A.5 Generic Webhook Router (Edge Function)

Instead of one Edge Function per source, use a **single webhook router** that dispatches based on the URL path:

**File**: `supabase/functions/webhook/index.ts`

```
URL pattern: /functions/v1/webhook/:provider
```

```typescript
// supabase/functions/webhook/index.ts
Deno.serve(async (req) => {
  const url = new URL(req.url);
  const pathParts = url.pathname.split("/");
  const providerId = pathParts[pathParts.length - 1]; // 'github', 'slack', etc.

  // Look up provider config from DB (cached)
  const provider = await getProvider(providerId);
  if (!provider) return new Response("Unknown provider", { status: 404 });

  // Get the handler for this provider
  const handler = getWebhookHandler(providerId);

  // Get the webhook secret for the user/provider
  const userId = await resolveUserId(providerId, req, provider);
  const secret = await getWebhookSecret(userId, providerId);

  // Handle special cases (e.g., Slack url_verification)
  const specialResponse = handler.handleSpecialCases?.(req, await req.clone().json());
  if (specialResponse) return specialResponse;

  // Validate
  const body = await req.text();
  const valid = await handler.validate(
    new Request(req.url, { headers: req.headers, body }),
    secret
  );
  if (!valid) return new Response("Invalid signature", { status: 401 });

  // Parse and extract
  const payload = JSON.parse(body);
  const eventType = handler.extractEventType(req, payload);
  const deliveryId = handler.extractDeliveryId(req);
  const metadata = handler.extractMetadata(payload, eventType);

  // Insert event
  await supabase.from("events").insert({
    user_id: userId,
    source: providerId,
    event_type: eventType,
    source_event_id: deliveryId,
    payload,
    metadata,
  });

  return new Response("OK", { status: 200 });
});
```

This means instead of deploying `webhook-github`, `webhook-slack`, `webhook-linear` separately, you deploy a single `webhook` function. Adding a new source = adding a handler file + a row in the `providers` table. No new Edge Function deployment needed.

### 3.A.6 OAuth Flow for Sources

For providers that use OAuth (e.g., GitHub App, Slack App), the dashboard needs an OAuth connection flow:

**File**: `apps/web/src/app/api/oauth/[provider]/route.ts` — initiates OAuth
**File**: `apps/web/src/app/api/oauth/[provider]/callback/route.ts` — handles callback

```typescript
// apps/web/src/app/api/oauth/[provider]/route.ts
export async function GET(req: Request, { params }: { params: { provider: string } }) {
  const provider = await getProvider(params.provider);
  if (!provider?.oauthConfig) return new Response("Not an OAuth provider", { status: 400 });

  const state = crypto.randomUUID(); // Store in session/cookie for CSRF protection
  const { authorizeUrl, clientIdEnv, scopes } = provider.oauthConfig;

  const url = new URL(authorizeUrl);
  url.searchParams.set("client_id", process.env[clientIdEnv]!);
  url.searchParams.set("scope", scopes.join(" "));
  url.searchParams.set("state", state);
  url.searchParams.set("redirect_uri",
    `${process.env.NEXT_PUBLIC_APP_URL}/api/oauth/${params.provider}/callback`);

  return Response.redirect(url.toString());
}
```

```typescript
// apps/web/src/app/api/oauth/[provider]/callback/route.ts
export async function GET(req: Request, { params }: { params: { provider: string } }) {
  const url = new URL(req.url);
  const code = url.searchParams.get("code");
  const provider = await getProvider(params.provider);

  // Exchange code for access token
  const tokenResponse = await fetch(provider.oauthConfig!.tokenUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json", Accept: "application/json" },
    body: JSON.stringify({
      client_id: process.env[provider.oauthConfig!.clientIdEnv],
      client_secret: process.env[provider.oauthConfig!.clientSecretEnv],
      code,
    }),
  });
  const { access_token } = await tokenResponse.json();

  // Store token in Vault via store-secret Edge Function
  // Create/update connection record
  // Optionally: auto-register webhooks on the provider

  return Response.redirect("/connections?success=true");
}
```

### 3.A.7 Polling Engine (Trigger.dev Scheduled Tasks)

For providers that use polling, a Trigger.dev scheduled task runs periodically:

**File**: `apps/trigger/src/tasks/poll-source.ts`

```typescript
import { schedules } from "@trigger.dev/sdk";

export const pollSourceTask = schedules.task({
  id: "poll-source",
  run: async (payload) => {
    // Get all active polling connections
    const { data: connections } = await supabase
      .from("connections")
      .select("*, providers!inner(*)")
      .eq("providers.integration_method", "polling")
      .eq("active", true);

    for (const conn of connections || []) {
      const handler = getPollingHandler(conn.provider);
      const events = await handler.poll(conn.config, conn.last_polled_at);

      for (const event of events) {
        await supabase.from("events").upsert({
          user_id: conn.user_id,
          source: conn.provider,
          event_type: event.eventType,
          source_event_id: event.sourceEventId,
          payload: event.payload,
          metadata: event.metadata,
        }, { onConflict: "source,source_event_id" });
      }

      await supabase.from("connections")
        .update({ last_polled_at: new Date().toISOString() })
        .eq("id", conn.id);
    }
  },
});
```

### 3.A.8 Adding a New Provider (Step-by-Step)

To add a new event source (e.g., Discord):

1. **Create handler** — `packages/shared/src/handlers/discord.ts` implementing `WebhookHandler`
2. **Register handler** — add to the handler registry in `packages/shared/src/handlers/index.ts`
3. **Insert provider row** — SQL or via dashboard: provider id, name, icon, integration method, auth type, webhook config, known event types
4. **Done** — the generic webhook router auto-picks it up. No new Edge Function deployment needed.

For OAuth-based providers, also:
5. **Add OAuth config** to the provider row
6. **Set env vars** — `DISCORD_OAUTH_CLIENT_ID`, `DISCORD_OAUTH_CLIENT_SECRET`

For polling-based providers, also:
5. **Create polling handler** — `packages/shared/src/handlers/discord-poll.ts` implementing `PollingHandler`
6. **Set polling interval** in the provider config

### 3.A.9 Updated connections Table

Add fields to support OAuth tokens and polling:

```sql
-- Add to connections table (migration 00004)
ALTER TABLE public.connections
  ADD COLUMN oauth_token_secret_id uuid,     -- Vault ref for OAuth access token
  ADD COLUMN refresh_token_secret_id uuid,   -- Vault ref for OAuth refresh token
  ADD COLUMN token_expires_at timestamptz,    -- When the OAuth token expires
  ADD COLUMN last_polled_at timestamptz,      -- For polling-based connections
  ADD COLUMN polling_interval_seconds integer DEFAULT 300;
```

### 3.A.10 Updated Directory Structure (additions)

```
packages/shared/src/
  handlers/                    # NEW: per-provider webhook/polling handlers
    index.ts                   # handler registry
    github.ts                  # WebhookHandler implementation
    slack.ts                   # WebhookHandler implementation
    linear.ts                  # WebhookHandler implementation
    generic.ts                 # WebhookHandler implementation
  providers.ts                 # NEW: provider types and interfaces

supabase/functions/
  webhook/index.ts             # CHANGED: single generic router (replaces per-source functions)

apps/web/src/app/
  api/oauth/[provider]/        # NEW: OAuth initiation
    route.ts
    callback/route.ts          # NEW: OAuth callback

apps/trigger/src/tasks/
  poll-source.ts               # NEW: polling engine
```

---

## 4. Edge Functions (Supabase)

### 4.1 Shared Edge Function Utilities

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/_shared/supabase-client.ts`

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

export function createServiceSupabase() {
  return createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,
    { auth: { persistSession: false, autoRefreshToken: false } }
  );
}
```

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/_shared/cors.ts`

```typescript
export const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
};

export function handleCors(req: Request): Response | null {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }
  return null;
}
```

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/_shared/respond.ts`

```typescript
import { corsHeaders } from "./cors.ts";

export function jsonResponse(data: unknown, status = 200): Response {
  return new Response(JSON.stringify(data), {
    status,
    headers: { ...corsHeaders, "Content-Type": "application/json" },
  });
}

export function errorResponse(message: string, status = 400): Response {
  return jsonResponse({ error: message }, status);
}
```

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/_shared/validators.ts`

This is the Deno-compatible copy of the validators from the shared package. In Deno, `node:crypto` is available.

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";

export function validateGitHubSignature(
  body: string, signature: string | null, secret: string
): boolean {
  if (!signature) return false;
  const expected = "sha256=" + createHmac("sha256", secret).update(body).digest("hex");
  if (signature.length !== expected.length) return false;
  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}

export function validateSlackSignature(
  body: string, timestamp: string | null, signature: string | null, signingSecret: string
): boolean {
  if (!timestamp || !signature) return false;
  const ts = parseInt(timestamp, 10);
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - ts) > 300) return false;
  const basestring = `v0:${timestamp}:${body}`;
  const expected = "v0=" + createHmac("sha256", signingSecret).update(basestring).digest("hex");
  if (signature.length !== expected.length) return false;
  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}

export function validateLinearSignature(
  body: string, signature: string | null, secret: string
): boolean {
  if (!signature) return false;
  const expected = createHmac("sha256", secret).update(body).digest("hex");
  if (signature.length !== expected.length) return false;
  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/_shared/preconditions.ts`

Deno-compatible inline copy of the precondition evaluator and template renderer.

```typescript
// Duplicated from packages/shared for Deno Edge Functions.
// Kept identical to the source.

export function evaluatePrecondition(precondition: any, event: any): boolean {
  if (precondition === null || precondition === undefined) return true;
  if (precondition.and) return precondition.and.every((s: any) => evaluatePrecondition(s, event));
  if (precondition.or) return precondition.or.some((s: any) => evaluatePrecondition(s, event));
  if (precondition.not) return !evaluatePrecondition(precondition.not, event);

  const actual = getNestedValue(event, precondition.field);
  const { op, value } = precondition;

  switch (op) {
    case "eq": return actual === value;
    case "neq": return actual !== value;
    case "gt": return typeof actual === "number" && actual > value;
    case "gte": return typeof actual === "number" && actual >= value;
    case "lt": return typeof actual === "number" && actual < value;
    case "lte": return typeof actual === "number" && actual <= value;
    case "contains": return typeof actual === "string" && actual.includes(value);
    case "starts_with": return typeof actual === "string" && actual.startsWith(value);
    case "ends_with": return typeof actual === "string" && actual.endsWith(value);
    case "in": return Array.isArray(value) && value.includes(actual);
    case "exists": return actual !== undefined && actual !== null;
    case "regex":
      if (typeof actual !== "string" || typeof value !== "string") return false;
      try { return new RegExp(value).test(actual); } catch { return false; }
    default: return false;
  }
}

export function renderTemplate(template: string, context: any): string {
  return template.replace(/\{\{([^}]+)\}\}/g, (_, rawPath: string) => {
    const value = getNestedValue(context, rawPath.trim());
    if (value === undefined || value === null) return "";
    if (typeof value === "object") return JSON.stringify(value, null, 2);
    return String(value);
  });
}

export function getNestedValue(obj: any, path: string): any {
  return path.split(".").reduce((current, key) => current?.[key], obj);
}
```

### 4.2 Edge Function: webhook-github

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/webhook-github/index.ts`

```typescript
import { createServiceSupabase } from "../_shared/supabase-client.ts";
import { validateGitHubSignature } from "../_shared/validators.ts";
import { jsonResponse, errorResponse } from "../_shared/respond.ts";
import { handleCors } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  const corsResp = handleCors(req);
  if (corsResp) return corsResp;

  if (req.method !== "POST") {
    return errorResponse("Method not allowed", 405);
  }

  const body = await req.text();
  const signature = req.headers.get("x-hub-signature-256");
  const event = req.headers.get("x-github-event");
  const deliveryId = req.headers.get("x-github-delivery");

  if (!event || !deliveryId) {
    return errorResponse("Missing required GitHub headers", 400);
  }

  // Validate signature
  const secret = Deno.env.get("GITHUB_WEBHOOK_SECRET");
  if (!secret) {
    console.error("GITHUB_WEBHOOK_SECRET not configured");
    return errorResponse("Server configuration error", 500);
  }

  if (!validateGitHubSignature(body, signature, secret)) {
    return errorResponse("Invalid signature", 401);
  }

  const payload = JSON.parse(body);
  const action = payload.action;

  // Compute event_type: "push", "pull_request.opened", "issue.opened", etc.
  let eventType: string;
  if (event === "pull_request" && action === "closed" && payload.pull_request?.merged) {
    eventType = "pull_request.merged";
  } else if (event === "check_run" && payload.check_run?.conclusion === "failure") {
    eventType = "check_run.failed";
  } else {
    eventType = action ? `${event}.${action}` : event;
  }

  // Determine user_id
  const supabase = createServiceSupabase();
  const repoFullName = payload.repository?.full_name;

  const { data: userId } = await supabase.rpc("get_user_for_webhook", {
    p_provider: "github",
    p_config_key: "repo",
    p_config_value: repoFullName,
  });

  // Fallback to default user for single-user setups
  const resolvedUserId = userId || (await supabase.rpc("get_default_user")).data;

  if (!resolvedUserId) {
    console.error("No user found for GitHub webhook");
    return errorResponse("No user configured", 404);
  }

  // Extract metadata
  const metadata: Record<string, unknown> = {
    repo: repoFullName,
    sender: payload.sender?.login,
    action,
  };

  if (payload.pull_request) {
    metadata.pr_number = payload.pull_request.number;
    metadata.branch = payload.pull_request.head?.ref;
    metadata.base_branch = payload.pull_request.base?.ref;
  }
  if (payload.issue) {
    metadata.issue_number = payload.issue.number;
  }
  if (event === "push") {
    metadata.branch = payload.ref?.replace("refs/heads/", "");
    metadata.commit_sha = payload.after;
  }

  // Insert event (with dedup via source_event_id)
  const { error: insertError } = await supabase.from("events").insert({
    user_id: resolvedUserId,
    source: "github",
    event_type: eventType,
    source_event_id: deliveryId,
    payload,
    metadata,
  });

  if (insertError) {
    // Unique constraint violation = duplicate delivery, ignore
    if (insertError.code === "23505") {
      return jsonResponse({ status: "duplicate", delivery_id: deliveryId });
    }
    console.error("Failed to insert event:", insertError);
    return errorResponse("Failed to store event", 500);
  }

  return jsonResponse({ status: "ok", delivery_id: deliveryId });
});
```

**Acceptance criteria**:
- Rejects requests without valid HMAC-SHA256 signature with 401.
- Handles `ping` event (GitHub sends a ping when webhook is first configured) gracefully.
- Normalizes `pull_request.closed` + `merged: true` into `pull_request.merged`.
- Normalizes `check_run` with `conclusion: failure` into `check_run.failed`.
- Deduplicates by `deliveryId` and returns 200 for duplicates.
- Extracts meaningful metadata fields (repo, sender, PR number, branch).

### 4.3 Edge Function: webhook-slack

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/webhook-slack/index.ts`

```typescript
import { createServiceSupabase } from "../_shared/supabase-client.ts";
import { validateSlackSignature } from "../_shared/validators.ts";
import { jsonResponse, errorResponse } from "../_shared/respond.ts";
import { handleCors } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  const corsResp = handleCors(req);
  if (corsResp) return corsResp;

  if (req.method !== "POST") {
    return errorResponse("Method not allowed", 405);
  }

  const body = await req.text();
  const payload = JSON.parse(body);

  // Handle Slack URL verification challenge
  if (payload.type === "url_verification") {
    return jsonResponse({ challenge: payload.challenge });
  }

  // Validate signature
  const timestamp = req.headers.get("x-slack-request-timestamp");
  const signature = req.headers.get("x-slack-signature");
  const signingSecret = Deno.env.get("SLACK_SIGNING_SECRET");

  if (!signingSecret) {
    console.error("SLACK_SIGNING_SECRET not configured");
    return errorResponse("Server configuration error", 500);
  }

  if (!validateSlackSignature(body, timestamp, signature, signingSecret)) {
    return errorResponse("Invalid signature", 401);
  }

  // Slack wraps event data in an "event" field
  const slackEvent = payload.event;
  if (!slackEvent) {
    return jsonResponse({ status: "no_event" });
  }

  // Skip bot messages to prevent infinite loops
  if (slackEvent.bot_id || slackEvent.subtype === "bot_message") {
    return jsonResponse({ status: "bot_message_skipped" });
  }

  const eventType = slackEvent.type; // "message", "app_mention", etc.

  // Build dedup key from team_id + event_ts + channel
  const sourceEventId = `${payload.team_id}-${slackEvent.event_ts || slackEvent.ts}-${slackEvent.channel}`;

  const supabase = createServiceSupabase();

  // Resolve user
  const { data: userId } = await supabase.rpc("get_user_for_webhook", {
    p_provider: "slack",
    p_config_key: "team_id",
    p_config_value: payload.team_id,
  });
  const resolvedUserId = userId || (await supabase.rpc("get_default_user")).data;

  if (!resolvedUserId) {
    return errorResponse("No user configured", 404);
  }

  const metadata: Record<string, unknown> = {
    channel: slackEvent.channel,
    user: slackEvent.user,
    thread_ts: slackEvent.thread_ts || slackEvent.ts,
    text_preview: typeof slackEvent.text === "string"
      ? slackEvent.text.substring(0, 200)
      : "",
    team_id: payload.team_id,
  };

  const { error: insertError } = await supabase.from("events").insert({
    user_id: resolvedUserId,
    source: "slack",
    event_type: eventType,
    source_event_id: sourceEventId,
    payload,
    metadata,
  });

  if (insertError) {
    if (insertError.code === "23505") {
      return jsonResponse({ status: "duplicate" });
    }
    console.error("Failed to insert event:", insertError);
    return errorResponse("Failed to store event", 500);
  }

  return jsonResponse({ status: "ok" });
});
```

**Acceptance criteria**:
- Responds to `url_verification` challenge immediately (Slack requires this within 3 seconds).
- Validates signatures with timestamp-based replay protection (5-minute window).
- Skips bot messages (prevents infinite loops if the bot's own messages trigger webhooks).
- Extracts channel, user, thread_ts, text_preview into metadata.

### 4.4 Edge Function: webhook-linear

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/webhook-linear/index.ts`

Follows the same pattern as GitHub. Linear webhooks send a JSON body with fields `action`, `type`, `data`, `url`, and `webhookId`.

```typescript
import { createServiceSupabase } from "../_shared/supabase-client.ts";
import { validateLinearSignature } from "../_shared/validators.ts";
import { jsonResponse, errorResponse } from "../_shared/respond.ts";
import { handleCors } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  const corsResp = handleCors(req);
  if (corsResp) return corsResp;
  if (req.method !== "POST") return errorResponse("Method not allowed", 405);

  const body = await req.text();
  const signature = req.headers.get("linear-signature");
  const secret = Deno.env.get("LINEAR_WEBHOOK_SECRET");

  if (!secret) return errorResponse("Server configuration error", 500);
  if (!validateLinearSignature(body, signature, secret)) {
    return errorResponse("Invalid signature", 401);
  }

  const payload = JSON.parse(body);
  const { action, type, data } = payload;

  // Linear event type: "issue.created", "comment.created", etc.
  const eventType = `${type}.${action}`;
  const sourceEventId = payload.webhookId || `${type}-${data?.id}-${action}`;

  const supabase = createServiceSupabase();
  const { data: userId } = await supabase.rpc("get_user_for_webhook", {
    p_provider: "linear",
  });
  const resolvedUserId = userId || (await supabase.rpc("get_default_user")).data;
  if (!resolvedUserId) return errorResponse("No user configured", 404);

  const metadata: Record<string, unknown> = {
    team: data?.team?.name || data?.team?.key,
    assignee: data?.assignee?.name,
    state: data?.state?.name,
    priority: data?.priority,
    issue_identifier: data?.identifier,
  };

  const { error: insertError } = await supabase.from("events").insert({
    user_id: resolvedUserId,
    source: "linear",
    event_type: eventType,
    source_event_id: sourceEventId,
    payload,
    metadata,
  });

  if (insertError && insertError.code !== "23505") {
    console.error("Failed to insert event:", insertError);
    return errorResponse("Failed to store event", 500);
  }

  return jsonResponse({ status: "ok" });
});
```

### 4.5 Edge Function: webhook-generic

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/webhook-generic/index.ts`

```typescript
import { createServiceSupabase } from "../_shared/supabase-client.ts";
import { jsonResponse, errorResponse } from "../_shared/respond.ts";
import { handleCors } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  const corsResp = handleCors(req);
  if (corsResp) return corsResp;
  if (req.method !== "POST") return errorResponse("Method not allowed", 405);

  // Bearer token authentication
  const authHeader = req.headers.get("authorization");
  const expectedToken = Deno.env.get("GENERIC_WEBHOOK_TOKEN");
  if (!expectedToken) return errorResponse("Server configuration error", 500);

  const token = authHeader?.replace(/^Bearer\s+/i, "");
  if (token !== expectedToken) return errorResponse("Unauthorized", 401);

  const body = await req.json();
  const { source, event_type, payload, source_event_id } = body;

  if (!source || !event_type || !payload) {
    return errorResponse("Missing required fields: source, event_type, payload", 400);
  }

  const supabase = createServiceSupabase();
  const { data: userId } = await supabase.rpc("get_default_user");
  if (!userId) return errorResponse("No user configured", 404);

  const { error: insertError } = await supabase.from("events").insert({
    user_id: userId,
    source,
    event_type,
    source_event_id: source_event_id || null,
    payload,
    metadata: body.metadata || {},
  });

  if (insertError && insertError.code !== "23505") {
    return errorResponse("Failed to store event", 500);
  }

  return jsonResponse({ status: "ok" });
});
```

### 4.6 Edge Function: evaluate-event

This is the core of the system. It is invoked by the database webhook when a new event is inserted.

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/evaluate-event/index.ts`

```typescript
import { createServiceSupabase } from "../_shared/supabase-client.ts";
import { evaluatePrecondition, renderTemplate } from "../_shared/preconditions.ts";
import { jsonResponse, errorResponse } from "../_shared/respond.ts";

const TRIGGER_API_URL = Deno.env.get("TRIGGER_API_URL") || "https://api.trigger.dev";
const TRIGGER_SECRET_KEY = Deno.env.get("TRIGGER_SECRET_KEY");

Deno.serve(async (req) => {
  if (req.method !== "POST") {
    return errorResponse("Method not allowed", 405);
  }

  // Parse the database webhook payload
  const webhookPayload = await req.json();
  const event = webhookPayload.record;

  if (!event || !event.id) {
    return errorResponse("Invalid webhook payload", 400);
  }

  // Already processed? Skip.
  if (event.processed) {
    return jsonResponse({ status: "already_processed" });
  }

  const supabase = createServiceSupabase();

  // Find matching reactions
  const { data: reactions, error: reactionsError } = await supabase
    .from("reactions")
    .select("*")
    .eq("user_id", event.user_id)
    .eq("source", event.source)
    .eq("event_type", event.event_type)
    .eq("enabled", true);

  if (reactionsError) {
    console.error("Failed to query reactions:", reactionsError);
    return errorResponse("Failed to query reactions", 500);
  }

  const results: Array<{ reactionId: string; status: string }> = [];

  for (const reaction of reactions || []) {
    // 1. Evaluate precondition
    const eventContext = {
      source: event.source,
      event_type: event.event_type,
      payload: event.payload,
      metadata: event.metadata,
    };

    if (!evaluatePrecondition(reaction.precondition, eventContext)) {
      results.push({ reactionId: reaction.id, status: "precondition_failed" });
      continue;
    }

    // 2. Check cooldown
    if (reaction.cooldown_seconds > 0 && reaction.last_triggered_at) {
      const elapsed =
        (Date.now() - new Date(reaction.last_triggered_at).getTime()) / 1000;
      if (elapsed < reaction.cooldown_seconds) {
        results.push({ reactionId: reaction.id, status: "cooldown_active" });
        continue;
      }
    }

    // 3. Render prompt template
    const renderedPrompt = renderTemplate(reaction.prompt_template, eventContext);

    // 4. Create execution record
    const { data: execution, error: execError } = await supabase
      .from("executions")
      .insert({
        user_id: event.user_id,
        event_id: event.id,
        reaction_id: reaction.id,
        status: "pending",
        agent_type: reaction.agent_type,
        agent_model: reaction.agent_model,
        rendered_prompt: renderedPrompt,
        system_prompt: reaction.system_prompt,
      })
      .select()
      .single();

    if (execError) {
      console.error("Failed to create execution:", execError);
      results.push({ reactionId: reaction.id, status: "execution_create_failed" });
      continue;
    }

    // 5. Dispatch to Trigger.dev via HTTP API
    if (!TRIGGER_SECRET_KEY) {
      console.error("TRIGGER_SECRET_KEY not configured");
      results.push({ reactionId: reaction.id, status: "trigger_not_configured" });
      continue;
    }

    // Determine task ID based on whether tools are configured
    const taskId = reaction.tools && reaction.tools.length > 0
      ? "run-agent-with-tools"
      : "run-agent";

    try {
      const triggerResponse = await fetch(
        `${TRIGGER_API_URL}/api/v1/tasks/${taskId}/trigger`,
        {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${TRIGGER_SECRET_KEY}`,
          },
          body: JSON.stringify({
            payload: {
              executionId: execution.id,
              userId: event.user_id,
              agentType: reaction.agent_type,
              agentModel: reaction.agent_model,
              renderedPrompt,
              systemPrompt: reaction.system_prompt,
              tools: reaction.tools,
              maxTokens: reaction.max_tokens,
              outputActions: reaction.output_actions,
              eventPayload: event.payload,
              eventMetadata: event.metadata,
            },
          }),
        }
      );

      const triggerResult = await triggerResponse.json();

      // Update execution with Trigger.dev run ID
      await supabase
        .from("executions")
        .update({
          trigger_run_id: triggerResult.id,
          trigger_task_id: taskId,
        })
        .eq("id", execution.id);

      results.push({ reactionId: reaction.id, status: "dispatched" });
    } catch (triggerError) {
      console.error("Failed to dispatch to Trigger.dev:", triggerError);
      await supabase
        .from("executions")
        .update({
          status: "failed",
          error: `Trigger dispatch failed: ${triggerError}`,
          completed_at: new Date().toISOString(),
        })
        .eq("id", execution.id);
      results.push({ reactionId: reaction.id, status: "dispatch_failed" });
    }

    // 6. Update cooldown timestamp
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

  return jsonResponse({ status: "ok", results });
});
```

**Acceptance criteria**:
- Handles the database webhook payload format correctly (`{ type, table, schema, record, old_record }`).
- Queries reactions filtered by `user_id`, `source`, `event_type`, and `enabled = true`.
- Evaluates preconditions recursively.
- Respects cooldown_seconds.
- Renders prompt templates with full event context.
- Creates execution records before dispatching.
- Dispatches to the correct Trigger.dev task (with or without tools).
- Records the Trigger.dev run ID on the execution.
- Marks the event as processed after all reactions are evaluated.
- Gracefully handles errors at each step without crashing.

### 4.7 Edge Function: store-secret

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/store-secret/index.ts`

```typescript
import { createServiceSupabase } from "../_shared/supabase-client.ts";
import { jsonResponse, errorResponse } from "../_shared/respond.ts";
import { handleCors } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  const corsResp = handleCors(req);
  if (corsResp) return corsResp;
  if (req.method !== "POST") return errorResponse("Method not allowed", 405);

  const supabase = createServiceSupabase();

  // Authenticate the user
  const authHeader = req.headers.get("authorization");
  const token = authHeader?.replace("Bearer ", "");
  const {
    data: { user },
    error: authError,
  } = await supabase.auth.getUser(token);

  if (authError || !user) {
    return errorResponse("Unauthorized", 401);
  }

  const { provider, apiKey, label } = await req.json();

  if (!provider || !apiKey) {
    return errorResponse("Missing required fields: provider, apiKey", 400);
  }

  // Validate the API key before storing
  const validationError = await validateApiKey(provider, apiKey);
  if (validationError) {
    return errorResponse(validationError, 400);
  }

  // Store in Vault
  const secretName = `${provider}_key_${user.id}`;
  const { data: secretId } = await supabase.rpc("create_or_update_secret", {
    secret_name: secretName,
    secret_value: apiKey,
    secret_description: `${provider} API key for user ${user.id}`,
  });

  // Upsert the connection record
  const { error: connError } = await supabase.from("connections").upsert(
    {
      user_id: user.id,
      provider,
      label: label || `${provider}-default`,
      api_key_secret_id: secretId,
      active: true,
      last_verified_at: new Date().toISOString(),
    },
    { onConflict: "user_id,provider,label" }
  );

  if (connError) {
    console.error("Failed to upsert connection:", connError);
    return errorResponse("Failed to save connection", 500);
  }

  return jsonResponse({ success: true, provider });
});

async function validateApiKey(
  provider: string,
  apiKey: string
): Promise<string | null> {
  try {
    if (provider === "anthropic") {
      const resp = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: {
          "x-api-key": apiKey,
          "anthropic-version": "2023-06-01",
          "content-type": "application/json",
        },
        body: JSON.stringify({
          model: "claude-haiku-4-5",
          max_tokens: 1,
          messages: [{ role: "user", content: "test" }],
        }),
      });
      if (!resp.ok && resp.status !== 429) {
        return "Invalid Anthropic API key";
      }
    } else if (provider === "openai") {
      const resp = await fetch("https://api.openai.com/v1/models", {
        headers: { Authorization: `Bearer ${apiKey}` },
      });
      if (!resp.ok && resp.status !== 429) {
        return "Invalid OpenAI API key";
      }
    }
    // GitHub, Slack, Linear keys are not validated on storage
    // (they are webhook secrets, not API keys)
    return null;
  } catch (error) {
    return `Key validation failed: ${error}`;
  }
}
```

### 4.8 Edge Function Deno Configuration

**File**: `/Users/criss/work/cue++/reactor/supabase/functions/deno.json`

```json
{
  "compilerOptions": {
    "strict": true
  },
  "imports": {
    "@supabase/supabase-js": "https://esm.sh/@supabase/supabase-js@2"
  }
}
```

---

## 5. Trigger.dev Tasks

### 5.1 Queue Definitions

In Trigger.dev v4, queues must be defined upfront (not inline in task definitions). This is a new requirement vs v3.

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/queues.ts`

```typescript
import { queue } from "@trigger.dev/sdk";

export const agentRunQueue = queue({
  name: "agent-runs",
  concurrencyLimit: 5,
  releaseConcurrencyOnWaitpoint: true,
});

export const agentToolQueue = queue({
  name: "agent-tool-runs",
  concurrencyLimit: 3,
  releaseConcurrencyOnWaitpoint: true,
});

export const outputActionQueue = queue({
  name: "output-actions",
  concurrencyLimit: 10,
});
```

### 5.2 Trigger.dev Config

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/trigger.config.ts`

```typescript
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  project: process.env.TRIGGER_PROJECT_REF || "reactor",
  runtime: "node",
  machine: "small-1x",
  logLevel: "info",
  dirs: ["src/tasks", "src/queues"],
  retries: {
    enabledInDev: false,
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 30000,
      factor: 2,
    },
  },
});
```

### 5.3 Vault Helper (Trigger.dev Side)

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/lib/vault.ts`

```typescript
import { createServiceClient } from "@reactor/shared";

/**
 * Retrieves a decrypted secret from Supabase Vault.
 * @param secretName - The secret name (e.g., "anthropic_key_{userId}")
 * @returns The decrypted secret string
 * @throws If the secret is not found
 */
export async function getSecret(secretName: string): Promise<string> {
  const supabase = createServiceClient();
  const { data, error } = await supabase.rpc("get_decrypted_secret", {
    secret_name: secretName,
  });

  if (error) {
    throw new Error(`Failed to retrieve secret "${secretName}": ${error.message}`);
  }

  if (!data || data.length === 0) {
    throw new Error(`Secret "${secretName}" not found in Vault`);
  }

  return data[0].decrypted_secret;
}
```

### 5.4 Claude SDK Wrapper

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/lib/claude.ts`

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { getSecret } from "./vault";
import type { ToolDefinition } from "@reactor/shared";

export async function createAnthropicClient(userId: string): Promise<Anthropic> {
  const apiKey = await getSecret(`anthropic_key_${userId}`);
  return new Anthropic({ apiKey });
}

export interface ClaudeRunResult {
  output: string;
  inputTokens: number;
  outputTokens: number;
  toolCalls: Array<{ name: string; input: unknown; output: string; timestamp: string }>;
}

/**
 * Runs a simple Claude completion (no tool calling).
 */
export async function runClaude(params: {
  client: Anthropic;
  model: string;
  prompt: string;
  systemPrompt?: string;
  maxTokens?: number;
}): Promise<ClaudeRunResult> {
  const response = await params.client.messages.create({
    model: params.model,
    max_tokens: params.maxTokens || 4096,
    system: params.systemPrompt || undefined,
    messages: [{ role: "user", content: params.prompt }],
  });

  const output = response.content
    .filter((b) => b.type === "text")
    .map((b) => b.text)
    .join("\n");

  return {
    output,
    inputTokens: response.usage.input_tokens,
    outputTokens: response.usage.output_tokens,
    toolCalls: [],
  };
}

/**
 * Runs an agentic loop with tool calling.
 * Continues until the model stops calling tools or maxIterations is reached.
 */
export async function runClaudeWithTools(params: {
  client: Anthropic;
  model: string;
  prompt: string;
  systemPrompt?: string;
  tools: Anthropic.Tool[];
  maxTokens?: number;
  maxIterations?: number;
  onToolCall?: (name: string, input: unknown) => Promise<string>;
}): Promise<ClaudeRunResult> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: params.prompt },
  ];

  let totalInputTokens = 0;
  let totalOutputTokens = 0;
  const allToolCalls: ClaudeRunResult["toolCalls"] = [];
  const maxIterations = params.maxIterations || 20;
  let iterations = 0;

  while (iterations < maxIterations) {
    const response = await params.client.messages.create({
      model: params.model,
      max_tokens: params.maxTokens || 4096,
      system: params.systemPrompt || undefined,
      tools: params.tools,
      messages,
    });

    totalInputTokens += response.usage.input_tokens;
    totalOutputTokens += response.usage.output_tokens;

    // If no tool use, extract final text and return
    if (response.stop_reason === "end_turn") {
      const output = response.content
        .filter((b) => b.type === "text")
        .map((b) => b.text)
        .join("\n");

      return {
        output,
        inputTokens: totalInputTokens,
        outputTokens: totalOutputTokens,
        toolCalls: allToolCalls,
      };
    }

    // Process tool calls
    messages.push({ role: "assistant", content: response.content });
    const toolResults: Anthropic.ToolResultBlockParam[] = [];

    for (const block of response.content) {
      if (block.type === "tool_use") {
        let result: string;
        try {
          result = params.onToolCall
            ? await params.onToolCall(block.name, block.input)
            : `Tool "${block.name}" not implemented`;
        } catch (err) {
          result = `Tool error: ${err}`;
        }

        allToolCalls.push({
          name: block.name,
          input: block.input as Record<string, unknown>,
          output: result,
          timestamp: new Date().toISOString(),
        });

        toolResults.push({
          type: "tool_result",
          tool_use_id: block.id,
          content: result,
        });
      }
    }

    messages.push({ role: "user", content: toolResults });
    iterations++;
  }

  return {
    output: "[Max tool iterations reached]",
    inputTokens: totalInputTokens,
    outputTokens: totalOutputTokens,
    toolCalls: allToolCalls,
  };
}
```

### 5.5 Codex SDK Wrapper

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/lib/codex.ts`

```typescript
import { getSecret } from "./vault";

export interface CodexRunResult {
  output: string;
  inputTokens: number;
  outputTokens: number;
}

/**
 * Runs a Codex agent via the OpenAI Codex SDK.
 */
export async function runCodex(params: {
  userId: string;
  prompt: string;
}): Promise<CodexRunResult> {
  const apiKey = await getSecret(`openai_key_${params.userId}`);

  // Dynamic import to avoid loading Codex SDK if not needed
  const { Codex } = await import("@openai/codex-sdk");
  const codex = new Codex({ apiKey });
  const thread = codex.startThread();
  const result = await thread.run(params.prompt);

  return {
    output: result.output || "",
    inputTokens: result.usage?.input_tokens || 0,
    outputTokens: result.usage?.output_tokens || 0,
  };
}
```

### 5.6 Task: run-agent

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/tasks/run-agent.ts`

```typescript
import { task } from "@trigger.dev/sdk";
import { createServiceClient, estimateCost } from "@reactor/shared";
import type { RunAgentPayload } from "@reactor/shared";
import { createAnthropicClient, runClaude } from "../lib/claude";
import { runCodex } from "../lib/codex";
import { agentRunQueue } from "../queues";

export const runAgentTask = task({
  id: "run-agent",
  queue: agentRunQueue,
  machine: { preset: "small-1x" },
  run: async (payload: RunAgentPayload) => {
    const {
      executionId, userId, agentType, agentModel,
      renderedPrompt, systemPrompt, maxTokens,
      outputActions, eventPayload, eventMetadata,
    } = payload;

    const supabase = createServiceClient();

    // Update status to running
    await supabase
      .from("executions")
      .update({ status: "running", started_at: new Date().toISOString() })
      .eq("id", executionId);

    try {
      let output: string;
      let inputTokens = 0;
      let outputTokens = 0;

      if (agentType === "claude") {
        const client = await createAnthropicClient(userId);
        const result = await runClaude({
          client,
          model: agentModel,
          prompt: renderedPrompt,
          systemPrompt: systemPrompt || undefined,
          maxTokens: maxTokens || 4096,
        });
        output = result.output;
        inputTokens = result.inputTokens;
        outputTokens = result.outputTokens;
      } else {
        const result = await runCodex({ userId, prompt: renderedPrompt });
        output = result.output;
        inputTokens = result.inputTokens;
        outputTokens = result.outputTokens;
      }

      const costUsd = estimateCost(agentModel, inputTokens, outputTokens);

      // Update execution with results
      await supabase
        .from("executions")
        .update({
          status: "completed",
          agent_output: output,
          completed_at: new Date().toISOString(),
          input_tokens: inputTokens,
          output_tokens: outputTokens,
          estimated_cost_usd: costUsd,
        })
        .eq("id", executionId);

      // Dispatch output actions if configured
      if (outputActions && outputActions.length > 0) {
        const { tasks } = await import("@trigger.dev/sdk/v3");
        for (const action of outputActions) {
          await tasks.trigger("execute-output-action", {
            executionId,
            userId,
            action,
            agentOutput: output,
            eventPayload: eventPayload || {},
            eventMetadata: eventMetadata || {},
            eventSource: payload.agentType === "claude" ? "github" : "github",
            eventType: "",
          });
        }
      }

      return { success: true, output, inputTokens, outputTokens, costUsd };
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : String(error);
      await supabase
        .from("executions")
        .update({
          status: "failed",
          error: errorMessage,
          completed_at: new Date().toISOString(),
        })
        .eq("id", executionId);

      throw error; // Re-throw so Trigger.dev handles retry
    }
  },
});
```

### 5.7 Task: run-agent-with-tools

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/tasks/run-agent-with-tools.ts`

Same structure as `run-agent` but uses the `runClaudeWithTools` function and the `agentToolQueue`.

```typescript
import { task } from "@trigger.dev/sdk";
import { createServiceClient, estimateCost } from "@reactor/shared";
import type { RunAgentPayload } from "@reactor/shared";
import { createAnthropicClient, runClaudeWithTools } from "../lib/claude";
import { agentToolQueue } from "../queues";
import type Anthropic from "@anthropic-ai/sdk";

export const runAgentWithToolsTask = task({
  id: "run-agent-with-tools",
  queue: agentToolQueue,
  machine: { preset: "small-2x" },
  run: async (payload: RunAgentPayload) => {
    const {
      executionId, userId, agentModel,
      renderedPrompt, systemPrompt, tools, maxTokens,
      outputActions, eventPayload, eventMetadata,
    } = payload;

    const supabase = createServiceClient();

    await supabase
      .from("executions")
      .update({ status: "running", started_at: new Date().toISOString() })
      .eq("id", executionId);

    try {
      const client = await createAnthropicClient(userId);

      // Convert tool definitions to Anthropic format
      const anthropicTools: Anthropic.Tool[] = (tools || []).map((t) => ({
        name: t.name,
        description: t.description,
        input_schema: t.input_schema as Anthropic.Tool.InputSchema,
      }));

      const result = await runClaudeWithTools({
        client,
        model: agentModel,
        prompt: renderedPrompt,
        systemPrompt: systemPrompt || undefined,
        tools: anthropicTools,
        maxTokens: maxTokens || 4096,
        onToolCall: async (name, input) => {
          // Built-in tool implementations
          switch (name) {
            case "fetch_url": {
              const resp = await fetch((input as any).url);
              return await resp.text();
            }
            default:
              return `Tool "${name}" not implemented`;
          }
        },
      });

      const costUsd = estimateCost(agentModel, result.inputTokens, result.outputTokens);

      await supabase
        .from("executions")
        .update({
          status: "completed",
          agent_output: result.output,
          tool_calls: result.toolCalls,
          completed_at: new Date().toISOString(),
          input_tokens: result.inputTokens,
          output_tokens: result.outputTokens,
          estimated_cost_usd: costUsd,
        })
        .eq("id", executionId);

      // Dispatch output actions
      if (outputActions && outputActions.length > 0) {
        const { tasks } = await import("@trigger.dev/sdk/v3");
        for (const action of outputActions) {
          await tasks.trigger("execute-output-action", {
            executionId,
            userId,
            action,
            agentOutput: result.output,
            eventPayload: eventPayload || {},
            eventMetadata: eventMetadata || {},
            eventSource: "github",
            eventType: "",
          });
        }
      }

      return { success: true, output: result.output, toolCalls: result.toolCalls };
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : String(error);
      await supabase
        .from("executions")
        .update({
          status: "failed",
          error: errorMessage,
          completed_at: new Date().toISOString(),
        })
        .eq("id", executionId);
      throw error;
    }
  },
});
```

### 5.8 Task: execute-output-action

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/tasks/execute-output-action.ts`

```typescript
import { task } from "@trigger.dev/sdk";
import { createServiceClient, renderTemplate } from "@reactor/shared";
import type { ExecuteOutputActionPayload, TemplateContext, OutputActionResult } from "@reactor/shared";
import { getSecret } from "../lib/vault";
import { postGitHubComment, addGitHubLabels } from "../lib/output/github";
import { postSlackMessage, postSlackReply } from "../lib/output/slack";
import { postLinearComment } from "../lib/output/linear";
import { postWebhookCallback } from "../lib/output/webhook";
import { outputActionQueue } from "../queues";

export const executeOutputActionTask = task({
  id: "execute-output-action",
  queue: outputActionQueue,
  machine: { preset: "micro" },
  run: async (payload: ExecuteOutputActionPayload) => {
    const {
      executionId, userId, action, agentOutput,
      eventPayload, eventMetadata, eventSource, eventType,
    } = payload;

    const supabase = createServiceClient();

    // Build template context for rendering config values
    const templateContext: TemplateContext = {
      source: eventSource,
      event_type: eventType,
      payload: eventPayload,
      metadata: eventMetadata,
      agent_output: agentOutput,
      execution_id: executionId,
    };

    // Render template variables in config values
    const renderedConfig: Record<string, string> = {};
    for (const [key, value] of Object.entries(action.config)) {
      renderedConfig[key] = renderTemplate(value, templateContext);
    }

    let result: OutputActionResult;

    try {
      switch (action.type) {
        case "github_pr_comment":
        case "github_issue_comment": {
          const token = await getSecret(`github_key_${userId}`);
          const repo = renderedConfig.repo;
          const number = parseInt(
            renderedConfig.pr_number || renderedConfig.issue_number,
            10
          );
          await postGitHubComment(token, repo, number, agentOutput);
          result = { type: action.type, success: true, executed_at: new Date().toISOString() };
          break;
        }
        case "github_add_labels": {
          const token = await getSecret(`github_key_${userId}`);
          const labels = renderedConfig.labels.split(",").map((l) => l.trim());
          await addGitHubLabels(
            token,
            renderedConfig.repo,
            parseInt(renderedConfig.issue_number, 10),
            labels
          );
          result = { type: action.type, success: true, executed_at: new Date().toISOString() };
          break;
        }
        case "slack_reply": {
          const token = await getSecret(`slack_key_${userId}`);
          await postSlackReply(
            token,
            renderedConfig.channel,
            renderedConfig.thread_ts,
            agentOutput
          );
          result = { type: action.type, success: true, executed_at: new Date().toISOString() };
          break;
        }
        case "slack_post": {
          const token = await getSecret(`slack_key_${userId}`);
          await postSlackMessage(token, renderedConfig.channel, agentOutput);
          result = { type: action.type, success: true, executed_at: new Date().toISOString() };
          break;
        }
        case "linear_comment": {
          const token = await getSecret(`linear_key_${userId}`);
          await postLinearComment(token, renderedConfig.issue_id, agentOutput);
          result = { type: action.type, success: true, executed_at: new Date().toISOString() };
          break;
        }
        case "webhook_callback": {
          await postWebhookCallback(renderedConfig.url, {
            execution_id: executionId,
            output: agentOutput,
            source: eventSource,
            event_type: eventType,
          });
          result = { type: action.type, success: true, executed_at: new Date().toISOString() };
          break;
        }
        default:
          result = {
            type: action.type,
            success: false,
            error: `Unknown action type: ${action.type}`,
            executed_at: new Date().toISOString(),
          };
      }
    } catch (error) {
      result = {
        type: action.type,
        success: false,
        error: error instanceof Error ? error.message : String(error),
        executed_at: new Date().toISOString(),
      };
    }

    // Append result to execution's output_results array
    // Using Postgres jsonb_array_append pattern
    await supabase.rpc("append_output_result", {
      p_execution_id: executionId,
      p_result: result,
    });

    return result;
  },
});
```

This task requires an additional Postgres function for appending to the JSONB array:

Add to migration 00002 (or a new migration 00004):

```sql
CREATE OR REPLACE FUNCTION public.append_output_result(
  p_execution_id uuid,
  p_result jsonb
)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$
BEGIN
  UPDATE public.executions
  SET output_results = COALESCE(output_results, '[]'::jsonb) || p_result::jsonb
  WHERE id = p_execution_id;
END;
$$;
```

---

## 6. Output Actions System

### 6.1 GitHub Output Actions

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/lib/output/github.ts`

```typescript
/**
 * Posts a comment on a GitHub issue or pull request.
 * Uses the GitHub REST API.
 */
export async function postGitHubComment(
  token: string,
  repo: string,       // "owner/repo"
  issueNumber: number, // PR number is also an issue number in GitHub
  body: string
): Promise<void> {
  const response = await fetch(
    `https://api.github.com/repos/${repo}/issues/${issueNumber}/comments`,
    {
      method: "POST",
      headers: {
        Authorization: `token ${token}`,
        Accept: "application/vnd.github.v3+json",
        "Content-Type": "application/json",
        "User-Agent": "Reactor-Agent",
      },
      body: JSON.stringify({ body }),
    }
  );

  if (!response.ok) {
    const errorBody = await response.text();
    throw new Error(`GitHub API error (${response.status}): ${errorBody}`);
  }
}

/**
 * Adds labels to a GitHub issue or pull request.
 */
export async function addGitHubLabels(
  token: string,
  repo: string,
  issueNumber: number,
  labels: string[]
): Promise<void> {
  const response = await fetch(
    `https://api.github.com/repos/${repo}/issues/${issueNumber}/labels`,
    {
      method: "POST",
      headers: {
        Authorization: `token ${token}`,
        Accept: "application/vnd.github.v3+json",
        "Content-Type": "application/json",
        "User-Agent": "Reactor-Agent",
      },
      body: JSON.stringify({ labels }),
    }
  );

  if (!response.ok) {
    const errorBody = await response.text();
    throw new Error(`GitHub API error (${response.status}): ${errorBody}`);
  }
}
```

### 6.2 Slack Output Actions

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/lib/output/slack.ts`

```typescript
/**
 * Posts a message to a Slack channel.
 */
export async function postSlackMessage(
  token: string,
  channel: string,
  text: string
): Promise<void> {
  const response = await fetch("https://slack.com/api/chat.postMessage", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ channel, text }),
  });

  const result = await response.json();
  if (!result.ok) {
    throw new Error(`Slack API error: ${result.error}`);
  }
}

/**
 * Posts a reply in a Slack thread.
 */
export async function postSlackReply(
  token: string,
  channel: string,
  threadTs: string,
  text: string
): Promise<void> {
  const response = await fetch("https://slack.com/api/chat.postMessage", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ channel, text, thread_ts: threadTs }),
  });

  const result = await response.json();
  if (!result.ok) {
    throw new Error(`Slack API error: ${result.error}`);
  }
}
```

### 6.3 Linear Output Actions

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/lib/output/linear.ts`

```typescript
/**
 * Posts a comment on a Linear issue via the GraphQL API.
 */
export async function postLinearComment(
  token: string,
  issueId: string,
  body: string
): Promise<void> {
  const response = await fetch("https://api.linear.app/graphql", {
    method: "POST",
    headers: {
      Authorization: token,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      query: `
        mutation CommentCreate($input: CommentCreateInput!) {
          commentCreate(input: $input) {
            success
            comment { id }
          }
        }
      `,
      variables: {
        input: { issueId, body },
      },
    }),
  });

  const result = await response.json();
  if (result.errors) {
    throw new Error(`Linear API error: ${JSON.stringify(result.errors)}`);
  }
}
```

### 6.4 Generic Webhook Callback

**File**: `/Users/criss/work/cue++/reactor/apps/trigger/src/lib/output/webhook.ts`

```typescript
/**
 * POSTs results to a generic webhook URL.
 */
export async function postWebhookCallback(
  url: string,
  payload: Record<string, unknown>
): Promise<void> {
  const response = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  });

  if (!response.ok) {
    throw new Error(`Webhook callback failed (${response.status}): ${await response.text()}`);
  }
}
```

### 6.5 Output Actions Schema (for reactions.output_actions)

The `output_actions` column on `reactions` stores an array of output action configurations. Each element uses template variables that get rendered with the full event context plus the agent's output at execution time.

Example values for `output_actions`:

**Post PR review comment on GitHub**:
```json
[
  {
    "type": "github_pr_comment",
    "config": {
      "repo": "{{metadata.repo}}",
      "pr_number": "{{metadata.pr_number}}"
    }
  }
]
```

**Reply in Slack thread**:
```json
[
  {
    "type": "slack_reply",
    "config": {
      "channel": "{{metadata.channel}}",
      "thread_ts": "{{metadata.thread_ts}}"
    }
  }
]
```

**Multiple output actions (comment on GitHub + post to Slack)**:
```json
[
  {
    "type": "github_issue_comment",
    "config": {
      "repo": "{{metadata.repo}}",
      "issue_number": "{{metadata.issue_number}}"
    }
  },
  {
    "type": "slack_post",
    "config": {
      "channel": "#engineering-alerts"
    }
  }
]
```

---

## 7. Next.js Dashboard (shadcn/ui)

All UI uses **shadcn/ui** components from `@reactor/ui`. The app imports primitives (Button, Card, Table, Dialog, etc.) from `@reactor/ui/components/ui/*` and composes them into feature components in `apps/web/src/components/`.

**shadcn/ui components needed** (install in `packages/ui`):
`button`, `card`, `dialog`, `dropdown-menu`, `input`, `label`, `select`, `separator`, `switch`, `table`, `tabs`, `textarea`, `toast`, `tooltip`, `badge`, `popover`, `form`, `command`, `sheet`, `skeleton`, `scroll-area`, `avatar`, `alert`

### 7.1 Authentication

**File**: `/Users/criss/work/cue++/reactor/apps/web/src/lib/supabase/client.ts`

```typescript
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

**File**: `/Users/criss/work/cue++/reactor/apps/web/src/lib/supabase/server.ts`

```typescript
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll(); },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch { /* Server Component */ }
        },
      },
    }
  );
}
```

**File**: `/Users/criss/work/cue++/reactor/apps/web/src/middleware.ts`

```typescript
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll(); },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => {
            request.cookies.set(name, value);
            response.cookies.set(name, value, options);
          });
        },
      },
    }
  );

  const { data: { user } } = await supabase.auth.getUser();

  // Redirect unauthenticated users to login
  if (!user && !request.nextUrl.pathname.startsWith("/login") &&
      !request.nextUrl.pathname.startsWith("/signup") &&
      !request.nextUrl.pathname.startsWith("/callback")) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  // Redirect authenticated users away from auth pages
  if (user && (request.nextUrl.pathname.startsWith("/login") ||
               request.nextUrl.pathname.startsWith("/signup"))) {
    const url = request.nextUrl.clone();
    url.pathname = "/dashboard";
    return NextResponse.redirect(url);
  }

  return response;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|api).*)"],
};
```

### 7.2 Page Specifications

Each page below describes its purpose, data source, and key shadcn/ui components.

**Dashboard (`/dashboard`)**:
- 3 summary `Card` components: total events (24h), active reactions, running executions
- Recent events in a `Table` (last 10)
- Recent executions in a `Table` (last 10) with realtime `Badge` status updates
- Uses `Skeleton` for loading states
- Data: server component fetching from Supabase

**Events List (`/events`)**:
- `Table` with columns: Time, Source (icon + `Badge`), Event Type, Source Event ID, Processed (`Badge`), metadata preview
- Filters row: `Select` for source, `Input` for event_type, date range, `Select` for processed
- Cursor-based pagination with `Button` prev/next
- Click row → navigate to detail

**Event Detail (`/events/[id]`)**:
- `Card` with event metadata
- Raw payload in collapsible JSON viewer (custom component using `ScrollArea` + syntax highlighting)
- Metadata as key-value pairs in a `Table`
- Linked executions in a `Table`

**Reactions List (`/reactions`)**:
- `Card` grid showing each reaction: name, source `Badge`, event_type, `Switch` for enabled toggle, last triggered, execution count
- `Button` "New Reaction" → `/reactions/new`
- Click card → edit

**Reaction Form (`/reactions/new` and `/reactions/[id]`)**:
- Most complex UI. Uses `react-hook-form` + zod + `Form` component from shadcn
- **Section 1 - Matching**: `Select` for source, `Input` for event type (with `Command`/`Popover` autocomplete per source)
- **Section 2 - Precondition Builder** (`precondition-builder.tsx`): Visual tree builder. Each leaf = `Input` (field) + `Select` (operator) + `Input` (value). AND/OR/NOT groups via `Button` toggles. Raw JSON `Textarea` toggle for power users. Live preview in `Card`
- **Section 3 - Agent Config**: `Select` for agent type, `Select` for model, `Input` for max tokens, `Textarea` for system prompt
- **Section 4 - Prompt Template** (`prompt-editor.tsx`): `Textarea` with `{{variable}}` syntax highlighting. Available variables sidebar via `Popover`. Live preview `Card` with sample payload
- **Section 5 - Output Actions** (`output-action-config.tsx`): Dynamic list. Each action: `Select` for type + config `Input` fields. `Button` to add/remove. Config fields support `{{template}}` variables
- **Section 6 - Controls**: `Switch` for enabled, `Input` for cooldown seconds
- `Button` submit / `Dialog` confirm delete

**Executions List (`/executions`)**:
- `Table`: Time, Status (`Badge` with color — pending=gray, running=blue, completed=green, failed=red), Reaction name, Agent, Duration, Cost, external link to Trigger.dev
- Realtime: `useRealtimeTable` hook updates `Badge` status live
- Filters: `Select` for status, `Select` for agent_type, date range

**Execution Detail (`/executions/[id]`)**:
- `Tabs` with panels: Input, Output, Tool Calls, Output Actions
- **Input tab**: rendered prompt + system prompt in `Card` with `ScrollArea`
- **Output tab**: agent_output rendered as markdown in `Card`
- **Tool Calls tab**: expandable `Table` of tool call records (name, input/output JSON)
- **Output Actions tab**: `Table` of results with success/failure `Badge`
- Sidebar `Card`: tokens, cost, duration, status `Badge` (realtime), Trigger.dev run link
- `Skeleton` loading state

**Connections (`/connections`)**:
- `Card` list with provider icons (Lucide: `Github`, `Slack`, `MessageSquare`, `Key`)
- `Dialog` form to add: `Select` provider, `Input` API key (password type), `Input` label
- Last verified timestamp + `Button` "Test" to re-validate
- `Card` showing webhook URLs per source (computed, read-only `Input` with copy `Button`)

**Settings (`/settings`)**:
- `Card` with user email
- `Button` to change password (opens `Dialog`)
- `Button` sign out (with `AlertDialog` confirm)

### 7.3 Realtime Hook

**File**: `/Users/criss/work/cue++/reactor/apps/web/src/lib/hooks/use-realtime.ts`

```typescript
import { useEffect, useState } from "react";
import { createClient } from "../supabase/client";
import type { RealtimeChannel } from "@supabase/supabase-js";

/**
 * Hook to subscribe to Supabase Realtime changes on a table.
 * Returns updated rows in realtime.
 */
export function useRealtimeTable<T extends { id: string }>(
  table: string,
  filter?: string // e.g., "user_id=eq.abc123"
) {
  const [updates, setUpdates] = useState<Map<string, Partial<T>>>(new Map());

  useEffect(() => {
    const supabase = createClient();
    let channel: RealtimeChannel;

    const channelConfig = filter
      ? supabase.channel(`${table}-changes`).on(
          "postgres_changes",
          { event: "*", schema: "public", table, filter },
          (payload) => {
            setUpdates((prev) => {
              const next = new Map(prev);
              next.set((payload.new as T).id, payload.new as T);
              return next;
            });
          }
        )
      : supabase.channel(`${table}-changes`).on(
          "postgres_changes",
          { event: "*", schema: "public", table },
          (payload) => {
            setUpdates((prev) => {
              const next = new Map(prev);
              next.set((payload.new as T).id, payload.new as T);
              return next;
            });
          }
        );

    channel = channelConfig.subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [table, filter]);

  return updates;
}
```

---

## 8. Testing Strategy

### 8.1 Unit Tests (packages/shared)

**Test files**: `/Users/criss/work/cue++/reactor/packages/shared/src/__tests__/`

- `preconditions.test.ts`: Test every operator, compound conditions, dot-notation traversal, null preconditions, edge cases (missing fields, wrong types, invalid regex).
- `templates.test.ts`: Test interpolation, missing values, object values, nested paths, edge cases.
- `validators/*.test.ts`: Test each validator with known-good and known-bad signatures.
- `output-actions.test.ts`: Test validation function.
- `cost.test.ts`: Test cost estimation.

Run with: `cd packages/shared && npm test`

### 8.2 Integration Tests (Trigger.dev tasks)

**Test files**: `/Users/criss/work/cue++/reactor/apps/trigger/src/__tests__/`

Test Trigger.dev tasks using Trigger.dev's test framework:
- Mock Supabase client (use `vitest` mocks).
- Mock Anthropic SDK.
- Test `run-agent` task with mock payloads.
- Test `execute-output-action` with mock API responses.

### 8.3 Edge Function Tests

Edge Functions can be tested locally with `supabase functions serve`. Manual testing approach:

1. Start Supabase locally: `supabase start`
2. Serve functions: `supabase functions serve --env-file supabase/.env.local`
3. Send test webhooks with `curl`:

```bash
# Test GitHub webhook
curl -X POST http://localhost:54321/functions/v1/webhook-github \
  -H "Content-Type: application/json" \
  -H "X-GitHub-Event: pull_request" \
  -H "X-GitHub-Delivery: test-001" \
  -H "X-Hub-Signature-256: sha256=<computed_hmac>" \
  -d '{"action":"opened","pull_request":{"title":"Test","base":{"ref":"main"},"head":{"ref":"feat"},"user":{"login":"dev"},"draft":false,"number":1},"repository":{"full_name":"test/repo"},"sender":{"login":"dev"}}'
```

### 8.4 End-to-End Test Scenario

1. Start Supabase locally (`supabase start`).
2. Run migrations (`supabase db push`).
3. Create a test user via Supabase Auth (dashboard at `localhost:54323`).
4. Store a test Anthropic API key in Vault via SQL.
5. Create a test reaction (PR review) via SQL.
6. Start Trigger.dev dev mode (`npx trigger.dev dev`).
7. Serve Edge Functions (`supabase functions serve`).
8. Send a simulated GitHub webhook to `webhook-github`.
9. Verify: event appears in `events` table, execution created in `executions`, Trigger.dev task runs, execution status updates to `completed`, output action posts a comment.

### 8.5 Local Development Workflow

```bash
# Terminal 1: Supabase
pnpm dev:supabase

# Terminal 2: Edge Functions
supabase functions serve --env-file supabase/.env.local

# Terminal 3: Trigger.dev + Dashboard (via Turborepo)
pnpm dev

# Terminal 4: Send test events
curl -X POST http://localhost:54321/functions/v1/webhook-generic \
  -H "Authorization: Bearer test-token" \
  -H "Content-Type: application/json" \
  -d '{"source":"test","event_type":"test.event","payload":{"message":"hello"}}'
```

---

## 9. Deployment

### 9.1 Supabase Project

1. Create project at supabase.com (or use existing).
2. Link: `supabase link --project-ref <ref>`
3. Push migrations: `supabase db push`
4. Configure database webhook via Dashboard (Database > Webhooks, as specified in section 2.4).
5. Set Edge Function secrets:
   ```bash
   supabase secrets set TRIGGER_SECRET_KEY=tr_prod_...
   supabase secrets set GITHUB_WEBHOOK_SECRET=whsec_...
   supabase secrets set SLACK_SIGNING_SECRET=...
   supabase secrets set LINEAR_WEBHOOK_SECRET=...
   supabase secrets set GENERIC_WEBHOOK_TOKEN=...
   ```

### 9.2 Edge Functions Deployment

```bash
supabase functions deploy webhook-github --project-ref <ref>
supabase functions deploy webhook-slack --project-ref <ref>
supabase functions deploy webhook-linear --project-ref <ref>
supabase functions deploy webhook-generic --project-ref <ref>
supabase functions deploy evaluate-event --project-ref <ref>
supabase functions deploy store-secret --project-ref <ref>
```

**Important**: The `evaluate-event` function is called by the database webhook, which authenticates via the service role key automatically. Other functions need `--no-verify-jwt` only if called without auth headers; in this design, all webhook functions handle auth themselves (signature validation), so they should have JWT verification disabled in `supabase/config.toml`:

```toml
[functions.webhook-github]
verify_jwt = false

[functions.webhook-slack]
verify_jwt = false

[functions.webhook-linear]
verify_jwt = false

[functions.webhook-generic]
verify_jwt = false

[functions.evaluate-event]
verify_jwt = false

[functions.store-secret]
verify_jwt = false
```

The `store-secret` function handles its own auth (extracts JWT from the Authorization header and calls `supabase.auth.getUser()`).

### 9.3 Trigger.dev Deployment

1. Create project at cloud.trigger.dev.
2. Set environment variables in the Trigger.dev dashboard:
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_ROLE_KEY`
3. Deploy:
   ```bash
   cd trigger
   npx trigger.dev deploy
   ```

### 9.4 Next.js Dashboard Deployment (Vercel)

1. Connect the repository to Vercel.
2. Set root directory to `apps/web/`.
3. Set environment variables:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
4. Vercel auto-detects Turborepo and builds dependencies.
5. Deploy.

### 9.5 Complete Environment Variable Reference

| Variable | Where | Description |
|----------|-------|-------------|
| `SUPABASE_URL` | Edge Functions (auto), Trigger.dev, Dashboard | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Edge Functions (auto), Trigger.dev | Service role key (bypasses RLS) |
| `SUPABASE_ANON_KEY` | Dashboard | Anon key for browser client |
| `TRIGGER_SECRET_KEY` | Edge Functions (secret) | Trigger.dev API key for dispatching |
| `TRIGGER_API_URL` | Edge Functions (secret) | `https://api.trigger.dev` (default) |
| `GITHUB_WEBHOOK_SECRET` | Edge Functions (secret) | GitHub webhook signing secret |
| `SLACK_SIGNING_SECRET` | Edge Functions (secret) | Slack app signing secret |
| `LINEAR_WEBHOOK_SECRET` | Edge Functions (secret) | Linear webhook signing secret |
| `GENERIC_WEBHOOK_TOKEN` | Edge Functions (secret) | Bearer token for generic webhook |
| `NEXT_PUBLIC_SUPABASE_URL` | Dashboard (env) | Same as SUPABASE_URL (public) |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Dashboard (env) | Same as SUPABASE_ANON_KEY (public) |

---

## Implementation Order

The recommended build sequence (each step produces a testable artifact):

1. **Turborepo scaffold** -- `pnpm init`, `pnpm-workspace.yaml`, `turbo.json`, tsconfig presets. Verify `pnpm build` runs cleanly with empty packages.

2. **packages/shared** -- types, precondition engine, template renderer, provider interfaces, webhook handler interface. All pure functions, fully unit-testable. Run `pnpm test --filter=@reactor/shared`.

3. **packages/shared/handlers** -- Built-in webhook handlers (github, slack, linear, generic) implementing the `WebhookHandler` interface. Unit test each handler.

4. **packages/ui** -- Initialize shadcn/ui (`pnpm dlx shadcn@latest init`), add all needed components. Verify build.

5. **Database migrations** -- all tables including `providers` table + seed built-in providers. Run `supabase db push`. Verify with SQL.

6. **supabase/functions/_shared** -- the Deno-compatible utility layer.

7. **Generic webhook router** (`supabase/functions/webhook/index.ts`) -- single Edge Function that dispatches by provider ID. Test with curl against `/webhook/generic`. Verify events land in DB.

8. **evaluate-event Edge Function** -- configure the database webhook on INSERT. Create a test reaction via SQL. Verify the evaluation pipeline creates an execution and dispatches to Trigger.dev.

9. **apps/trigger run-agent task** -- deploy a minimal version. Test end-to-end: webhook -> event -> evaluate -> dispatch -> agent runs -> execution updated.

10. **store-secret Edge Function** -- enables the dashboard to store API keys.

11. **execute-output-action task** -- post results back to sources. Test with a real GitHub comment.

12. **run-agent-with-tools task** -- add the agentic loop with tool calling.

13. **poll-source task** -- Trigger.dev scheduled task for polling-based providers.

14. **apps/web Dashboard** -- build in order:
    - Auth + middleware
    - Layout + sidebar (shadcn Sheet/Sidebar)
    - Connections page (API keys + OAuth flows for providers)
    - Reactions CRUD (precondition builder + prompt editor + output action config)
    - Events list + detail
    - Executions list (realtime) + detail
    - Dashboard overview
    - Provider management (optional: add custom providers via UI)

15. **OAuth routes** (`apps/web/src/app/api/oauth/[provider]`) -- OAuth initiation + callback for providers that support it (GitHub App, Slack App).

---

### Critical Files for Implementation

| File | Why Critical |
|------|-------------|
| `packages/shared/src/types.ts` | All type definitions — single source of truth for Edge Functions, Trigger.dev, and dashboard |
| `supabase/migrations/00001_create_reactor_tables.sql` | Database schema is the foundation; output_actions column design affects the entire flow |
| `supabase/functions/evaluate-event/index.ts` | Core orchestration: matches reactions, evaluates preconditions, renders prompts, dispatches to Trigger.dev |
| `apps/trigger/src/tasks/run-agent.ts` | Primary agent task: loads keys from Vault, runs Claude/Codex, writes results, handles errors |
| `apps/trigger/src/tasks/execute-output-action.ts` | Closes the loop: posts results back to GitHub/Slack/Linear with proper API auth |
| `turbo.json` | Turborepo pipeline — must correctly express dependency graph for builds |
| `packages/ui/components.json` | shadcn/ui config — aliases and paths must match the monorepo structure |
| `apps/web/src/components/reaction-form.tsx` | Most complex UI: precondition builder + prompt editor + output action config |
| `apps/web/src/middleware.ts` | Auth gate — all app routes depend on this working correctly |

### Verification

1. `pnpm build` — all packages and apps build cleanly via Turborepo
2. `pnpm test --filter=@reactor/shared` — precondition engine + template renderer pass all tests
3. `curl` test webhook → event lands in DB → evaluate-event fires → execution created → Trigger.dev task runs → status = completed
4. Dashboard shows events, executions update in realtime
5. Output action posts a GitHub comment on a test PR