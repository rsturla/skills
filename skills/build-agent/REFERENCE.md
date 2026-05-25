# Build Agent — Reference

Core agent patterns, cost tracking, tools, guardrails, multi-agent, observability, and project templates.

All code targets **AI SDK v6+** (`ai@^6.0.0`).

## Cost Tracking

```typescript
// src/cost.ts
export interface ModelPricing {
  inputPerMillion: number;
  outputPerMillion: number;
}

const PRICING: Record<string, ModelPricing> = {
  "claude-sonnet-4-6": { inputPerMillion: 3, outputPerMillion: 15 },
  "claude-opus-4-7": { inputPerMillion: 15, outputPerMillion: 75 },
  "claude-haiku-4-5": { inputPerMillion: 0.8, outputPerMillion: 4 },
  "gpt-4o": { inputPerMillion: 2.5, outputPerMillion: 10 },
  "gpt-4o-mini": { inputPerMillion: 0.15, outputPerMillion: 0.6 },
};

export interface UsageRecord {
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;
  costUsd: number;
  model: string;
}

export function calculateCost(
  model: string,
  inputTokens: number,
  outputTokens: number,
): UsageRecord {
  const pricing = PRICING[model];
  if (!pricing) {
    throw new Error(
      `No pricing for model "${model}". Add it to PRICING table before use. ` +
      `Without pricing, cost budgets cannot be enforced.`,
    );
  }
  const costUsd =
    (inputTokens / 1_000_000) * pricing.inputPerMillion +
    (outputTokens / 1_000_000) * pricing.outputPerMillion;

  return { inputTokens, outputTokens, totalTokens: inputTokens + outputTokens, costUsd, model };
}
```

## Provider Examples

```typescript
// Vertex AI (default)
import { vertex } from "@ai-sdk/google-vertex";
export const MODEL_ID = "claude-sonnet-4-6";
export function createModel() { return vertex(MODEL_ID); }

// Anthropic direct
import { anthropic } from "@ai-sdk/anthropic";
export const MODEL_ID = "claude-sonnet-4-6";
export function createModel() { return anthropic(MODEL_ID); }

// OpenAI
import { openai } from "@ai-sdk/openai";
export const MODEL_ID = "gpt-4o";
export function createModel() { return openai(MODEL_ID); }

// AWS Bedrock
import { bedrock } from "@ai-sdk/amazon-bedrock";
export const MODEL_ID = "anthropic.claude-sonnet-4-6-v1";
export function createModel() { return bedrock(MODEL_ID); }

// Ollama (local)
import { ollama } from "ollama-ai-provider";
export const MODEL_ID = "llama3.1";
export function createModel() { return ollama(MODEL_ID); }
```

## Tool Definition (AI SDK v6)

In v6, `parameters` is renamed to `inputSchema`:

```typescript
// src/tools/weather.ts
import { tool } from "ai";
import { z } from "zod";

export const weatherTool = tool({
  description: "Get current weather for a city",
  inputSchema: z.object({
    city: z.string().min(1).max(100).describe("City name"),
    unit: z.enum(["celsius", "fahrenheit"]).default("celsius"),
  }),
  execute: async ({ city, unit }) => {
    const response = await fetch(
      `https://api.weather.example/v1/current?city=${encodeURIComponent(city)}&unit=${unit}`,
    );
    if (!response.ok) {
      return { error: `Weather API returned ${response.status}` };
    }
    return response.json();
  },
});
```

### Reusable Tool Wrapper

```typescript
export function defineTool<TInput extends z.ZodTypeAny, TOutput>(
  config: {
    description: string;
    inputSchema: TInput;
    execute: (input: z.infer<TInput>) => Promise<TOutput>;
  },
) {
  return tool({
    description: config.description,
    inputSchema: config.inputSchema,
    execute: async (rawInput: z.infer<TInput>) => {
      try {
        return { ok: true as const, data: await config.execute(rawInput) };
      } catch (error) {
        return { ok: false as const, error: String(error) };
      }
    },
  });
}
```

## Manual Agent Loop

Use when you need custom per-step logic (cost budgets, loop detection, conditional tool filtering):

```typescript
import { generateText, stepCountIs, type ModelMessage } from "ai";
import { createModel, MODEL_ID } from "./provider.js";
import { calculateCost, type UsageRecord } from "./cost.js";
import * as tools from "./tools/index.js";

