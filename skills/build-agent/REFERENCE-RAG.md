# Build Agent — RAG Reference

Retrieval-augmented generation pipeline: embedding, vector storage, retrieval, chunking, and integration
with the agent loop.

## When to Use RAG

Add RAG when the agent needs knowledge beyond the model's training data or context window:

- Large documentation sets (API docs, knowledge bases, runbooks)
- Codebase search across repositories too large to fit in context
- Domain-specific data that changes frequently (tickets, incidents, configs)
- Multi-tenant agents where each tenant has private data

Do not use RAG when the data fits in the context window — direct inclusion is simpler and more reliable.

## Architecture

```text
Query → Embed → Search vector store → Retrieve top-K chunks → Inject into prompt → LLM
```

```typescript
// src/rag.ts
import { z } from "zod";

export interface Chunk {
  id: string;
  text: string;
  metadata: Record<string, string>;
  embedding?: number[];
}

export interface RetrievalResult {
  chunk: Chunk;
  score: number;
}

export interface Embedder {
  embed(text: string): Promise<number[]>;
  embedBatch(texts: string[]): Promise<number[][]>;
  dimensions: number;
}

export interface VectorStore {
  upsert(chunks: Chunk[]): Promise<void>;
  search(embedding: number[], topK: number, filter?: Record<string, string>): Promise<RetrievalResult[]>;
  delete(ids: string[]): Promise<void>;
}

export interface RAGClient {
  ingest(documents: Document[]): Promise<number>;
  retrieve(query: string, topK?: number): Promise<RetrievalResult[]>;
}
```

## Embedding

Use Vertex AI's text embedding API. Choose task type based on your use case:

```typescript
// src/rag/embedder.ts
import { VertexAI } from "@google-cloud/vertexai";

export type EmbeddingTaskType =
  | "SEMANTIC_SIMILARITY"
  | "RETRIEVAL_DOCUMENT"
  | "RETRIEVAL_QUERY"
  | "QUESTION_ANSWERING"
  | "CLASSIFICATION";

export class VertexEmbedder implements Embedder {
  private client: VertexAI;
  readonly dimensions: number;

  constructor(
    private config: {
      project: string;
      location: string;
      model?: string;
      dimensions?: number;
      taskType?: EmbeddingTaskType;
    },
  ) {
    this.client = new VertexAI({ project: config.project, location: config.location });
    this.dimensions = config.dimensions ?? 768;
  }

  async embed(text: string): Promise<number[]> {
    const [result] = await this.embedBatch([text]);
    return result;
  }

  async embedBatch(texts: string[]): Promise<number[][]> {
    const model = this.client.getGenerativeModel({
      model: this.config.model ?? "text-embedding-005",
    });

    const response = await model.batchEmbedContents({
      requests: texts.map((text) => ({
        content: { role: "user", parts: [{ text }] },
        taskType: this.config.taskType ?? "RETRIEVAL_DOCUMENT",
        outputDimensionality: this.dimensions,
      })),
    });

    return response.embeddings.map((e) => e.values);
  }
}
```

| Task type | Use for |
| --------- | ------- |
| `RETRIEVAL_DOCUMENT` | Indexing documents for later search |
| `RETRIEVAL_QUERY` | Embedding search queries |
| `SEMANTIC_SIMILARITY` | Comparing two texts for similarity |
| `QUESTION_ANSWERING` | Q&A over documents |
| `CLASSIFICATION` | Categorizing text |

Use `RETRIEVAL_DOCUMENT` when indexing and `RETRIEVAL_QUERY` when searching — they are asymmetric
embeddings optimized for the retrieval use case.

## Chunking

Split documents into chunks that fit the embedding model's context and preserve semantic coherence:

```typescript
// src/rag/chunker.ts
export interface ChunkConfig {
  maxChunkSize: number;
  overlap: number;
  separator?: string;
}

export interface Document {
  id: string;
  text: string;
  metadata: Record<string, string>;
}

export function chunkDocument(doc: Document, config: ChunkConfig): Chunk[] {
  const { maxChunkSize, overlap, separator = "\n\n" } = config;
  const sections = doc.text.split(separator);
  const chunks: Chunk[] = [];
  let current = "";
  let chunkIndex = 0;

  for (const section of sections) {
    if (current.length + section.length > maxChunkSize && current.length > 0) {
      chunks.push({
        id: `${doc.id}#${chunkIndex}`,
        text: current.trim(),
        metadata: { ...doc.metadata, source: doc.id, chunkIndex: String(chunkIndex) },
      });
      chunkIndex++;
      const words = current.split(/\s+/);
      current = words.slice(-Math.floor(overlap / 4)).join(" ") + separator;
    }
    current += section + separator;
  }

  if (current.trim().length > 0) {
    chunks.push({
      id: `${doc.id}#${chunkIndex}`,
      text: current.trim(),
      metadata: { ...doc.metadata, source: doc.id, chunkIndex: String(chunkIndex) },
    });
  }

  return chunks;
}
```

### Chunking strategies

| Strategy | When | Chunk size |
| -------- | ---- | ---------- |
| Fixed-size with overlap | General text, logs | 500-1000 tokens, 10-20% overlap |
| Section-based | Markdown, docs with headers | Split on `##` headers |
| Semantic | Mixed content | Split on paragraph boundaries |
| Code-aware | Source files | Split on function/class boundaries |

For code, split on AST boundaries (function declarations, class definitions) rather than fixed character
counts. A function split in the middle is useless for retrieval.

### Source text preservation

Store the original source text in chunk metadata. When upgrading embedding models, re-embed from stored
source text rather than re-fetching documents:

```typescript
export function chunkWithSource(doc: Document, config: ChunkConfig): Chunk[] {
  const chunks = chunkDocument(doc, config);
  return chunks.map((chunk) => ({
    ...chunk,
    metadata: { ...chunk.metadata, sourceText: chunk.text },
  }));
}
```

## Vector Store

### In-Memory (Development)

```typescript
// src/rag/store-memory.ts
export class InMemoryVectorStore implements VectorStore {
  private chunks = new Map<string, Chunk>();

  async upsert(chunks: Chunk[]): Promise<void> {
    for (const chunk of chunks) {
      this.chunks.set(chunk.id, chunk);
    }
  }

  async search(
    embedding: number[],
    topK: number,
    filter?: Record<string, string>,
  ): Promise<RetrievalResult[]> {
    let candidates = [...this.chunks.values()];

    if (filter) {
      candidates = candidates.filter((chunk) =>
        Object.entries(filter).every(([key, value]) => chunk.metadata[key] === value),
      );
    }

    return candidates
      .map((chunk) => ({
        chunk,
        score: cosineSimilarity(embedding, chunk.embedding ?? []),
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK);
  }

  async delete(ids: string[]): Promise<void> {
    for (const id of ids) this.chunks.delete(id);
  }
}

function cosineSimilarity(a: number[], b: number[]): number {
  if (a.length !== b.length || a.length === 0) return 0;
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  const denom = Math.sqrt(normA) * Math.sqrt(normB);
  return denom === 0 ? 0 : dot / denom;
}
```

### Production Store

For production, use a managed vector database. Implement the `VectorStore` interface for your backend:

| Backend | Best for | Notes |
| ------- | -------- | ----- |
| Vertex AI Vector Search | GCP-native, large scale | Managed, supports streaming upsert |
| pgvector (PostgreSQL) | Already using Postgres | Extension, good for < 1M vectors |
| Qdrant | Self-hosted, high performance | Good filtering, supports namespaces |
| Weaviate | Multi-modal, hybrid search | Supports BM25 + vector search |

## RAG Client

Compose embedder, chunker, and store into a single client:

```typescript
// src/rag/client.ts
export class DefaultRAGClient implements RAGClient {
  constructor(
    private embedder: Embedder,
    private store: VectorStore,
    private chunkConfig: ChunkConfig = { maxChunkSize: 800, overlap: 100 },
  ) {}

  async ingest(documents: Document[]): Promise<number> {
    let totalChunks = 0;

    for (const doc of documents) {
      const chunks = chunkDocument(doc, this.chunkConfig);
      const texts = chunks.map((c) => c.text);
      const embeddings = await this.embedder.embedBatch(texts);

      const embeddedChunks = chunks.map((chunk, i) => ({
        ...chunk,
        embedding: embeddings[i],
      }));

      await this.store.upsert(embeddedChunks);
      totalChunks += embeddedChunks.length;
    }

    return totalChunks;
  }

  async retrieve(query: string, topK = 5): Promise<RetrievalResult[]> {
    const queryEmbedding = await this.embedder.embed(query);
    return this.store.search(queryEmbedding, topK);
  }
}
```

