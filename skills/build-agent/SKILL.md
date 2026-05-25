---
name: build-agent
description: >
  Build TypeScript AI agents with eval framework, provider abstraction, and production guardrails. Uses Vercel AI SDK
  for model-agnostic tool-use loops, Zod for structured outputs, Vitest for evals, and OpenTelemetry for observability.
  Covers prompt caching, extended thinking, RAG pipelines, structured output enforcement, tool annotations, seed tool
  calls, toolcall regression evals, and duplicate call detection. Use when the user wants to build an agent, create an
  AI assistant, implement tool-use, or scaffold an agentic system — even if they just say "build a bot", "add AI to
  this", or "make it agentic."
compatibility: Requires Bun (runtime + package manager), git.
allowed-tools:
  - Bash(bun *)
  - Bash(bunx *)
  - Bash(git *)
  - Bash(ls *)
  - Read
  - Write
  - Edit
  - Agent
metadata:
  author: rsturla
  version: 1.0.0
---

# Build Agent

Build TypeScript AI agents that are provider-agnostic, eval-tested, and production-ready.

## Quick Start

```text
User: /build-agent
Agent: asks agent purpose, tools needed, provider preference
Agent: scaffolds project with agent loop, tools, evals, observability
User: reviews, iterates
```

## Architecture Principles

