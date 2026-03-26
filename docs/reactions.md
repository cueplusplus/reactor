# Reactions

## Overview

Reactions are user-defined rules that map external events to AI agent actions. Each reaction specifies:

1. **What to match**: source + event type
2. **When to trigger**: precondition on the event payload
3. **What to do**: agent type, model, prompt template, tools

## Reaction Lifecycle

```
Event arrives
    │
    ▼
Match reactions by (source, event_type, enabled=true)
    │
    ▼
For each matching reaction:
    │
    ├─ Evaluate precondition against event payload
    │   └─ If precondition fails → skip
    │
    ├─ Check cooldown (last_triggered_at + cooldown_seconds > now → skip)
    │
    ├─ Render prompt template with event data
    │
    ├─ Create execution record (status: pending)
    │
    └─ Dispatch to trigger.dev
```

## Example Reactions

### 1. Auto-review PRs to main

```json
{
  "name": "Review PRs to main",
  "source": "github",
  "event_type": "pull_request.opened",
  "precondition": {
    "and": [
      { "field": "payload.pull_request.base.ref", "op": "eq", "value": "main" },
      { "field": "payload.pull_request.draft", "op": "eq", "value": false }
    ]
  },
  "agent_type": "claude",
  "agent_model": "claude-sonnet-4-6",
  "prompt_template": "Review this pull request for bugs, security issues, and code quality.\n\nTitle: {{payload.pull_request.title}}\nAuthor: {{payload.pull_request.user.login}}\nDescription:\n{{payload.pull_request.body}}\n\nDiff URL: {{payload.pull_request.diff_url}}\nFiles changed: {{payload.pull_request.changed_files}}\nAdditions: {{payload.pull_request.additions}}\nDeletions: {{payload.pull_request.deletions}}\n\nProvide a structured review with sections: Summary, Issues Found, Suggestions.",
  "system_prompt": "You are a senior code reviewer. Be concise, focus on substantive issues, not style nitpicks.",
  "max_tokens": 4096,
  "cooldown_seconds": 0
}
```

### 2. Summarize Slack threads when mentioned

```json
{
  "name": "Summarize thread on mention",
  "source": "slack",
  "event_type": "app_mention",
  "precondition": {
    "field": "payload.event.text",
    "op": "contains",
    "value": "summarize"
  },
  "agent_type": "claude",
  "agent_model": "claude-haiku-4-5",
  "prompt_template": "Summarize this Slack thread concisely:\n\nChannel: {{metadata.channel}}\nMessage: {{payload.event.text}}\nThread timestamp: {{payload.event.thread_ts}}\n\nProvide a 2-3 sentence summary of the key points and any action items.",
  "max_tokens": 1024,
  "cooldown_seconds": 60
}
```

### 3. Triage new GitHub issues

```json
{
  "name": "Auto-triage issues",
  "source": "github",
  "event_type": "issue.opened",
  "precondition": null,
  "agent_type": "claude",
  "agent_model": "claude-haiku-4-5",
  "prompt_template": "Triage this GitHub issue:\n\nTitle: {{payload.issue.title}}\nBody:\n{{payload.issue.body}}\nLabels: {{payload.issue.labels}}\nRepo: {{metadata.repo}}\n\nClassify as: bug, feature-request, question, or docs.\nEstimate priority: P0 (critical), P1 (high), P2 (medium), P3 (low).\nSuggest relevant labels.\nProvide a one-line summary.",
  "max_tokens": 512,
  "cooldown_seconds": 0
}
```

### 4. Alert on CI failures

```json
{
  "name": "Analyze CI failures",
  "source": "github",
  "event_type": "check_run.completed",
  "precondition": {
    "field": "payload.check_run.conclusion",
    "op": "eq",
    "value": "failure"
  },
  "agent_type": "claude",
  "agent_model": "claude-sonnet-4-6",
  "prompt_template": "A CI check failed:\n\nCheck: {{payload.check_run.name}}\nRepo: {{metadata.repo}}\nBranch: {{payload.check_run.check_suite.head_branch}}\nCommit: {{payload.check_run.head_sha}}\nOutput title: {{payload.check_run.output.title}}\nOutput summary: {{payload.check_run.output.summary}}\n\nAnalyze the failure and suggest a fix. If the failure is a flaky test, say so.",
  "max_tokens": 2048,
  "cooldown_seconds": 300
}
```