## Integration with Agent Loop

Wire RAG into the agent as a tool, not as prompt injection. This lets the model decide when to search and
what to search for:

```typescript
// src/tools/search-knowledge.ts
import { tool } from "ai";
import { z } from "zod";
import type { RAGClient } from "../rag/client.js";

export function createSearchTool(rag: RAGClient) {
  return tool({
    description: "Search the knowledge base for relevant information. Use when the user asks about " +
      "topics that require domain-specific knowledge.",
    inputSchema: z.object({
      query: z.string().min(3).max(500).describe("Search query — be specific"),
      topK: z.number().int().min(1).max(20).default(5).describe("Number of results"),
    }),
    execute: async ({ query, topK }) => {
      const results = await rag.retrieve(query, topK);
      if (results.length === 0) {
        return { ok: true, data: { results: [], message: "No relevant documents found" } };
      }
      return {
        ok: true,
        data: {
          results: results.map((r) => ({
            text: r.chunk.text,
            source: r.chunk.metadata.source ?? "unknown",
            score: r.score,
          })),
        },
      };
    },
  });
}
```

### Prompt guidance for RAG tools

Add retrieval instructions to the system prompt:

```text
When answering questions, search the knowledge base first. Base your answer on retrieved documents.
If no relevant documents are found, say so — do not guess or use general knowledge.
Cite sources by referencing the document source field.
```

## Distance Thresholds

Not all retrieved chunks are relevant. Filter by cosine distance to avoid injecting noise:

```typescript
export async function retrieveWithThreshold(
  rag: RAGClient,
  query: string,
  topK: number,
  minScore = 0.7,
): Promise<RetrievalResult[]> {
  const results = await rag.retrieve(query, topK);
  return results.filter((r) => r.score >= minScore);
}
```

Tune `minScore` per use case. Start at 0.7 and adjust based on eval results. Too high = misses relevant
content. Too low = injects noise that confuses the model.

## Namespace Filtering

For multi-tenant agents or agents with multiple knowledge domains, use metadata filters to scope retrieval:

```typescript
const results = await store.search(queryEmbedding, topK, {
  tenant: "acme-corp",
  docType: "runbook",
});
```

Namespace filtering is AND across keys (all filters must match) and OR within values if the store supports
array values. This prevents cross-tenant data leakage in multi-tenant deployments.

## RAG Evals

Evaluate retrieval quality separately from generation quality:

```typescript
// evals/rag.eval.ts
import { describe, it, expect } from "vitest";
import { DefaultRAGClient } from "../src/rag/client.js";
import { recordEval } from "./setup.js";

describe("RAG retrieval", () => {
  it("retrieves relevant chunks for known queries", async () => {
    const results = await rag.retrieve("How do I configure rate limiting?", 5);
    const sources = results.map((r) => r.chunk.metadata.source);

    expect(results.length).toBeGreaterThan(0);
    expect(sources).toContain("docs/rate-limiting.md");
    expect(results[0].score).toBeGreaterThanOrEqual(0.7);
  });

  it("returns empty for out-of-domain queries", async () => {
    const results = await retrieveWithThreshold(rag, "recipe for chocolate cake", 5, 0.7);
    expect(results.length).toBe(0);
  });
});
```

Key RAG metrics to track:

| Metric | What it measures | Target |
| ------ | --------------- | ------ |
| Recall@K | Fraction of relevant docs in top-K | > 0.8 |
| Precision@K | Fraction of top-K that are relevant | > 0.6 |
| MRR | Mean reciprocal rank of first relevant result | > 0.7 |
| Groundedness | Agent claims supported by retrieved docs | > 0.9 |

Use the groundedness and faithfulness scorers from REFERENCE-EVALS.md to verify the agent stays faithful to
retrieved content.

## Cost Considerations

| Component | Cost driver | Mitigation |
| --------- | ----------- | ---------- |
| Embedding queries | Per-token, each search | Cache frequent query embeddings |
| Embedding documents | Per-token, at ingest time | Batch upserts, re-embed only on model upgrade |
| Vector store | Storage + queries | Use metadata filters to reduce search space |
| Retrieved context | Adds to LLM input tokens | Limit topK, use distance thresholds |

Approximate token estimation for embedding cost tracking:

```typescript
export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}
```