interface LoopConfig {
  system: string;
  maxSteps: number;
  maxTimeMs: number;
  maxCostUsd: number;
}

interface LoopResult {
  text: string;
  steps: number;
  usage: UsageRecord;
  durationMs: number;
  earlyStop: string | null;
}

export async function agentLoop(
  prompt: string,
  config: LoopConfig,
): Promise<LoopResult> {
  const messages: ModelMessage[] = [{ role: "user", content: prompt }];
  const startTime = Date.now();
  let steps = 0;
  let totalInputTokens = 0;
  let totalOutputTokens = 0;
  const recentToolCalls: string[] = [];

  while (steps < config.maxSteps) {
    const elapsed = Date.now() - startTime;
    if (elapsed > config.maxTimeMs) {
      return synthesize(messages, config, steps, totalInputTokens, totalOutputTokens, elapsed, "timeout");
    }

    const currentCost = calculateCost(MODEL_ID, totalInputTokens, totalOutputTokens);
    if (currentCost.costUsd > config.maxCostUsd) {
      return synthesize(messages, config, steps, totalInputTokens, totalOutputTokens, elapsed, "cost_limit");
    }

    const result = await generateText({
      model: createModel(),
      system: config.system,
      messages,
      tools,
      stopWhen: stepCountIs(1),
    });

    totalInputTokens += result.usage.inputTokens;
    totalOutputTokens += result.usage.outputTokens;
    steps++;

    messages.push(...result.response.messages);

    if (result.finishReason === "stop") {
      return {
        text: result.text,
        steps,
        usage: calculateCost(MODEL_ID, totalInputTokens, totalOutputTokens),
        durationMs: Date.now() - startTime,
        earlyStop: null,
      };
    }

    if (result.finishReason === "tool-calls") {
      for (const step of result.steps) {
        for (const call of step.toolCalls) {
          const fingerprint = `${call.toolName}:${JSON.stringify(call.input).slice(0, 100)}`;
          recentToolCalls.push(fingerprint);
          if (recentToolCalls.length > 6) recentToolCalls.shift();

          if (detectLoop(recentToolCalls)) {
            return synthesize(
              messages, config, steps, totalInputTokens, totalOutputTokens,
              Date.now() - startTime, "loop_detected",
            );
          }
        }
      }
    }
  }

  return synthesize(
    messages, config, steps, totalInputTokens, totalOutputTokens, Date.now() - startTime, "max_steps",
  );
}

function detectLoop(recent: string[]): boolean {
  if (recent.length < 4) return false;
  const last = recent[recent.length - 1];
  const secondLast = recent[recent.length - 2];
  return recent.filter((r) => r === last).length >= 3
    || (recent.length >= 4 && recent[recent.length - 3] === last && recent[recent.length - 4] === secondLast);
}

async function synthesize(
  messages: ModelMessage[],
  config: LoopConfig,
  steps: number,
  inputTokens: number,
  outputTokens: number,
  durationMs: number,
  reason: string,
): Promise<LoopResult> {
  const result = await generateText({
    model: createModel(),
    system: `${config.system}\n\nYou were stopped early (${reason}). Provide your best answer with what you have.`,
    messages,
    stopWhen: stepCountIs(1),
  });

  return {
    text: result.text,
    steps,
    usage: calculateCost(
      MODEL_ID,
      inputTokens + result.usage.inputTokens,
      outputTokens + result.usage.outputTokens,
    ),
    durationMs,
    earlyStop: reason,
  };
}
```

## Guardrails

### Input Validation

```typescript
// src/guardrails.ts
const DangerousPatterns = [
  /ignore\s+(all\s+)?previous\s+instructions/i,
  /system\s*prompt/i,
  /\b(DROP|DELETE|TRUNCATE)\s+TABLE/i,
];

