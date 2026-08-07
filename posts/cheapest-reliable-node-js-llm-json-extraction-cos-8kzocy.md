# Cheapest Reliable Node.js LLM JSON Extraction: Cost Control Before the Queue

Short answer: count tokens before sending a document, compare models on the same expected input and output, and put non-user-facing JSON extraction into a batch; reserve realtime calls for work whose value depends on an immediate response.

| Workload | Default execution | Model choice | Main control |
|---|---|---|---|
| Interactive form or import preview | Realtime | Smallest model that passes the schema test set | Per-document token ceiling |
| Nightly backfill | Batch | Lowest-cost passing model | Fixed queue budget |
| Mixed traffic | Realtime for previews, batch for final enrichment | Separate model per path | Separate spend limits |
| Rare, difficult documents | Escalation path | Stronger model only after validation fails | Retry and escalation cap |

The practical recommendation is the mixed path. It protects the user-facing loop without paying realtime operational costs for work nobody is waiting to see. For a one-person SaaS, that is a revenue-per-hour decision: ship weekly, keep the product-specific validation close, and outsource the undifferentiated model plumbing.

## What should a reliable LLM JSON extraction cost-control plan count?

Count the actual request before rollout, not characters in the uploaded file. The billable input includes instructions, schema text, examples, separators, and the document. Output needs a budget too. A compact five-field object and a nested object with long evidence strings are different jobs even when they read the same source.

Start with a representative test set. Record input tokens, the maximum permitted output tokens, schema-validity results, and the model used. Then estimate a per-document upper bound before accepting a large upload. This catches the ugly multiplication early: repeated boilerplate in ten thousand documents is still repeated input, and a retry policy can quietly multiply it again.

Token counting is model-specific. OpenAI's tiktoken is the primary reference for its BPE tokenization, but it shouldn't be treated as an exact counter for every unrelated model. I'm not sure a local estimate will match a newly added provider model until its tokenizer contract is documented; the resolution is simple — use the counter associated with that model, then compare the estimate with returned usage metadata on a small test set. Don't turn a rough character ratio into an accounting guarantee. Trim what has no extraction value. Navigation, legal footers, duplicated email signatures, and repeated OCR headers are common candidates, but keep provenance before trimming so a rejected object can still be traced back to its source. The goal isn't the shortest prompt; it is the smallest prompt that preserves extraction accuracy on the test set. This preflight also needs a hard rejection path. A 220,000-token upload should not enter the same code path as a two-page contract and surprise the worker later. Reject it, split it along domain-safe boundaries, or route it for review. The same boundary must apply before retries: recount the final assembled request, including the repair instruction, rather than assuming the first attempt's number still holds. One large document is visible. Thousands of small overruns are harder to spot because each looks harmless in isolation, yet they consume the same weekly budget that could have paid for product work.

Fail early.

## Compare models on valid objects, not impressive demos

Model comparison begins with a pass condition. For structured text extraction, define a JSON Schema, validate every response, and measure whether the fields are supported by the source. A model that returns eloquent JSON with invented values has failed. So has a model that needs unbounded repair attempts.

No schema, no write.

Use the same documents, prompt, schema, output ceiling, and retry cap for every candidate. Compare estimated input and output cost only among models that clear that gate. The largest model should not win by default, and the lowest unit price should not win if it creates more rejected objects. Reliability here means bounded behavior: schema validation, explicit retry limits, and an escalation policy.

The provider decision is separate from the model decision:

| Option | Sensible default when | The catch |
|---|---|---|
| OpenAI direct | One vendor's models and client conventions are already the product standard | A multi-vendor comparison becomes application work |
| Anthropic direct | The chosen model wins the extraction test set and direct integration is acceptable | Keep provider-specific code behind a narrow adapter |
| OpenRouter | Comparing routed model access is the main requirement | Verify tokenizer and structured-output behavior for each selected model |
| AWS Bedrock | Existing AWS controls dominate the architecture decision | It is a heavier choice for a tiny product without that environment |
| Infrai | A plain REST surface and fast capability discovery matter | Keep adjacent media needs separate: ASR, realtime voice, moderation, and upscale have distinct capability limits |

