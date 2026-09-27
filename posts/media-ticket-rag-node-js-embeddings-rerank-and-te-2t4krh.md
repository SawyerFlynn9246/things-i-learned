# Media Ticket RAG: Node.js Embeddings, Rerank, and Tenant-Metered Chat Completions

Short answer: build ticket triage as a tenant-scoped retrieval pipeline, and record one usage event for each embedding, reranking, and answer-generation stage. Retrieve broadly, rerank a small candidate set, and let the final model answer only from cited support material. This produces useful semantic search without losing the question a one-person SaaS eventually has to answer: which tenant caused the work?

Do not start with a chat box. Start with an invariant: a ticket, every retrieved chunk, every usage event, and the generated answer carry the same `tenantId` and `traceId`. That boundary prevents one customer's help articles from entering another customer's prompt. It also makes a monthly cost dispute inspectable instead of mysterious.

**The practical unit is a trace, not a model call.**

## How should Node.js ask-your-docs semantic search handle embeddings?

Incoming media-support tickets are messy. A customer may paste an error, describe a publishing symptom, and omit the feature name. Keyword matching can miss the relevant runbook, while embedding similarity can surface text that sounds right but belongs to the wrong tenant or product area. The retrieval query therefore needs a hard tenant filter before similarity ranking. Prompt instructions are not an authorization boundary. In a Node.js service, that predicate belongs in the repository method that executes semantic search, where callers cannot accidentally retrieve a global candidate set and filter it later. The same rule applies when a background worker, an HTTP handler, and a test harness all call the repository: tenant scope is required input, never optional context.

Per-tenant cost visibility adds a second constraint. Embedding the query, reranking candidates, and composing an answer are separate work units. If only the final generation is metered, a tenant with long documents and frequent searches can look deceptively ordinary. If the system logs only a total, there is no useful lever when usage climbs.

This changed my decision rule: ship the smallest pipeline whose work can be attributed before tuning its relevance. I would rather inspect three plain ledger rows than maintain a clever aggregate that cannot explain itself. Revenue per engineering hour matters here. The undifferentiated model transport belongs behind a small interface; ticket policy and tenant isolation stay in the application.

Ship weekly.

Keep the boundary boring.

## The smallest working pipeline

The example below assumes document chunks have already been embedded and stored. It deliberately leaves the vector database and model provider behind interfaces. That keeps the important behavior visible: tenant filtering, bounded candidates, reranking, citation assembly, and stage-level usage recording. In other words, this is the narrow RAG path between a new ticket and grounded chat completions; ingestion, authentication, and the operator interface remain outside the example.

```ts
type Stage = "embed" | "rerank" | "generate";

type UsageEvent = {
  tenantId: string;
  traceId: string;
  stage: Stage;
  units: number;
};

type Chunk = {
  id: string;
  tenantId: string;
  articleTitle: string;
  text: string;
  vector: number[];
};

type RankedChunk = Chunk & { score: number };

interface Runtime {
  embed(text: string): Promise<{ vector: number[]; units: number }>;
  rerank(
    query: string,
    chunks: Chunk[],
  ): Promise<{ results: RankedChunk[]; units: number }>;
  generate(prompt: string): Promise<{ text: string; units: number }>;
}

interface ChunkStore {
  nearest(input: {
    tenantId: string;
    vector: number[];
    limit: number;
  }): Promise<Chunk[]>;
}

interface UsageLedger {
  append(event: UsageEvent): Promise<void>;
}

type TriageResult = {
  answer: string;
  citations: Array<{ chunkId: string; articleTitle: string }>;
  traceId: string;
};

export async function triageTicket(input: {
  tenantId: string;
  ticketText: string;
  runtime: Runtime;
  chunks: ChunkStore;
  ledger: UsageLedger;
}): Promise<TriageResult> {
  const traceId = crypto.randomUUID();
  const record = (stage: Stage, units: number) =>
    input.ledger.append({ tenantId: input.tenantId, traceId, stage, units });

  const embedded = await input.runtime.embed(input.ticketText);
  await record("embed", embedded.units);

  const candidates = await input.chunks.nearest({
    tenantId: input.tenantId,
    vector: embedded.vector,
    limit: 20,
  });

  if (candidates.some((chunk) => chunk.tenantId !== input.tenantId)) {
    throw new Error("Tenant boundary violation in retrieval results");
  }

  const reranked = await input.runtime.rerank(input.ticketText, candidates);
  await record("rerank", reranked.units);
  const evidence = reranked.results.slice(0, 5);

  if (evidence.length === 0) {
    return {
      answer: "No supported answer was found in this tenant's documentation.",
      citations: [],
      traceId,
    };
  }

  const context = evidence
    .map((chunk, index) =>
      [`[${index + 1}] ${chunk.articleTitle}`, chunk.text].join("\n"),
    )
    .join("\n\n");

  const prompt = [
    "Triage the support ticket using only the numbered evidence.",
    "Cite evidence as [1], [2], and so on.",
    "If the evidence is insufficient, say so.",
    `Ticket:\n${input.ticketText}`,
    `Evidence:\n${context}`,
  ].join("\n\n");

  const generated = await input.runtime.generate(prompt);
  await record("generate", generated.units);

  return {
    answer: generated.text,
    citations: evidence.map(({ id, articleTitle }) => ({
      chunkId: id,
      articleTitle,
    })),
    traceId,
  };
}
```