export function validateInput(input: string): { ok: boolean; reason?: string } {
  if (input.length > 10_000) {
    return { ok: false, reason: "Input exceeds 10,000 character limit" };
  }
  for (const pattern of DangerousPatterns) {
    if (pattern.test(input)) {
      return { ok: false, reason: "Input contains potentially dangerous content" };
    }
  }
  return { ok: true };
}
```

### Output Validation

```typescript
export function validateOutput(output: string): { ok: boolean; reason?: string } {
  const piiPatterns = [
    /\b\d{3}-\d{2}-\d{4}\b/,
    /\b\d{16}\b/,
    /\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b/i,
  ];
  for (const pattern of piiPatterns) {
    if (pattern.test(output)) {
      return { ok: false, reason: "Output may contain PII" };
    }
  }
  return { ok: true };
}
```

### Tool Authorization

```typescript
interface ToolPermission {
  tool: string;
  allowed: boolean;
  requiresConfirmation: boolean;
}

const PERMISSIONS: ToolPermission[] = [
  { tool: "readFile", allowed: true, requiresConfirmation: false },
  { tool: "writeFile", allowed: true, requiresConfirmation: true },
  { tool: "deleteFile", allowed: false, requiresConfirmation: false },
  { tool: "executeCommand", allowed: true, requiresConfirmation: true },
];

export function checkToolPermission(toolName: string): ToolPermission {
  return PERMISSIONS.find((p) => p.tool === toolName)
    ?? { tool: toolName, allowed: false, requiresConfirmation: false };
}
```

## Error Handling and Retries

LLM API calls fail routinely: 429 (rate limit), 529 (overloaded), 500 (internal), network errors. Wrap all
calls with retry logic:

```typescript
// src/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  opts: { maxRetries?: number; baseDelayMs?: number } = {},
): Promise<T> {
  const { maxRetries = 3, baseDelayMs = 1000 } = opts;
  const retryableStatuses = [429, 529, 500, 502, 503, 504];

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error: unknown) {
      if (attempt === maxRetries) throw error;

      const status = (error as { status?: number }).status;
      if (status && !retryableStatuses.includes(status)) throw error;

      const retryAfter = (error as { headers?: Record<string, string> })
        .headers?.["retry-after"];
      const delay = retryAfter
        ? Number(retryAfter) * 1000
        : baseDelayMs * 2 ** attempt + Math.random() * 500;
      await new Promise((r) => setTimeout(r, delay));
    }
  }
  throw new Error("unreachable");
}
```

Usage in agent loop:

```typescript
const result = await withRetry(() =>
  generateText({ model: createModel(), system: config.system, messages, tools,
    stopWhen: stepCountIs(config.maxSteps) }),
);
```

### Rate Limit Awareness

Providers enforce RPM, input TPM, and output TPM limits. Key patterns:

- **Read `retry-after` headers** on 429 responses — the retry utility above handles this
- **Separate eval and production API keys** — eval suites can starve live traffic
- **Ramp up gradually** — sharp traffic increases trigger 429s even within limits
- **Vertex AI has its own quota** — separate from direct Anthropic limits

For concurrent agents, add a semaphore:

```typescript
// src/concurrency.ts
export class Semaphore {
  private queue: (() => void)[] = [];
  private active = 0;

  constructor(private max: number) {}

  async acquire(): Promise<void> {
    if (this.active < this.max) { this.active++; return; }
    return new Promise((resolve) => this.queue.push(resolve));
  }

  release(): void {
    this.active--;
    const next = this.queue.shift();
    if (next) { this.active++; next(); }
  }
}

const llmSemaphore = new Semaphore(5);

export async function withConcurrencyLimit<T>(fn: () => Promise<T>): Promise<T> {
  await llmSemaphore.acquire();
  try { return await fn(); }
  finally { llmSemaphore.release(); }
}
```

## Streaming

For user-facing agents, use `streamText` instead of `generateText` — users see tokens as they arrive:

```typescript
// src/agent-stream.ts
import { streamText, stepCountIs } from "ai";
import { createModel, MODEL_ID } from "./provider.js";
import { calculateCost } from "./cost.js";
import * as tools from "./tools/index.js";