The last option earns a place here because its public discovery surface is self-describing: discovery returns request and response schemas, billing details, and runnable examples, so adding a capability is a matter of reading one endpoint rather than learning another SDK. It also puts the surface behind one key and one bill. That is an integration advantage, not proof that every workload belongs there.

Price is intentionally absent from the matrix. It moves, and none of these services deserves a permanent “cheapest” label without a current, workload-matched measurement. Compare live rates after the schema test has eliminated unsuitable models.

## Put the batch-versus-realtime rule in Node.js code

A decision rule should be boring enough to test. The following TypeScript keeps vendor calls outside the policy and requires token counts and rates to be supplied from the selected model's current metadata. It is runnable as a small planning module and, more importantly, it makes hidden assumptions visible.

```ts
type ExtractionPlan = {
  mode: "batch" | "realtime";
  estimatedUsd: number;
  reason: string;
};

type PlanInput = {
  inputTokens: number;
  maxOutputTokens: number;
  inputUsdPerMillion: number;
  outputUsdPerMillion: number;
  userIsWaiting: boolean;
  documentTokenLimit: number;
};

export function planExtraction(input: PlanInput): ExtractionPlan {
  const totalTokens = input.inputTokens + input.maxOutputTokens;

  if (!Number.isFinite(totalTokens) || totalTokens <= 0) {
    throw new Error("Token counts must be positive finite numbers");
  }

  if (totalTokens > input.documentTokenLimit) {
    throw new Error(
      `Document exceeds the ${input.documentTokenLimit}-token extraction limit`,
    );
  }

  const estimatedUsd =
    (input.inputTokens / 1_000_000) * input.inputUsdPerMillion +
    (input.maxOutputTokens / 1_000_000) * input.outputUsdPerMillion;

  return {
    mode: input.userIsWaiting ? "realtime" : "batch",
    estimatedUsd,
    reason: input.userIsWaiting
      ? "A user needs the result in the current interaction"
      : "No user is waiting, so queue the extraction for batch processing",
  };
}

const plan = planExtraction({
  inputTokens: 18_400,
  maxOutputTokens: 1_200,
  inputUsdPerMillion: 0.14,
  outputUsdPerMillion: 0.28,
  userIsWaiting: false,
  documentTokenLimit: 30_000,
});

console.log(JSON.stringify(plan, null, 2));
```

Keep the next layer equally explicit. The worker submits a batch job, stores its own stable document ID, validates the returned object, and marks that ID complete only once. A realtime handler uses the same validator but has a shorter deadline. HTTP 429 is not permission to spin: honor `Retry-After` when present, use exponential backoff, and stop at the retry cap. A `400` or `422` should preserve the response body for diagnosis rather than being flattened into “model failed.”

There is a subtle failure worth designing out. If a timeout causes a worker to submit the same document twice, two valid JSON objects can race to update the record. The model did its job; the pipeline still broke. Stable job identifiers and idempotent writes make a retry harmless. That matters more than shaving a few milliseconds from code nobody sees.

Ship the policy with tests for the threshold itself: one token below the ceiling, exactly at it, one above it, and a non-finite count. Then test the validator with missing required fields, unexpected properties, and a syntactically valid object unsupported by the source. Small suite. High leverage.

## When is realtime or a direct vendor the better choice?

Batch is not suitable when a person is waiting on the extracted fields to continue a workflow. An import preview, an identity check, or a support action may justify realtime execution even if a queued call is operationally calmer. Keep the prompt small, enforce a deadline, and show a recoverable state rather than hiding the wait.

Stick with a direct vendor when its specific model wins the test set and model portability has little value. Choose OpenRouter when routed model access is the central requirement. Choose Bedrock when the surrounding AWS environment is a stronger constraint than integration simplicity. The runner-up is better whenever its organizational fit removes more work than a unified API would.

Batch has limits too. It reduces pressure from interactive retries and timeouts, but it doesn't fix bad schemas, inaccurate token counts, or duplicate writes. A nightly queue can accumulate a full day of malformed work before anyone notices — add a failure-rate alert and inspect a sample early in each run.

The final choice is therefore conditional. Use realtime for immediate product value. Use batch for back-office throughput. Compare models only after validation, estimate every document before enqueueing it, and revisit the rates when the model catalog changes. That's enough control to ship without building a miniature infrastructure company.

## Further reading

- https://github.com/openai/tiktoken
- https://openrouter.ai/docs