Follow [Anthropic's guidance](https://www.anthropic.com/research/building-effective-agents): start simple, add
complexity only when needed.

| Level | Pattern | When |
| ----- | ------- | ---- |
| 1 | Single LLM call | Straightforward tasks with no tool use |
| 2 | Prompt chain | Fixed sequential steps with validation gates |
| 3 | Router | Distinct input categories need different handling |
| 4 | Parallelization | Independent subtasks or voting |
| 5 | Agent loop | Open-ended problems, unknown number of steps |
| 6 | Multi-agent | Specialized capabilities need delegation |

**Default to level 5** (agent loop) unless the user specifies otherwise. Most "build an agent" requests need a
tool-use loop.

## Process

1. Gather requirements (purpose, tools, provider)
2. Scaffold project structure
3. Write system prompt
4. Implement agent core (provider, tools, loop)
5. Add eval framework
6. Add feedback collection and A/B testing
7. Add observability
8. Present for review

## Step 1: Gather Requirements

Ask in order:

1. **Agent purpose** — what problem does it solve?
2. **Tools needed** — what actions can the agent take? (file operations, API calls, database queries, web search, etc.)
3. **Provider** — which LLM provider? Default: Vertex AI (Google Cloud). Alternatives: OpenAI, Anthropic direct, AWS
   Bedrock, Ollama (local)
4. **Structured output** — does the agent need typed responses? (default: yes, use Zod)
5. **Max iterations** — hard cap on agent loop steps (default: 25)

## Step 2: Scaffold Project

```text
<agent-name>/
├── src/
│   ├── index.ts              # Entry point
│   ├── agent.ts              # Agent loop + configuration
│   ├── provider.ts           # Model provider setup
│   ├── cost.ts               # Token pricing + cost calculation
│   ├── feedback.ts           # Feedback collection + storage
│   ├── experiment.ts         # A/B testing framework
│   ├── tools/
│   │   ├── index.ts          # Tool registry (re-exports all tools)
│   │   └── <tool-name>.ts    # One file per tool
│   └── types.ts              # Shared types and Zod schemas
├── evals/
│   ├── setup.ts              # Eval helpers and fixtures
│   ├── scorers.ts            # Custom scoring functions
│   └── <scenario>.eval.ts    # One file per eval scenario
├── scripts/
│   ├── feedback-report.ts    # Feedback summary CLI
│   ├── feedback-to-evals.ts  # Mine corrections into eval cases
│   └── run-experiment.ts     # Offline A/B experiment runner
├── data/                     # Gitignored
│   ├── feedback.jsonl        # Collected feedback
│   └── experiments/          # Experiment results
├── package.json
├── tsconfig.json
├── vitest.config.ts          # Separate config for evals
└── biome.json
```

Initialize with:

```bash
bun init
# Remove bun init artifacts (index.ts, README.md) — project uses src/ structure
rm -f index.ts README.md
bun add ai @ai-sdk/google-vertex zod
bun add -d vitest @biomejs/biome
```

Add provider packages only for providers the user needs. Always install `ai` (Vercel AI SDK core).

## Step 3: Write System Prompt

System prompt defines agent behavior. Write it before code — the prompt IS the spec.

### Structure

```text
[Role] — who the agent is, one sentence
[Task] — what it does, scope boundaries
[Rules] — hard constraints (always/never)
[Output format] — how to structure responses
[Examples] — 2-3 input/output pairs showing ideal behavior
```

### Principles

- **Be specific** — "You are a weather assistant that provides current conditions and 5-day forecasts for cities
  worldwide" beats "You are a helpful assistant"
- **State what, not how** — describe desired behavior, not implementation. The model picks the approach
- **Set boundaries** — what the agent should refuse. "If the user asks about topics outside weather, say you can
  only help with weather queries"
- **Include examples** — few-shot examples in the system prompt dramatically improve tool selection and output
  quality. Show the exact tool calls you expect for representative inputs
- **Keep it short** — long system prompts dilute focus. Under 500 words. If you need more, the agent is doing
  too much — split into multiple agents
- **Use imperative mood** — "Return temperatures in Celsius" not "You should return temperatures in Celsius"

### Example

```typescript
const SYSTEM_PROMPT = `You are a weather assistant. You provide current conditions and forecasts for cities worldwide.

Rules:
- Always include temperature, humidity, and wind speed
- Use metric units (Celsius, km/h) unless the user requests imperial
- If a city is not found, say so clearly — do not guess
- Do not answer questions unrelated to weather

When the user asks about weather, call the weather tool with the city name. If the user mentions multiple cities,
call the tool once per city.

Examples:

User: "What's the weather in London?"
→ Call weather tool with city="London"
→ Respond with current conditions including temperature, humidity, wind

User: "Compare Tokyo and Sydney"
→ Call weather tool with city="Tokyo"
→ Call weather tool with city="Sydney"
→ Respond with side-by-side comparison`;
```

### Anti-Patterns

- **Vague role** — "You are a helpful AI" gives the model no constraints
- **Over-specification** — step-by-step algorithms in the prompt. Let the model reason
- **Contradictory rules** — "be concise" + "explain your reasoning in detail"
- **No examples** — the model guesses how to use tools without seeing expected patterns
- **Including implementation details** — API keys, endpoints, internal logic. These belong in tool code, not
  the prompt

### Iterating on Prompts

1. Write initial prompt based on requirements
2. Run evals — identify failure modes
3. Add rules or examples that address failures
4. Re-run evals — confirm improvement without regression
5. A/B test significant changes against baseline

Never tune prompts without evals. You'll fix one case and break three others.

## Step 4: Implement Agent Core

### Provider Abstraction

Vercel AI SDK handles provider abstraction. `src/provider.ts` is the only file that changes when switching
providers:

```typescript
// src/provider.ts
import { vertex } from "@ai-sdk/google-vertex";
export const MODEL_ID = "claude-sonnet-4-6";
export function createModel() { return vertex(MODEL_ID); }
```

Swap provider by changing import and model ID. See REFERENCE.md for all provider examples.

### Tool Definition

One file per tool, Zod schema for inputs. Key rules:

- Bound everything: `.min()`, `.max()`, `.int()`
- Enums over open strings
- `.describe()` on each field — model reads these
- Return structured errors `{ error: "..." }`, never throw
- Re-export from `src/tools/index.ts`

See REFERENCE.md for full tool example and reusable wrapper.

### Cost Tracking

**Mandatory.** `src/cost.ts` maps model IDs to per-token pricing. Every `AgentResult` includes `usage.costUsd`.
No silent token consumption — if model pricing is unknown, log a warning.

See REFERENCE.md for full `calculateCost` implementation and pricing table.

### Agent Loop

```typescript
// src/agent.ts
import { generateText, stepCountIs, type StepResult } from "ai";
import type { ModelMessage } from "ai";
import { createModel, MODEL_ID } from "./provider.js";
import { calculateCost, type UsageRecord } from "./cost.js";
import * as tools from "./tools/index.js";

export interface AgentConfig {
  system: string;
  maxSteps: number;
  variant?: string;
}

export interface AgentResult {
  runId: string;
  sessionId: string;
  text: string;
  stepCount: number;
  steps: StepResult<typeof tools>[];
  usage: UsageRecord;
  durationMs: number;
  variant: string;
}

export async function runAgent(
  prompt: string,
  config: AgentConfig,
  sessionId?: string,
): Promise<AgentResult> {
  const messages: ModelMessage[] = [{ role: "user", content: prompt }];
  const startTime = Date.now();

  const result = await generateText({
    model: createModel(),
    system: config.system,
    messages,
    tools,
    stopWhen: stepCountIs(config.maxSteps),
  });

  const usage = calculateCost(
    MODEL_ID,
    result.totalUsage.inputTokens ?? 0,
    result.totalUsage.outputTokens ?? 0,
  );

  return {
    runId: crypto.randomUUID(),
    sessionId: sessionId ?? crypto.randomUUID(),
    text: result.text,
    stepCount: result.steps.length,
    steps: result.steps,
    usage,
    durationMs: Date.now() - startTime,
    variant: config.variant ?? "default",
  };
}
```

AI SDK `stopWhen` handles the loop. Use manual loop from REFERENCE.md only for custom per-step logic (cost
budgets, loop detection, conditional tool filtering).

### Safety Controls

| Control | Default | Purpose |
| ------- | ------- | ------- |
| `stopWhen` | `stepCountIs(25)` | Hard cap on iterations |
| Wall-clock timeout | 300s | Prevent runaway agents |
| Cost budget | $2.00/run | Abort if `usage.costUsd` exceeds limit |
| Loop detection | Fingerprint last 3 tool calls | Catch repetitive behavior |

## Step 5: Eval Framework

Evals are **not optional**. Every agent ships with evals. Every eval records cost.

### Eval Structure

Three layers, cheapest first:

| Layer | Approach | Cost | Use For |
| ----- | -------- | ---- | ------- |
| Deterministic | Regex, exact match, JSON schema | Free | Format compliance, required fields |
| Semantic | Embedding distance | Low | Meaning preservation |
| LLM-as-judge | Model grades model output | Medium | Tone, helpfulness, correctness |

### Eval Config

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["evals/**/*.eval.ts"],
    testTimeout: 120_000,
    maxConcurrency: 3,
  },
});
```

### Writing Evals

Every eval calls `recordEval()` with score, tokens, cost, and duration. `afterAll` prints a cost report and
checks against `EVAL_COST_CEILING_USD`. Example:

```typescript
// evals/weather.eval.ts
import { describe, it, expect } from "vitest";
import { runAgent } from "../src/agent.js";
import { containsAll } from "./scorers.js";
import { recordEval } from "./setup.js";