### 5. Codex-based code fix on push

```json
{
  "name": "Auto-fix lint errors on push",
  "source": "github",
  "event_type": "push",
  "precondition": {
    "field": "payload.ref",
    "op": "starts_with",
    "value": "refs/heads/fix/"
  },
  "agent_type": "codex",
  "agent_model": "gpt-5.3-codex",
  "prompt_template": "The following commits were pushed to {{payload.ref}}:\n\n{{payload.head_commit.message}}\n\nFiles modified: {{payload.head_commit.modified}}\n\nRun the linter and fix any errors found. Commit the fixes.",
  "max_tokens": 4096,
  "cooldown_seconds": 120
}
```

## Precondition Evaluation Engine

The evaluation engine is a recursive JSON predicate evaluator. Implementation:

```typescript
function evaluatePrecondition(precondition: any, event: any): boolean {
  if (precondition === null || precondition === undefined) {
    return true; // No precondition = always match
  }

  // Compound: AND
  if (precondition.and) {
    return precondition.and.every((sub: any) => evaluatePrecondition(sub, event));
  }

  // Compound: OR
  if (precondition.or) {
    return precondition.or.some((sub: any) => evaluatePrecondition(sub, event));
  }

  // Compound: NOT
  if (precondition.not) {
    return !evaluatePrecondition(precondition.not, event);
  }

  // Leaf: field comparison
  const { field, op, value } = precondition;
  const actual = getNestedValue(event, field);

  switch (op) {
    case "eq":       return actual === value;
    case "neq":      return actual !== value;
    case "gt":       return actual > value;
    case "gte":      return actual >= value;
    case "lt":       return actual < value;
    case "lte":      return actual <= value;
    case "contains": return typeof actual === "string" && actual.includes(value);
    case "starts_with": return typeof actual === "string" && actual.startsWith(value);
    case "ends_with":   return typeof actual === "string" && actual.endsWith(value);
    case "in":       return Array.isArray(value) && value.includes(actual);
    case "exists":   return actual !== undefined && actual !== null;
    case "regex":    return typeof actual === "string" && new RegExp(value).test(actual);
    default:         return false;
  }
}

function getNestedValue(obj: any, path: string): any {
  return path.split(".").reduce((current, key) => current?.[key], obj);
}
```

## Prompt Template Rendering

The template engine uses Mustache-style interpolation:

```typescript
function renderTemplate(template: string, event: any): string {
  return template.replace(/\{\{([^}]+)\}\}/g, (match, path) => {
    const value = getNestedValue(event, path.trim());
    if (value === undefined || value === null) return "";
    if (typeof value === "object") return JSON.stringify(value, null, 2);
    return String(value);
  });
}
```

The `event` object passed to rendering has this shape:

```typescript
{
  source: "github",
  event_type: "pull_request.opened",
  payload: { /* raw webhook body */ },
  metadata: { /* extracted fields */ },
}
```

## LLM-Based Preconditions (Advanced)

For cases where rule-based matching isn't expressive enough, you can use a lightweight LLM call as the precondition:

```json
{
  "name": "Respond to deploy requests in Slack",
  "source": "slack",
  "event_type": "message",
  "precondition": {
    "llm": {
      "model": "claude-haiku-4-5",
      "prompt": "Does this Slack message contain a request to deploy something? Answer only 'yes' or 'no'.\n\nMessage: {{payload.event.text}}"
    }
  },
  "agent_type": "claude",
  "agent_model": "claude-sonnet-4-6",
  "prompt_template": "..."
}
```

This costs ~$0.001 per evaluation with Haiku and handles fuzzy intent matching that rules can't express.

## Cooldowns

The `cooldown_seconds` field prevents rapid-fire triggering. Use cases:

- **0**: No cooldown — fire on every matching event (good for PRs)
- **60**: 1 minute — prevent duplicate processing of rapid Slack messages
- **300**: 5 minutes — CI failure analysis (avoid spam on flaky tests)
- **3600**: 1 hour — daily digest-style reactions

The cooldown is checked by comparing `reaction.last_triggered_at + cooldown_seconds` against the current time. After dispatching, `last_triggered_at` is updated.

## Managing Reactions

Reactions are managed via:

1. **Direct database access** — Supabase dashboard or SQL
2. **API** — Supabase auto-generated REST API (protected by RLS)
3. **UI** (future) — A frontend dashboard for creating/editing reactions