export async function runAgentStream(
  prompt: string,
  config: AgentConfig,
  onChunk: (text: string) => void,
): Promise<AgentResult> {
  const startTime = Date.now();

  const result = streamText({
    model: createModel(),
    system: config.system,
    prompt,
    tools,
    stopWhen: stepCountIs(config.maxSteps),
    onStepFinish: ({ usage }) => {
      // Check cost budget per step
    },
  });

  for await (const chunk of result.textStream) {
    onChunk(chunk);
  }

  const finalResult = await result;
  const usage = calculateCost(
    MODEL_ID,
    finalResult.totalUsage.inputTokens,
    finalResult.totalUsage.outputTokens,
  );

  return {
    runId: crypto.randomUUID(),
    sessionId: crypto.randomUUID(),
    text: finalResult.text,
    steps: finalResult.steps.length,
    usage,
    durationMs: Date.now() - startTime,
    variant: config.variant ?? "default",
  };
}
```

Use `streamText` when: user-facing UI, responses > 5 seconds, long agent loops. Use `generateText` when:
batch processing, evals, no UI.

## Tool Result Validation

Tool results from external APIs can contain prompt injection payloads (indirect injection). Validate all tool
outputs before returning them to the model:

```typescript
// src/tools/validated-tool.ts
import { tool } from "ai";
import { z } from "zod";

export function defineValidatedTool<TIn extends z.ZodTypeAny, TOut extends z.ZodTypeAny>(config: {
  description: string;
  inputSchema: TIn;
  outputSchema: TOut;
  execute: (input: z.infer<TIn>) => Promise<unknown>;
}) {
  return tool({
    description: config.description,
    inputSchema: config.inputSchema,
    execute: async (input: z.infer<TIn>) => {
      try {
        const raw = await config.execute(input);
        const validated = config.outputSchema.safeParse(raw);
        if (!validated.success) {
          return { ok: false, error: "Tool returned unexpected data format" };
        }
        return { ok: true, data: validated.data };
      } catch (error) {
        return { ok: false, error: String(error) };
      }
    },
  });
}
```

This ensures external API responses are schema-validated before the model sees them, preventing injection via
crafted tool outputs.

## Context Window Management

Agent loops accumulate messages. Without management, context windows overflow:

```typescript
// src/context.ts
import type { ModelMessage } from "ai";

const MODEL_CONTEXT_LIMITS: Record<string, number> = {
  "claude-sonnet-4-6": 200_000,
  "claude-opus-4-7": 200_000,
  "claude-haiku-4-5": 200_000,
  "gpt-4o": 128_000,
};

export function estimateTokens(messages: ModelMessage[]): number {
  return Math.ceil(JSON.stringify(messages).length / 4);
}

export function shouldTruncate(messages: ModelMessage[], model: string): boolean {
  const limit = MODEL_CONTEXT_LIMITS[model] ?? 128_000;
  return estimateTokens(messages) > limit * 0.8;
}

export function truncateMessages(messages: ModelMessage[]): ModelMessage[] {
  if (messages.length <= 4) return messages;
  const system = messages.filter((m) => m.role === "system");
  const first = messages.find((m) => m.role === "user");
  const recent = messages.slice(-6);
  return [...system, ...(first ? [first] : []), ...recent];
}
```

### Prompt Caching

System prompts and tool definitions are identical across calls — cache them to reduce cost and rate limit
pressure. The AI SDK supports provider-specific caching. For Anthropic:

```typescript
const result = await generateText({
  model: anthropic("claude-sonnet-4-6", { cacheControl: true }),
  system: config.system,  // cached after first call
  messages,
  tools,  // cached after first call
  stopWhen: stepCountIs(config.maxSteps),
});
```

Cached tokens do not count toward input TPM limits on most providers.

## Graceful Degradation

### Per-Tool Timeouts

```typescript
export function withTimeout<T>(fn: () => Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    fn(),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(`Tool timed out after ${ms}ms`)), ms),
    ),
  ]);
}
```

### Circuit Breaker

Track tool failures. After repeated failures, skip the tool and tell the model it's unavailable:

```typescript
// src/circuit-breaker.ts
export class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: "closed" | "open" | "half-open" = "closed";

  constructor(
    private threshold: number = 3,
    private resetMs: number = 60_000,
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.lastFailure > this.resetMs) {
        this.state = "half-open";
      } else {
        throw new Error("Circuit open — tool temporarily unavailable");
      }
    }

    try {
      const result = await fn();
      this.failures = 0;
      this.state = "closed";
      return result;
    } catch (error) {
      this.failures++;
      this.lastFailure = Date.now();
      if (this.failures >= this.threshold) this.state = "open";
      throw error;
    }
  }
}
```

## Configuration Management

Never hardcode model IDs, cost budgets, or prompts. Load from environment with validation:

```typescript
// src/config.ts
import { z } from "zod";

