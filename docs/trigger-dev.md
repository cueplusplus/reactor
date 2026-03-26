# Trigger.dev Integration

## Overview

Trigger.dev v4 serves as the agent execution layer. It receives dispatched tasks from the evaluation pipeline and runs AI agents (Claude or Codex) with no execution timeouts, built-in retries, and full observability.

## Why Trigger.dev

| Requirement | Trigger.dev Solution |
|-------------|---------------------|
| Agent loops can be long-running | No execution timeouts |
| Don't pay for idle wait time | CRIU checkpoint/restore during API waits |
| Reliability | Configurable retries with exponential backoff |
| Concurrency control | Named queues with concurrency limits |
| Observability | OpenTelemetry traces, dashboard, logs |
| Cost efficiency | ~$0.001 for a 60-second agent run on Micro |

## Setup

### Install

```bash
npm install @trigger.dev/sdk
npx trigger.dev@latest init
```

### Configure

```typescript
// trigger.config.ts
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  project: "<your-project-ref>",
  runtime: "node",
  machine: "small-1x",  // 0.5 vCPU / 0.5 GB — good for most agent tasks
  logLevel: "info",
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

## Core Task: Agent Runner

```typescript
// src/trigger/run-agent.ts
import { task } from "@trigger.dev/sdk";
import Anthropic from "@anthropic-ai/sdk";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export const runAgentTask = task({
  id: "run-agent",
  queue: { name: "agent-runs", concurrencyLimit: 5 },
  run: async (payload: {
    executionId: string;
    userId: string;
    agentType: "claude" | "codex";
    agentModel: string;
    renderedPrompt: string;
    systemPrompt?: string;
    tools?: string[];
    maxTokens?: number;
  }) => {
    const { executionId, userId, agentType, agentModel, renderedPrompt, systemPrompt, maxTokens } = payload;

    // Update execution status to running
    await supabase
      .from("executions")
      .update({ status: "running", started_at: new Date().toISOString() })
      .eq("id", executionId);

    try {
      let output: string;
      let inputTokens = 0;
      let outputTokens = 0;

      if (agentType === "claude") {
        output = await runClaudeAgent({ userId, agentModel, renderedPrompt, systemPrompt, maxTokens });
      } else {
        output = await runCodexAgent({ userId, renderedPrompt });
      }

      // Update execution with results
      await supabase
        .from("executions")
        .update({
          status: "completed",
          agent_output: output,
          completed_at: new Date().toISOString(),
          input_tokens: inputTokens,
          output_tokens: outputTokens,
        })
        .eq("id", executionId);

      return { success: true, output };
    } catch (error) {
      await supabase
        .from("executions")
        .update({
          status: "failed",
          error: error instanceof Error ? error.message : String(error),
          completed_at: new Date().toISOString(),
        })
        .eq("id", executionId);

      throw error; // Let trigger.dev handle retry
    }
  },
});

async function runClaudeAgent(params: {
  userId: string;
  agentModel: string;
  renderedPrompt: string;
  systemPrompt?: string;
  maxTokens?: number;
}) {
  // Get user's API key from Vault
  const { data: secret } = await supabase
    .rpc("get_decrypted_secret", { secret_name: `anthropic_key_${params.userId}` });

  const anthropic = new Anthropic({ apiKey: secret.decrypted_secret });

  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: params.renderedPrompt },
  ];

  const response = await anthropic.messages.create({
    model: params.agentModel,
    max_tokens: params.maxTokens || 4096,
    system: params.systemPrompt || undefined,
    messages,
  });

  // Extract text content
  const textBlocks = response.content.filter((b) => b.type === "text");
  return textBlocks.map((b) => b.text).join("\n");
}

async function runCodexAgent(params: {
  userId: string;
  renderedPrompt: string;
}) {
  // Get user's OpenAI key from Vault
  const { data: secret } = await supabase
    .rpc("get_decrypted_secret", { secret_name: `openai_key_${params.userId}` });

  // Use Codex SDK
  const { Codex } = await import("@openai/codex-sdk");
  const codex = new Codex({ apiKey: secret.decrypted_secret });
  const thread = codex.startThread();
  const result = await thread.run(params.renderedPrompt);

  return result.output || "";
}
```

## Agentic Loop with Tool Use

For more complex agent tasks that require tool calling:

```typescript
// src/trigger/run-agent-with-tools.ts
import { task } from "@trigger.dev/sdk";
import Anthropic from "@anthropic-ai/sdk";