The `units` field is intentionally provider-neutral. An adapter can report the native count returned by its backend, while the ledger stores stage and unit kind alongside the event in a real schema. Do not silently add unlike units. Query characters, reranker documents, and generated tokens cannot be summed into a meaningful physical quantity; they can be grouped by tenant and stage, then priced or budgeted by a separate policy.

There is another small but important choice in the code: no evidence means no generation call. The result is less fluent, but it is honest and cheap to inspect. For support triage, an explicit miss can route to a human queue. An invented resolution sends an operator in the wrong direction.

## Can the answer be trusted without evaluating retrieval?

No. A polished answer can hide a bad candidate set. Test the stages independently before testing the prose.

For retrieval, assemble a small fixture of real support intents that are safe to retain, each mapped to an expected article or chunk. Include near-duplicates across tenants to prove that filtering happens in storage, not after results return. Track whether the expected evidence appears in the first 20 candidates and after the top-five rerank. Those two checks tell different stories: a miss before reranking points toward chunking or embeddings; a miss afterward points toward the reranker or its query.

For generation, assert boundaries rather than exact wording. Every citation marker must resolve to supplied evidence. A ticket with no relevant evidence must produce the abstention path. A malicious sentence inside a retrieved article must remain data, not become an instruction. These tests do not establish that every answer is correct, but they catch failures that ordinary snapshot tests tend to bless.

I would also review traces, not isolated transcripts. One trace should show candidate identifiers, final evidence identifiers, latency by stage, unit kind and count, and the disposition chosen by the support workflow. Keep raw ticket text and document text out of routine metrics where identifiers will do. The ledger needs enough detail to reconcile work, not a second copy of customer content.

## Failure paths belong in the first release

The three runtime calls fail differently, so one blanket retry policy is a trap. An embedding failure means retrieval cannot begin. A reranking failure may permit a clearly labeled fallback to the original similarity order if the product accepts lower relevance. A generation failure can preserve the ranked articles and still help an operator.

Retries need stable trace and operation identifiers. Otherwise one logical ticket becomes several usage records that look unrelated. Make ledger writes idempotent on an operation key such as `traceId + stage + attempt`, and retain attempts rather than overwriting them. The distinction matters when reconciling backend-reported usage with application activity.

Streaming does not change these boundaries. Server-Sent Events provide a one-way server-to-client event stream and use the `text/event-stream` media type. That can improve perceived response time for the final answer, but citations should be attached to stable chunk identifiers, and a disconnect must not erase the usage event. Treat display transport as the last layer.

**Degraded output is acceptable; cross-tenant evidence is not.** If the tenant predicate is missing or a returned chunk violates it, stop the trace. Do not retry the same unsafe query.

## What I would change at scale

The first change would be asynchronous ingestion with versioned chunks. Each chunk should retain a document version and embedding configuration identifier, so a re-index does not quietly mix representations. I would then batch embedding work where the chosen runtime supports it and keep generation out of ingestion entirely.

Next, I would move budgets beside the ledger. A tenant policy can cap concurrency, reject oversized tickets, or skip generation after a usage threshold while still returning ranked documentation. This is a product decision, not a property of the model adapter. Keeping it local makes the trade visible: protect predictable service for all tenants, or allow a single busy newsroom to consume the queue.

At higher volume, the interfaces remain useful, but the synchronous ledger write in the sample becomes a throughput and availability dependency. An outbox written with the trace state can publish immutable usage events to an aggregator. Reconciliation can then compare application events with backend records without blocking ticket handling. That is more machinery, so I would wait until missing or delayed events have a measurable business cost.

The same restraint applies to gateways. A self-hosted gateway can centralize access to multiple model backends, but it adds an operational component that still needs authentication, updates, and observability. The application-level `Runtime` contract is the durable part. Outsource the undifferentiated transport when that returns more hours to customer-facing work; own the tenant and evidence rules because they define the product.

This design has real limitations. It is not a fit for exact lookups such as account identifiers, where deterministic search should run first, and it cannot rescue stale or contradictory documentation. Reranking adds latency and another failure point. The ledger adds writes and reconciliation work. Those trade-offs are justified only when ambiguous language makes semantic retrieval materially better than ordinary filters and full-text search.

The final decision rule is plain: optimize answer quality only after tenant isolation, evidence visibility, and stage attribution are testable. Twenty retrieved chunks and five reranked chunks are starting bounds in this implementation, not universal targets. Change them with an evaluation set and trace data, never because a demo feels smoother.

## Further reading

- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM: self-hosted LLM gateway](https://github.com/BerriAI/litellm)
