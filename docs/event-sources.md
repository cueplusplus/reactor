# Event Sources

## Overview

Reactor receives events from external services via webhooks. Each source has a dedicated Supabase Edge Function that validates the webhook, normalizes the payload, and inserts it into the `events` table.

## Supported Sources

### GitHub

**Webhook setup**: Repository Settings > Webhooks > Add webhook

| Field | Value |
|-------|-------|
| Payload URL | `https://<project-ref>.supabase.co/functions/v1/webhook-github` |
| Content type | `application/json` |
| Secret | Generate and store in Supabase Vault |

**Useful event types**:

| GitHub Event | `event_type` in Reactor | Description |
|-------------|------------------------|-------------|
| `push` | `push` | Code pushed to a branch |
| `pull_request` (action: opened) | `pull_request.opened` | New PR created |
| `pull_request` (action: closed, merged: true) | `pull_request.merged` | PR merged |
| `pull_request_review` | `pull_request_review` | PR review submitted |
| `issue_comment` | `issue_comment.created` | Comment on issue or PR |
| `issues` (action: opened) | `issue.opened` | New issue created |
| `check_run` (conclusion: failure) | `check_run.failed` | CI check failed |
| `release` | `release.published` | New release published |
| `workflow_run` | `workflow_run.completed` | GitHub Actions workflow finished |

**Webhook validation**: GitHub signs payloads with HMAC-SHA256 using the webhook secret. The Edge Function must verify the `X-Hub-Signature-256` header.

```typescript
// Edge Function: webhook-github
import { createHmac } from "node:crypto";

Deno.serve(async (req) => {
  const body = await req.text();
  const signature = req.headers.get("x-hub-signature-256");
  const event = req.headers.get("x-github-event");
  const deliveryId = req.headers.get("x-github-delivery");

  // Verify signature
  const secret = Deno.env.get("GITHUB_WEBHOOK_SECRET");
  const expected = "sha256=" + createHmac("sha256", secret).update(body).digest("hex");

  if (signature !== expected) {
    return new Response("Invalid signature", { status: 401 });
  }

  const payload = JSON.parse(body);
  const action = payload.action;
  const eventType = action ? `${event}.${action}` : event;

  // Insert into events table
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL"),
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")
  );

  await supabase.from("events").insert({
    user_id: await getUserForConnection("github", payload),
    source: "github",
    event_type: eventType,
    source_event_id: deliveryId,
    payload,
    metadata: {
      repo: payload.repository?.full_name,
      sender: payload.sender?.login,
      action,
    },
  });

  return new Response("OK", { status: 200 });
});
```

---

### Slack

**Setup**: Create a Slack App at api.slack.com > Event Subscriptions > Enable

| Field | Value |
|-------|-------|
| Request URL | `https://<project-ref>.supabase.co/functions/v1/webhook-slack` |

**Important**: Slack sends a URL verification challenge on setup. Your Edge Function must handle this:

```typescript
if (payload.type === "url_verification") {
  return new Response(JSON.stringify({ challenge: payload.challenge }), {
    headers: { "Content-Type": "application/json" },
  });
}
```

**Useful event types**:

| Slack Event | `event_type` in Reactor | Description |
|-------------|------------------------|-------------|
| `message` | `message` | Message posted in a channel |
| `app_mention` | `app_mention` | Bot mentioned with @reactor |
| `reaction_added` | `reaction_added` | Emoji reaction added to a message |
| `channel_created` | `channel_created` | New channel created |
| `member_joined_channel` | `member_joined` | User joined a channel |

**Webhook validation**: Slack uses a signing secret with timestamp-based HMAC-SHA256. Verify the `X-Slack-Signature` header against `v0=HMAC(signing_secret, "v0:{timestamp}:{body}")`.

**Required scopes**: `channels:history`, `groups:history`, `chat:write`, `app_mentions:read`

---

### Linear

**Webhook setup**: Settings > API > Webhooks > New webhook

| Field | Value |
|-------|-------|
| URL | `https://<project-ref>.supabase.co/functions/v1/webhook-linear` |
| Resource types | Issues, Comments, Projects (select as needed) |

**Useful event types**:

| Linear Event | `event_type` in Reactor | Description |
|-------------|------------------------|-------------|
| Issue create | `issue.created` | New issue created |
| Issue update (state change) | `issue.updated` | Issue status changed |
| Comment create | `comment.created` | New comment on issue |
| Project update | `project.updated` | Project status changed |

**Webhook validation**: Linear sends a signing secret in the `Linear-Signature` header. Verify with HMAC-SHA256.

---

### Jira

**Webhook setup**: System > WebHooks > Create

| Field | Value |
|-------|-------|
| URL | `https://<project-ref>.supabase.co/functions/v1/webhook-jira` |
| Events | Issue created, Issue updated, Comment created |

**Useful event types**:

| Jira Event | `event_type` in Reactor | Description |
|------------|------------------------|-------------|
| `jira:issue_created` | `issue.created` | New issue |
| `jira:issue_updated` | `issue.updated` | Issue fields changed |
| `comment_created` | `comment.created` | Comment added |
| `sprint_started` | `sprint.started` | Sprint kicked off |

---

### Custom / Generic Webhooks

For any service that supports webhooks, use the generic webhook receiver:

**Endpoint**: `https://<project-ref>.supabase.co/functions/v1/webhook-generic`

Send a POST with:

```json
{
  "source": "my-custom-service",
  "event_type": "deployment.completed",
  "payload": { ... }
}
```

Authenticate with an API key in the `Authorization: Bearer <key>` header.

---

## Adding a New Source

To add support for a new event source:

1. **Create an Edge Function**: `supabase functions new webhook-<source>`
2. **Implement validation**: Each source has its own signing mechanism
3. **Normalize the event**: Map the source's event types to a consistent `event_type` format
4. **Extract metadata**: Pull out commonly-queried fields into the `metadata` jsonb column
5. **Insert into events**: Use the Supabase service role client
6. **Store the webhook secret**: Add the signing secret to Supabase Vault
7. **Deploy**: `supabase functions deploy webhook-<source>`
8. **Configure the external service**: Point the webhook URL to your new Edge Function

## Metadata Extraction

Each source handler extracts commonly-needed fields into the `metadata` column for fast querying without digging into the full payload:

| Source | Metadata Fields |
|--------|----------------|
| GitHub | `repo`, `sender`, `action`, `branch`, `pr_number` |
| Slack | `channel`, `user`, `thread_ts`, `text_preview` |
| Linear | `team`, `assignee`, `state`, `priority` |
| Jira | `project`, `issue_key`, `reporter`, `status` |

## Deduplication

The `events.source_event_id` column with a unique constraint on `(source, source_event_id)` prevents duplicate processing. Most webhook providers include a delivery/request ID in headers:

| Source | Delivery ID Header |
|--------|-------------------|
| GitHub | `X-GitHub-Delivery` |
| Slack | `X-Slack-Request-Timestamp` + event ID |
| Linear | Request body `webhookId` |
| Stripe | `Stripe-Webhook-Id` (if used in future) |