const AgentEnvConfig = z.object({
  MODEL_ID: z.string().default("claude-sonnet-4-6"),
  MAX_STEPS: z.coerce.number().int().min(1).max(100).default(25),
  MAX_COST_USD: z.coerce.number().min(0).default(2.0),
  EVAL_COST_CEILING_USD: z.coerce.number().min(0).default(1.0),
  CONCURRENCY_LIMIT: z.coerce.number().int().min(1).max(50).default(5),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export const config = AgentEnvConfig.parse(process.env);
```

Store system prompts as separate files, not inline strings:

```text
prompts/
├── v1.md    # Initial prompt
├── v2.md    # After first round of eval feedback
└── current  # Symlink to active version
```

### Prompt Versioning

Track which prompt version produced each result:

```typescript
import { readFileSync, realpathSync } from "node:fs";

export function loadPrompt(path = "./prompts/current"): { content: string; version: string } {
  const resolved = realpathSync(path);
  const version = resolved.split("/").pop()?.replace(".md", "") ?? "unknown";
  return { content: readFileSync(resolved, "utf-8"), version };
}
```

Include `promptVersion` in `AgentResult` and feedback records. This enables:

- Correlating quality metrics with specific prompt versions
- Rolling back by changing the symlink
- A/B testing prompt versions using the experiment framework

## Session Security

Generate session IDs server-side only. Never accept them from untrusted clients:

```typescript
// src/api.ts — secure session management
const sessions = new Map<string, { createdAt: number; userId?: string }>();

function createSession(userId?: string): string {
  const id = crypto.randomUUID();
  sessions.set(id, { createdAt: Date.now(), userId });
  return id;
}

function validateSession(sessionId: string): boolean {
  return sessions.has(sessionId);
}
```

If clients need to resume sessions, validate ownership via authentication — never trust client-supplied
session IDs directly.

## Multi-Agent Patterns

### Orchestrator-Worker

```typescript
import { generateText, Output, stepCountIs } from "ai";
import { z } from "zod";

const TaskPlan = z.object({
  subtasks: z.array(z.object({
    description: z.string(),
    tools: z.array(z.string()),
  })),
});

export async function orchestrate(prompt: string) {
  const { output: plan } = await generateText({
    model: createModel(),
    output: Output.object({ schema: TaskPlan }),
    prompt: `Break this task into independent subtasks: ${prompt}`,
  });

  const results = await Promise.all(
    plan.subtasks.map((subtask) =>
      runAgent(subtask.description, {
        system: `You are a specialist. Complete this subtask: ${subtask.description}`,
        maxSteps: 10,
      }),
    ),
  );

  const synthesis = await generateText({
    model: createModel(),
    prompt: `Synthesize these subtask results into a final answer:
${results.map((r, i) => `Subtask ${i + 1}: ${r.text}`).join("\n\n")}`,
  });

  return synthesis.text;
}
```

### Evaluator-Optimizer

```typescript
export async function refine(prompt: string, maxRounds = 3) {
  let draft = await runAgent(prompt, { system: "Produce a draft.", maxSteps: 15 });

  for (let round = 0; round < maxRounds; round++) {
    const { output: evaluation } = await generateText({
      model: createModel(),
      output: Output.object({
        schema: z.object({
          score: z.number().min(0).max(1),
          feedback: z.string(),
          done: z.boolean(),
        }),
      }),
      prompt: `Evaluate this draft:
${draft.text}

Original request: ${prompt}
Score 0-1. If score >= 0.9, set done=true. Otherwise provide specific feedback.`,
    });

    if (evaluation.done) break;

    draft = await runAgent(
      `Improve this draft based on feedback:\n\nDraft: ${draft.text}\n\nFeedback: ${evaluation.feedback}`,
      { system: "Revise the draft addressing all feedback points.", maxSteps: 15 },
    );
  }

  return draft.text;
}
```

## Structured Output (AI SDK v6)

In v6, use `Output.object()` with `generateText` instead of deprecated `generateObject`:

```typescript
import { generateText, Output } from "ai";
import { z } from "zod";

const RecipeSchema = z.object({
  name: z.string(),
  ingredients: z.array(z.object({ name: z.string(), amount: z.string() })),
  steps: z.array(z.string()),
});

const { output } = await generateText({
  model: createModel(),
  output: Output.object({ schema: RecipeSchema }),
  prompt: "Generate a lasagna recipe.",
});
// output is fully typed as { name: string, ingredients: ..., steps: ... }
```

### Retry on Validation Failure

```typescript
export async function generateValidated<T extends z.ZodTypeAny>(
  model: LanguageModel,
  schema: T,
  prompt: string,
  maxRetries = 3,
): Promise<z.infer<T>> {
  let lastError: string | null = null;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const augmentedPrompt = lastError
      ? `${prompt}\n\nPrevious attempt failed validation: ${lastError}. Fix these issues.`
      : prompt;
    try {
      const { output } = await generateText({
        model,
        output: Output.object({ schema }),
        prompt: augmentedPrompt,
      });
      return output;
    } catch (error) {
      lastError = String(error);
    }
  }

  throw new Error(`Failed to generate valid object after ${maxRetries} attempts. Last error: ${lastError}`);
}
```

## OpenTelemetry Setup

```typescript
// src/telemetry.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { Resource } from "@opentelemetry/resources";
import { ATTR_SERVICE_NAME } from "@opentelemetry/semantic-conventions";
import { trace } from "@opentelemetry/api";

export function initTelemetry(serviceName: string) {
  const sdk = new NodeSDK({
    resource: new Resource({ [ATTR_SERVICE_NAME]: serviceName }),
    traceExporter: new OTLPTraceExporter(),
  });
  sdk.start();
  return sdk;
}

const tracer = trace.getTracer("agent");

export function withSpan<T>(name: string, fn: () => Promise<T>): Promise<T> {
  return tracer.startActiveSpan(name, async (span) => {
    try {
      const result = await fn();
      span.setStatus({ code: 1 });
      return result;
    } catch (error) {
      span.setStatus({ code: 2, message: String(error) });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

## Project Templates

### package.json

```json
{
  "name": "<agent-name>",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "start": "bun src/index.ts",
    "eval": "bun vitest run --config vitest.config.ts",
    "eval:watch": "bun vitest --config vitest.config.ts",
    "check": "bunx biome check .",
    "check:fix": "bunx biome check --write ."
  },
  "dependencies": {
    "ai": "^6.0.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@biomejs/biome": "^2.0.0",
    "vitest": "^4.0.0"
  }
}
```

`@types/bun` is installed by `bun init` — do not add manually.

Provider packages:

| Provider | Package |
| -------- | ------- |
| Vertex AI (Claude) | `@ai-sdk/google-vertex` |
| Anthropic direct | `@ai-sdk/anthropic` |
| OpenAI | `@ai-sdk/openai` |
| AWS Bedrock | `@ai-sdk/amazon-bedrock` |
| Ollama (local) | `ollama-ai-provider` |

### tsconfig.json

Use bun's defaults as base, add strict settings:

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "Preserve",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "noEmit": true,
    "verbatimModuleSyntax": true,
    "types": ["bun-types"]
  },
  "include": ["src/**/*", "evals/**/*", "scripts/**/*"]
}
```

Do not set `rootDir` — evals and scripts live outside `src/`. Do not set `outDir`/`declaration`/`sourceMap` —
bun runs TypeScript directly without compilation.

### biome.json

Generate config with `bunx biome init`, then customize. If biome version mismatches the schema, run
`bunx biome migrate --write` to auto-update:

```json
{
  "assist": {
    "actions": {
      "source": { "organizeImports": "on" }
    }
  },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 120
  }
}
```

See also:

- [REFERENCE-EVALS.md](REFERENCE-EVALS.md) — scorers, cost tracking, trajectory evaluation, regression detection
- [REFERENCE-FEEDBACK.md](REFERENCE-FEEDBACK.md) — feedback collection, A/B testing, experiment runner
