# Build Agent — Feedback and A/B Testing Reference

Production feedback collection, implicit signals, feedback-to-evals pipeline, and A/B testing (offline + online).

## Feedback Schema

Define in `src/types.ts` — shared across feedback, experiments, and evals:

```typescript
// src/types.ts
import { z } from "zod";

export const FeedbackSchema = z.object({
  runId: z.string().uuid(),
  sessionId: z.string().uuid(),
  timestamp: z.string(),
  rating: z.enum(["positive", "negative"]).optional(),
  comment: z.string().max(2000).optional(),
  correction: z.string().max(5000).optional(),
  signals: z.object({
    retried: z.boolean().default(false),
    abandoned: z.boolean().default(false),
    responseTimeMs: z.number().optional(),
    stepsUsed: z.number().optional(),
  }).default({ retried: false, abandoned: false }),
  input: z.string(),
  output: z.string(),
  variant: z.string().default("default"),
  promptVersion: z.string().default("unknown"),
  usage: z.object({
    totalTokens: z.number(),
    costUsd: z.number(),
  }),
});

export type Feedback = z.infer<typeof FeedbackSchema>;
```

## Feedback Storage

### Interface

```typescript
// src/feedback.ts
import { type Feedback, FeedbackSchema } from "./types.js";

export interface FeedbackStore {
  record(feedback: Feedback): Promise<void>;
  load(filter?: FeedbackFilter): Promise<Feedback[]>;
}

export interface FeedbackFilter {
  rating?: "positive" | "negative";
  variant?: string;
  since?: Date;
  hasCorrection?: boolean;
}
```

### JSONL Backend

Local dev and single-instance deployments:

```typescript
// src/feedback-jsonl.ts
import { appendFile, readFile, mkdir } from "node:fs/promises";
import { existsSync } from "node:fs";
import type { FeedbackStore, FeedbackFilter, Feedback } from "./feedback.js";
import { FeedbackSchema } from "./types.js";

export class JsonlFeedbackStore implements FeedbackStore {
  constructor(private path = "./data/feedback.jsonl") {}

  async record(feedback: Feedback): Promise<void> {
    await mkdir("./data", { recursive: true });
    await appendFile(this.path, JSON.stringify(feedback) + "\n");
  }

  async load(filter?: FeedbackFilter): Promise<Feedback[]> {
    if (!existsSync(this.path)) return [];
    const lines = (await readFile(this.path, "utf-8")).trim().split("\n").filter(Boolean);
    let results = lines.map((line) => FeedbackSchema.parse(JSON.parse(line)));

    if (filter?.rating) results = results.filter((f) => f.rating === filter.rating);
    if (filter?.variant) results = results.filter((f) => f.variant === filter.variant);
    if (filter?.since) {
      const since = filter.since;
      results = results.filter((f) => new Date(f.timestamp) >= since);
    }
    if (filter?.hasCorrection) results = results.filter((f) => !!f.correction);

    return results;
  }
}
```

For production multi-instance: implement `FeedbackStore` interface with Postgres, SQLite, etc.

## Feedback API Endpoint

Agent response includes `runId` — client submits feedback asynchronously. API must include input validation,
authentication, and rate limiting:

