# Authentication & Security

## Overview

Reactor uses a multi-layer authentication approach:

1. **User authentication**: Supabase Auth (email/password, magic link, or OAuth)
2. **API key storage**: Supabase Vault (encrypted at rest)
3. **Webhook validation**: Per-source signing secrets
4. **Service-to-service**: Supabase service role key + Trigger.dev secret key

## Multi-User Model

Each user brings their own API keys. There is no shared billing or shared rate limits.

```
User A                          User B
  │                               │
  ├─ Anthropic API key ─┐    ├─ Anthropic API key ─┐
  ├─ OpenAI API key ────┤    ├─ OpenAI API key ────┤
  ├─ GitHub webhook ────┤    ├─ GitHub webhook ────┤
  └─ Slack webhook ─────┘    └─ Slack webhook ─────┘
          │                           │
          ▼                           ▼
    Supabase Vault              Supabase Vault
    (encrypted, per-user)       (encrypted, per-user)
```

Benefits:
- Each user's API usage bills to their own account
- No shared rate limits
- No risk of one user consuming another's credits
- If a key is compromised, only that user is affected
- Revoking access = deactivate their Supabase Auth account

## API Key Setup Flow

### 1. User stores their Anthropic API key

```typescript
// Frontend: settings page
const storeApiKey = async (provider: string, apiKey: string) => {
  const { data: { user } } = await supabase.auth.getUser();

  // Store in Vault via Edge Function (client can't access Vault directly)
  const { data, error } = await supabase.functions.invoke("store-secret", {
    body: {
      provider,
      apiKey,
    },
  });
};
```

```typescript
// Edge Function: store-secret
Deno.serve(async (req) => {
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  // Verify the user is authenticated
  const authHeader = req.headers.get("Authorization");
  const { data: { user }, error } = await supabase.auth.getUser(
    authHeader?.replace("Bearer ", "")
  );
  if (error || !user) {
    return new Response("Unauthorized", { status: 401 });
  }

  const { provider, apiKey } = await req.json();
  const secretName = `${provider}_key_${user.id}`;

  // Validate the key works before storing
  if (provider === "anthropic") {
    try {
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
          messages: [{ role: "user", content: "hi" }],
        }),
      });
      if (!resp.ok) throw new Error("Invalid key");
    } catch {
      return new Response(JSON.stringify({ error: "Invalid API key" }), {
        status: 400,
      });
    }
  }

  // Store in Vault
  await supabase.rpc("create_or_update_secret", {
    secret_name: secretName,
    secret_value: apiKey,
    secret_description: `${provider} API key for user ${user.id}`,
  });

  // Update connections table
  await supabase.from("connections").upsert({
    user_id: user.id,
    provider,
    label: `${provider}-default`,
    active: true,
    last_verified_at: new Date().toISOString(),
  });

  return new Response(JSON.stringify({ success: true }));
});
```

### 2. Vault helper function (Postgres)

```sql
-- Helper to create or update a Vault secret
CREATE OR REPLACE FUNCTION create_or_update_secret(
  secret_name text,
  secret_value text,
  secret_description text DEFAULT ''
)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  existing_id uuid;
BEGIN
  SELECT id INTO existing_id
  FROM vault.secrets
  WHERE name = secret_name;

  IF existing_id IS NOT NULL THEN
    UPDATE vault.secrets
    SET secret = secret_value, description = secret_description, updated_at = now()
    WHERE id = existing_id;
  ELSE
    INSERT INTO vault.secrets (name, secret, description)
    VALUES (secret_name, secret_value, secret_description);
  END IF;
END;
$$;

-- Helper to get a decrypted secret
CREATE OR REPLACE FUNCTION get_decrypted_secret(secret_name text)
RETURNS TABLE (decrypted_secret text)
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT decrypted_secret
  FROM vault.decrypted_secrets
  WHERE name = secret_name;
$$;
```

### 3. Retrieving keys at runtime (in trigger.dev tasks)

```typescript
// Inside a trigger.dev task
const { data } = await supabase
  .rpc("get_decrypted_secret", { secret_name: `anthropic_key_${userId}` });

const anthropic = new Anthropic({ apiKey: data[0].decrypted_secret });
```

## Webhook Security

Each external source uses its own signing mechanism. Webhook secrets are stored in Vault.

### Storing webhook secrets

When a user configures a GitHub webhook:

1. Generate a random secret: `crypto.randomUUID()`
2. Store in Vault: `github_webhook_${userId}`
3. Display to user so they can paste it into GitHub's webhook settings
4. The Edge Function retrieves the secret to validate incoming webhooks

### Validation per source

| Source | Method | Header |
|--------|--------|--------|
| GitHub | HMAC-SHA256 | `X-Hub-Signature-256` |
| Slack | HMAC-SHA256 with timestamp | `X-Slack-Signature` |
| Linear | HMAC-SHA256 | `Linear-Signature` |
| Generic | Bearer token | `Authorization` |

## Row Level Security (RLS)

All tables use RLS scoped by `user_id`:

```sql
-- Users can only see their own data
CREATE POLICY "user_isolation" ON events
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "user_isolation" ON reactions
  FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "user_isolation" ON executions
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "user_isolation" ON connections
  FOR ALL USING (auth.uid() = user_id);
```

Edge Functions and trigger.dev tasks use the **service role key** to bypass RLS when they need cross-user access (e.g., inserting events from webhooks, updating execution status).

## Service-to-Service Auth

| Connection | Credential | Storage |
|------------|-----------|---------|
| Edge Function → Supabase DB | `SUPABASE_SERVICE_ROLE_KEY` | Edge Function env var |
| Edge Function → Trigger.dev | `TRIGGER_SECRET_KEY` | Edge Function env var |
| Trigger.dev → Supabase DB | `SUPABASE_SERVICE_ROLE_KEY` | Trigger.dev env var |
| Trigger.dev → Claude API | Per-user key from Vault | Supabase Vault |
| Trigger.dev → OpenAI API | Per-user key from Vault | Supabase Vault |

## Security Considerations

1. **Never log API keys** — mask them in logs and error messages
2. **Validate keys on storage** — make a test API call before accepting a key
3. **Rotate webhook secrets** — provide a UI to regenerate webhook secrets
4. **Rate limit webhook endpoints** — prevent abuse of your Edge Functions
5. **Audit trail** — the `executions` table provides a full audit trail of what was run and when
6. **Vault limits** — Supabase Vault supports up to 100 secrets per project; plan accordingly for multi-user scale