describe("weather agent", () => {
  it("returns temperature for valid city", async () => {
    const result = await runAgent("What is the weather in London?",
      { system: "You are a helpful weather assistant.", maxSteps: 10 },
    );
    const score = containsAll(["london", "temperature"])(result.text);
    recordEval({
      name: "weather/valid-city",
      score: score.score,
      tokens: result.usage.totalTokens,
      costUsd: result.usage.costUsd,
      durationMs: result.durationMs,
    });
    expect(score.score).toBe(1);
  });
});
```

Run with `bun vitest run --config vitest.config.ts`. Set cost ceiling for suites (e.g., $1.00/run).

See [REFERENCE-EVALS.md](REFERENCE-EVALS.md) for scorer implementations (deterministic, LLM-as-judge), eval
cost tracking setup, trajectory evaluation, and regression detection.

## Step 6: Feedback and A/B Testing

Evals measure offline quality. Production feedback measures real-world quality. A/B tests compare alternatives.
All three feed the iteration loop.

### Feedback Collection

Collect structured feedback **after deployment**. Every `AgentResult` includes `runId` and `sessionId` — clients
submit feedback asynchronously (seconds to hours after response).

Two signal types:

| Type | Source | Signal |
| ---- | ------ | ------ |
| Explicit | Thumbs up/down, written correction | Direct quality signal (high precision) |
| Implicit | Retries, abandonment, time-to-next-action | Indirect signal (high volume) |

Storage is pluggable — `FeedbackStore` interface with JSONL backend for dev, database for production. Expose
feedback submission via HTTP endpoint so any client can report.

**Corrections are the highest-value signal** — negative feedback with a correction becomes a future eval case.
Run `bun scripts/feedback-to-evals.ts` to mine corrections into `evals/fixtures/from-feedback.json`.

See [REFERENCE-FEEDBACK.md](REFERENCE-FEEDBACK.md) for full FeedbackSchema, storage backends, API endpoint,
implicit signal detection, feedback summary/reporting, and feedback-to-eval pipeline.

### A/B Testing

Two modes:

- **Offline** — run same test corpus against different configs, score with LLM-as-judge, compare. Best for
  prompt/model changes before deployment
- **Online** — split live traffic between variants (session-sticky), compare via production feedback after
  sufficient sample size (~100 per variant)

```bash
bun scripts/run-experiment.ts
```

See [REFERENCE-FEEDBACK.md](REFERENCE-FEEDBACK.md) for full experiment runner (offline + online), variant
assignment, cost-quality tradeoff analysis, and graduation criteria.

### Iteration Workflow

```text
1. Write evals → establish baseline scores
2. Deploy agent → collect feedback
3. Mine negative feedback → new eval cases
4. A/B test changes → compare against baseline
5. Ship winner → update baseline
6. Repeat
```

## Step 7: Observability

Use OpenTelemetry. The Vercel AI SDK has built-in telemetry support:

```typescript
// src/agent.ts — add to generateText call
const result = await generateText({
  model: createModel(),
  system: config.system,
  messages,
  tools,
  stopWhen: stepCountIs(config.maxSteps),
  experimental_telemetry: {
    isEnabled: true,
    metadata: { agentName: "weather-agent" },
  },
});
```

For custom spans around tool execution, see REFERENCE.md.

## Step 8: Review

Present the complete file tree and ask user to review before any further work. Suggest:

- Run evals: `bun vitest run`
- Run experiment: `bun scripts/run-experiment.ts`
- Add more tools
- Tune system prompt based on eval failures and feedback
- Add CI pipeline with `/scaffold-repo`

## Gotchas

- **Bun, not npm** — all commands use `bun`. Never `npm install`, `npx`, `yarn`, or `pnpm`
- **Provider abstraction is a single file** — `src/provider.ts`. Do NOT build custom abstraction layers on top of the
  AI SDK. The SDK already handles this
- **`stopWhen` vs manual loop** — use `stopWhen: stepCountIs(n)` (AI SDK built-in) by default. Only write a
  manual while-loop when you need custom per-step logic (cost tracking, conditional tool filtering, step-level
  logging). See REFERENCE.md for the manual loop pattern
- **Tool errors should return data, not throw** — if a tool throws, the agent loop may terminate. Return
  `{ error: "message" }` so the model can recover
- **Eval timeout** — LLM calls are slow. Set `testTimeout: 120_000` minimum. Use `pool: "threads"` with
  `maxThreads: 3` for parallel execution without overwhelming rate limits
- **Zod `.describe()` matters** — the model sees field descriptions when deciding how to call tools. Missing
  descriptions = worse tool use
- **Vertex AI provider** — when user picks Vertex AI, use `@ai-sdk/google-vertex` package and set
  `GOOGLE_VERTEX_PROJECT` and `GOOGLE_VERTEX_LOCATION` env vars. Model IDs: `claude-sonnet-4-6`,
  `claude-opus-4-7`, `claude-haiku-4-5` (no date suffixes)
- **Cost tracking is mandatory** — every `AgentResult` must include `usage.costUsd`. Unknown models throw, not
  silently report `$0.00`. Add model to `PRICING` table before use
- **Wrap all LLM calls with retry** — 429/529/500 are routine. Use `withRetry()` from REFERENCE.md. Never let
  a single API failure crash the agent
- **Validate tool results** — external APIs can return prompt injection payloads (indirect injection). Schema-
  validate all tool outputs before the model sees them. See `defineValidatedTool` in REFERENCE.md
- **Streaming for UIs** — use `streamText` for user-facing agents, `generateText` for batch/evals. See
  REFERENCE.md streaming section
- **Feedback corrections need human review** — never auto-promote user corrections to eval ground truth.
  Unauthenticated users can submit poisoned corrections. Review before promoting
- **Feedback endpoint needs auth** — rate limiting, runId validation, and API key checks. See REFERENCE-FEEDBACK.md
  for the secured API pattern
- **Session IDs are server-generated** — never accept session IDs from untrusted clients. Session fixation risk
- **Run evals 3-5 times** — LLM outputs are non-deterministic. Assert on mean scores, not single runs. See
  REFERENCE-EVALS.md statistical rigor section
- **Safety evals are not optional** — every agent needs prompt injection resistance, PII leakage, and refusal
  accuracy evals. See REFERENCE-EVALS.md safety evals section
- **Feedback data is gold** — negative feedback with corrections becomes eval cases. Gitignore `data/` but back it
  up. Never delete feedback without extracting insights first
- **A/B tests need identical inputs** — compare variants on same prompts. Random inputs invalidate comparison.
  Build a fixed test corpus from real usage and negative feedback
- **System prompt is the spec** — write it before code. If you can't describe agent behavior in under 500 words,
  the agent is doing too much
- **Few-shot examples in system prompt** — showing 2-3 expected tool-call patterns improves tool selection more
  than any amount of descriptive text
- **Don't over-engineer** — start with one agent, one loop, concrete tools. Add multi-agent orchestration, routing,
  and workflows only when single-agent eval scores plateau
- **Eval before prompt tuning** — write evals first, then iterate on system prompt. Without evals you are guessing

See also:

- [REFERENCE.md](REFERENCE.md) — extended thinking, tool annotations, seed tool calls, prompt caching, prompt
  templating, structured output enforcement, retry/rate limits, streaming, tool validation, context management,
  graceful degradation, config/prompt versioning, session security, manual loop, guardrails, multi-agent, OTel
  cardinality, resource labels, payload gating, templates
- [REFERENCE-EVALS.md](REFERENCE-EVALS.md) — scorers, toolcall regression evals, duplicate call detection,
  hallucination detection, safety evals, statistical rigor, scorer composition, CI integration, regression detection
- [REFERENCE-FEEDBACK.md](REFERENCE-FEEDBACK.md) — FeedbackSchema, storage backends, secured API endpoint,
  implicit signals, human review gate, feedback-to-evals, A/B testing
- [REFERENCE-RAG.md](REFERENCE-RAG.md) — embedding, chunking, vector store, retrieval, agent integration, namespace
  filtering, RAG evals, cost considerations