```typescript
// src/api.ts
import { FeedbackSchema } from "./types.js";
import type { FeedbackStore } from "./feedback.js";
import { runAgent, type AgentConfig } from "./agent.js";
import { validateInput, validateOutput } from "./guardrails.js";
import { withRetry } from "./retry.js";

interface ApiConfig {
  feedbackStore: FeedbackStore;
  agentConfig: AgentConfig;
  apiKeyHeader?: string;
  rateLimitPerMinute?: number;
}

const runLog = new Set<string>();
const rateLimit = new Map<string, number[]>();

function checkRateLimit(clientId: string, maxPerMinute: number): boolean {
  const now = Date.now();
  const window = rateLimit.get(clientId)?.filter((t) => now - t < 60_000) ?? [];
  if (window.length >= maxPerMinute) return false;
  window.push(now);
  rateLimit.set(clientId, window);
  return true;
}

export function createApi(config: ApiConfig) {
  const { feedbackStore, agentConfig, rateLimitPerMinute = 30 } = config;

  return {
    async handleRun(req: Request): Promise<Response> {
      const apiKey = req.headers.get("authorization")?.replace("Bearer ", "");
      if (config.apiKeyHeader && !apiKey) {
        return Response.json({ error: "Missing authorization" }, { status: 401 });
      }

      if (!checkRateLimit(apiKey ?? "anonymous", rateLimitPerMinute)) {
        return Response.json({ error: "Rate limit exceeded" }, { status: 429 });
      }

      let body: { prompt: string };
      try { body = (await req.json()) as { prompt: string }; }
      catch { return Response.json({ error: "Invalid JSON" }, { status: 400 }); }

      const inputCheck = validateInput(body.prompt);
      if (!inputCheck.ok) {
        return Response.json({ error: inputCheck.reason }, { status: 400 });
      }

      const result = await withRetry(() => runAgent(body.prompt, agentConfig));

      const outputCheck = validateOutput(result.text);
      if (!outputCheck.ok) {
        return Response.json({
          runId: result.runId,
          sessionId: result.sessionId,
          text: "I cannot provide that response.",
          usage: { totalTokens: result.usage.totalTokens, costUsd: result.usage.costUsd },
        });
      }

      runLog.add(result.runId);

      return Response.json({
        runId: result.runId,
        sessionId: result.sessionId,
        text: result.text,
        usage: { totalTokens: result.usage.totalTokens, costUsd: result.usage.costUsd },
      });
    },

    async handleFeedback(req: Request): Promise<Response> {
      let body: unknown;
      try { body = await req.json(); }
      catch { return Response.json({ error: "Invalid JSON" }, { status: 400 }); }

      const parsed = FeedbackSchema.safeParse(body);
      if (!parsed.success) {
        return Response.json({ error: parsed.error.issues }, { status: 400 });
      }

      if (!runLog.has(parsed.data.runId)) {
        return Response.json({ error: "Unknown runId" }, { status: 404 });
      }

      await feedbackStore.record(parsed.data);
      return Response.json({ ok: true });
    },
  };
}
```

Client flow:

```text
1. POST /run { prompt, sessionId }  ->  { runId, sessionId, text, usage }
2. User reads response, decides quality
3. POST /feedback { runId, sessionId, rating, correction?, comment?, ... }
```

## Implicit Signal Detection

Track behavioral signals that correlate with quality:

```typescript
// src/signals.ts
export class SessionTracker {
  private sessions = new Map<string, { inputs: string[]; lastActivity: number }>();

  recordQuery(sessionId: string, input: string): void {
    const session = this.sessions.get(sessionId) ?? { inputs: [], lastActivity: 0 };
    session.inputs.push(input);
    session.lastActivity = Date.now();
    this.sessions.set(sessionId, session);
  }

  detectRetry(sessionId: string, input: string, threshold = 0.8): boolean {
    const session = this.sessions.get(sessionId);
    if (!session || session.inputs.length < 2) return false;
    const previous = session.inputs[session.inputs.length - 2];
    return similarity(previous, input) > threshold;
  }

  detectAbandonment(sessionId: string, timeoutMs = 300_000): boolean {
    const session = this.sessions.get(sessionId);
    if (!session) return false;
    return Date.now() - session.lastActivity > timeoutMs;
  }
}

function similarity(a: string, b: string): number {
  const wordsA = new Set(a.toLowerCase().split(/\s+/));
  const wordsB = new Set(b.toLowerCase().split(/\s+/));
  const intersection = new Set([...wordsA].filter((w) => wordsB.has(w)));
  const union = new Set([...wordsA, ...wordsB]);
  return intersection.size / union.size;
}
```