export const runAgentWithToolsTask = task({
  id: "run-agent-with-tools",
  queue: { name: "agent-runs", concurrencyLimit: 3 },
  machine: { preset: "medium-1x" },  // More resources for tool-heavy tasks
  run: async (payload: {
    executionId: string;
    userId: string;
    renderedPrompt: string;
    systemPrompt?: string;
    tools: Anthropic.Tool[];
  }) => {
    const anthropic = await getAnthropicClient(payload.userId);

    const messages: Anthropic.MessageParam[] = [
      { role: "user", content: payload.renderedPrompt },
    ];

    // Agentic loop — keep going until the model stops calling tools
    let iterations = 0;
    const maxIterations = 20;

    while (iterations < maxIterations) {
      const response = await anthropic.messages.create({
        model: "claude-sonnet-4-6",
        max_tokens: 4096,
        system: payload.systemPrompt,
        tools: payload.tools,
        messages,
      });

      // If no tool use, we're done
      if (response.stop_reason === "end_turn") {
        const text = response.content
          .filter((b) => b.type === "text")
          .map((b) => b.text)
          .join("\n");
        return { output: text, iterations };
      }

      // Process tool calls
      messages.push({ role: "assistant", content: response.content });

      const toolResults: Anthropic.ToolResultBlockParam[] = [];
      for (const block of response.content) {
        if (block.type === "tool_use") {
          const result = await executeToolCall(block.name, block.input);
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

    return { output: "Max iterations reached", iterations };
  },
});

async function executeToolCall(name: string, input: any): Promise<string> {
  // Implement tool execution logic
  // e.g., fetch a URL, query a database, call an API
  switch (name) {
    case "fetch_pr_diff":
      return await fetchGitHubDiff(input.url);
    case "post_comment":
      return await postGitHubComment(input.repo, input.issue_number, input.body);
    case "search_codebase":
      return await searchCode(input.query, input.repo);
    default:
      return `Unknown tool: ${name}`;
  }
}
```

## Dispatching from the Evaluation Pipeline

The evaluation Edge Function dispatches to trigger.dev like this:

```typescript
// In the evaluate-event Edge Function
import { tasks } from "@trigger.dev/sdk/v3";
import type { runAgentTask } from "../trigger/run-agent";

// After precondition evaluation passes and prompt is rendered:
const handle = await tasks.trigger<typeof runAgentTask>("run-agent", {
  executionId: execution.id,
  userId: reaction.user_id,
  agentType: reaction.agent_type,
  agentModel: reaction.agent_model,
  renderedPrompt: renderedPrompt,
  systemPrompt: reaction.system_prompt,
  tools: reaction.tools,
  maxTokens: reaction.max_tokens,
});

// Update execution with trigger.dev run ID
await supabase
  .from("executions")
  .update({ trigger_run_id: handle.id })
  .eq("id", execution.id);
```

## Development Workflow

```bash
# Start local development (tasks run on your machine)
npx trigger.dev@latest dev

# Deploy to trigger.dev cloud
npx trigger.dev@latest deploy

# View runs in the dashboard
# https://cloud.trigger.dev
```

## Machine Sizes

Choose based on workload:

| Preset | vCPU | RAM | Cost/sec | Use Case |
|--------|------|-----|----------|----------|
| `micro` | 0.25 | 0.25 GB | $0.0000169 | Simple prompt → response |
| `small-1x` | 0.5 | 0.5 GB | $0.0000338 | Standard agent tasks |
| `small-2x` | 1 | 1 GB | $0.0000675 | Tool-heavy agents |
| `medium-1x` | 1 | 2 GB | $0.0000850 | Large context processing |
| `medium-2x` | 2 | 4 GB | $0.0001700 | Parallel subtasks |

## Queue Configuration

```typescript
// Different queues for different priorities
export const reviewPRTask = task({
  id: "review-pr",
  queue: { name: "high-priority", concurrencyLimit: 10 },
  // ...
});

export const digestTask = task({
  id: "daily-digest",
  queue: { name: "low-priority", concurrencyLimit: 2 },
  // ...
});
```

## Environment Variables

Set these in your trigger.dev project settings:

| Variable | Description |
|----------|-------------|
| `SUPABASE_URL` | Your Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key (bypasses RLS) |

User-specific API keys (Anthropic, OpenAI) are stored in Supabase Vault and retrieved at runtime per-execution.