| Signal | Meaning | How to Record |
| ------ | ------- | ------------- |
| **Retry** | User asks same question again | `signals.retried = true` |
| **Abandonment** | User leaves without follow-up | `signals.abandoned = true` after timeout |
| **Fast follow-up** | User immediately asks clarifying question | Likely incomplete answer |
| **Correction** | User provides better answer | `correction` field — highest value signal |
| **Thumbs up/down** | Explicit rating | `rating` field |

## Feedback Summary

```typescript
// src/feedback.ts (continued)

export interface FeedbackSummary {
  total: number;
  rated: number;
  positive: number;
  negative: number;
  positiveRate: number;
  totalCostUsd: number;
  avgCostUsd: number;
  withCorrections: number;
  retryRate: number;
  abandonRate: number;
  byVariant: Record<string, VariantFeedback>;
}

interface VariantFeedback {
  total: number;
  positive: number;
  positiveRate: number;
  avgCostUsd: number;
  retryRate: number;
}

export function summarizeFeedback(feedback: Feedback[]): FeedbackSummary {
  const total = feedback.length;
  const rated = feedback.filter((f) => f.rating).length;
  const positive = feedback.filter((f) => f.rating === "positive").length;
  const negative = feedback.filter((f) => f.rating === "negative").length;
  const totalCostUsd = feedback.reduce((sum, f) => sum + f.usage.costUsd, 0);
  const withCorrections = feedback.filter((f) => f.correction).length;
  const retries = feedback.filter((f) => f.signals.retried).length;
  const abandons = feedback.filter((f) => f.signals.abandoned).length;

  const byVariant: Record<string, VariantFeedback> = {};
  for (const f of feedback) {
    const v = byVariant[f.variant] ?? { total: 0, positive: 0, positiveRate: 0, avgCostUsd: 0, retryRate: 0 };
    v.total++;
    if (f.rating === "positive") v.positive++;
    v.avgCostUsd += f.usage.costUsd;
    if (f.signals.retried) v.retryRate++;
    byVariant[f.variant] = v;
  }
  for (const v of Object.values(byVariant)) {
    v.positiveRate = v.total > 0 ? v.positive / v.total : 0;
    v.avgCostUsd = v.total > 0 ? v.avgCostUsd / v.total : 0;
    v.retryRate = v.total > 0 ? v.retryRate / v.total : 0;
  }

  return {
    total, rated, positive, negative,
    positiveRate: rated > 0 ? positive / rated : 0,
    totalCostUsd,
    avgCostUsd: total > 0 ? totalCostUsd / total : 0,
    withCorrections,
    retryRate: total > 0 ? retries / total : 0,
    abandonRate: total > 0 ? abandons / total : 0,
    byVariant,
  };
}
```

## Mining Feedback into Eval Cases

Negative feedback with corrections = free eval data. **Critical: corrections must be human-reviewed before
promotion to eval ground truth.** Unauthenticated users could submit poisoned corrections to shift agent
behavior.

### Step 1: Extract Candidates

```typescript
// scripts/feedback-to-evals.ts
import { JsonlFeedbackStore } from "../src/feedback-jsonl.js";
import { writeFileSync, mkdirSync } from "node:fs";

const store = new JsonlFeedbackStore();
const feedback = await store.load({ hasCorrection: true, rating: "negative" });

const candidates = feedback.map((f) => ({
  input: f.input,
  expectedOutput: f.correction,
  originalOutput: f.output,
  source: "feedback",
  feedbackDate: f.timestamp,
  variant: f.variant,
  reviewed: false,
}));

mkdirSync("./evals/fixtures", { recursive: true });
writeFileSync("./evals/fixtures/candidates.json", JSON.stringify(candidates, null, 2));
console.log(`Extracted ${candidates.length} candidates for review`);
```

### Step 2: Human Review

Review `candidates.json` manually. Set `reviewed: true` on approved cases. Delete low-quality or adversarial
corrections. Only reviewed cases become eval fixtures:

```typescript
// scripts/promote-reviewed.ts
import { readFileSync, writeFileSync } from "node:fs";

const candidates = JSON.parse(readFileSync("./evals/fixtures/candidates.json", "utf-8"));
const approved = candidates.filter((c: { reviewed: boolean }) => c.reviewed);

writeFileSync("./evals/fixtures/from-feedback.json", JSON.stringify(approved, null, 2));
console.log(`Promoted ${approved.length} of ${candidates.length} candidates to eval fixtures`);
```

Never auto-promote corrections to eval ground truth without review.

Use in evals:

```typescript
// evals/regression-from-feedback.eval.ts
import { describe, it, expect } from "vitest";
import { runAgent } from "../src/agent.js";
import { llmJudge } from "./scorers.js";
import { recordEval } from "./setup.js";
import feedbackCases from "./fixtures/from-feedback.json";

const agentConfig = { system: "...", maxSteps: 15 };

describe("feedback regressions", () => {
  for (const testCase of feedbackCases) {
    it(`handles: ${testCase.input.slice(0, 60)}...`, async () => {
      const result = await runAgent(testCase.input, agentConfig);
      const judge = await llmJudge(
        `Response should match this expected answer: ${testCase.expectedOutput}`,
      );
      const verdict = await judge(testCase.input, result.text);
      recordEval({
        name: `feedback/${testCase.input.slice(0, 40)}`,
        score: verdict.score,
        tokens: result.usage.totalTokens,
        costUsd: result.usage.costUsd,
        durationMs: result.durationMs,
      });
      expect(verdict.score).toBeGreaterThanOrEqual(0.7);
    });
  }
});
```

## Feedback Report CLI

```typescript
// scripts/feedback-report.ts
import { JsonlFeedbackStore } from "../src/feedback-jsonl.js";
import { summarizeFeedback } from "../src/feedback.js";

const store = new JsonlFeedbackStore();
const feedback = await store.load();
const summary = summarizeFeedback(feedback);

console.log("\n--- Feedback Report ---");
console.table({
  "Total responses": summary.total,
  "Rated": summary.rated,
  "Positive": `${summary.positive} (${(summary.positiveRate * 100).toFixed(1)}%)`,
  "Negative": summary.negative,
  "With corrections": summary.withCorrections,
  "Retry rate": `${(summary.retryRate * 100).toFixed(1)}%`,
  "Abandon rate": `${(summary.abandonRate * 100).toFixed(1)}%`,
  "Total cost": `$${summary.totalCostUsd.toFixed(4)}`,
  "Avg cost/response": `$${summary.avgCostUsd.toFixed(4)}`,
});

if (Object.keys(summary.byVariant).length > 1) {
  console.log("\n--- By Variant ---");
  console.table(
    Object.entries(summary.byVariant).map(([name, v]) => ({
      Variant: name,
      Total: v.total,
      "Positive Rate": `${(v.positiveRate * 100).toFixed(1)}%`,
      "Retry Rate": `${(v.retryRate * 100).toFixed(1)}%`,
      "Avg Cost": `$${v.avgCostUsd.toFixed(4)}`,
    })),
  );
}
```

## A/B Testing — Offline Experiments

Run same inputs against different configs:

```typescript
// src/experiment.ts
import { runAgent, type AgentConfig, type AgentResult } from "./agent.js";
import { writeFileSync, mkdirSync } from "node:fs";

export interface Variant {
  name: string;
  config: AgentConfig;
}

export interface ExperimentSummary {
  name: string;
  timestamp: string;
  inputCount: number;
  variants: Record<string, VariantSummary>;
  winner: string | null;
  totalCostUsd: number;
}

interface VariantSummary {
  avgScore: number;
  avgCostUsd: number;
  avgDurationMs: number;
  avgTokens: number;
  totalCostUsd: number;
}

export async function runExperiment(
  name: string,
  inputs: string[],
  variants: Variant[],
  scorer: (input: string, output: string) => Promise<number>,
): Promise<ExperimentSummary> {
  const results: { input: string; variants: Record<string, AgentResult> }[] = [];
  const scores: Record<string, number[]> = {};
  for (const variant of variants) scores[variant.name] = [];

  for (const input of inputs) {
    const variantResults: Record<string, AgentResult> = {};
    for (const variant of variants) {
      const result = await runAgent(input, { ...variant.config, variant: variant.name });
      variantResults[variant.name] = result;
      scores[variant.name].push(await scorer(input, result.text));
    }
    results.push({ input, variants: variantResults });
  }

  const variantSummaries: Record<string, VariantSummary> = {};
  for (const variant of variants) {
    const data = results.map((r) => r.variants[variant.name]);
    variantSummaries[variant.name] = {
      avgScore: avg(scores[variant.name]),
      avgCostUsd: avg(data.map((d) => d.usage.costUsd)),
      avgDurationMs: avg(data.map((d) => d.durationMs)),
      avgTokens: avg(data.map((d) => d.usage.totalTokens)),
      totalCostUsd: sum(data.map((d) => d.usage.costUsd)),
    };
  }

  const winner = pickWinner(variantSummaries);
  const totalCostUsd = Object.values(variantSummaries).reduce((s, v) => s + v.totalCostUsd, 0);

  const summary: ExperimentSummary = {
    name, timestamp: new Date().toISOString(), inputCount: inputs.length,
    variants: variantSummaries, winner, totalCostUsd,
  };

  mkdirSync("./data/experiments", { recursive: true });
  writeFileSync(`./data/experiments/${name}-${Date.now()}.json`, JSON.stringify({ summary, results }, null, 2));
  return summary;
}

function pickWinner(variants: Record<string, VariantSummary>): string | null {
  const entries = Object.entries(variants);
  if (entries.length < 2) return null;
  entries.sort(([, a], [, b]) => b.avgScore - a.avgScore);
  if (entries[0][1].avgScore - entries[1][1].avgScore < 0.05) return null;
  return entries[0][0];
}

function avg(nums: number[]): number {
  return nums.length > 0 ? nums.reduce((s, n) => s + n, 0) / nums.length : 0;
}

function sum(nums: number[]): number {
  return nums.reduce((s, n) => s + n, 0);
}
```

## A/B Testing — Online (Live Traffic)

Route live requests to variants. Session-sticky assignment:

```typescript
// src/experiment.ts (continued)

export interface OnlineExperiment {
  name: string;
  variants: Variant[];
  weights: number[];
}

export function assignVariant(experiment: OnlineExperiment, sessionId: string): Variant {
  const hash = simpleHash(sessionId + experiment.name);
  const normalizedHash = (hash % 1000) / 1000;

  let cumulative = 0;
  for (let i = 0; i < experiment.variants.length; i++) {
    cumulative += experiment.weights[i];
    if (normalizedHash < cumulative) return experiment.variants[i];
  }
  return experiment.variants[experiment.variants.length - 1];
}

function simpleHash(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash + str.charCodeAt(i)) | 0;
  }
  return Math.abs(hash);
}
```

After ~100 feedback entries per variant, compare via `summarizeFeedback` with `byVariant` breakdown.

## What to A/B Test

| Variable | How | Example |
| -------- | --- | ------- |
| System prompt | Different `system` strings | Concise vs detailed |
| Model | Different `MODEL_ID` | claude-sonnet vs claude-haiku |
| Tool set | Different tool registries | With/without calculator |
| Max steps | Different step counts | 10 vs 25 |
| Temperature | Different values | 0.0 vs 0.3 |

## Cost-Quality Tradeoff

```typescript
function costAdjustedScore(score: number, costUsd: number, costWeight = 0.3): number {
  const normalizedCost = Math.min(costUsd / 0.10, 1);
  return score * (1 - costWeight) + (1 - normalizedCost) * costWeight;
}
```

## Graduation Criteria

Promote winner when:

1. **Sufficient sample** — 100+ responses per variant
2. **Clear winner** — >5% score difference sustained
3. **Cost acceptable** — within budget constraints
4. **No regressions** — passes all existing eval cases
5. **Feedback confirms** — online positive rate >= control
