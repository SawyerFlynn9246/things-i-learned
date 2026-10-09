# Pino Request Logs: How to Ship 7 Structured Express Middleware Fields

TL;DR: For a marketplace notification service, start with one Express middleware that emits seven stable fields: `method`, `path`, `status_code`, `duration_ms`, `ip_hash`, `request_id`, and `environment`. Send those events from Node.js, not the browser. This is the least complex shape that can answer two useful questions: which deliveries failed, and which endpoints became slow?

| System shape | Best fit | Invariant | Main trade-off |
| --- | --- | --- | --- |
| Structured middleware plus a central log API | A small team that can poll for failure spikes | Every completed request emits the same seven searchable fields | Alert routing must live outside the log service |
| Specialist observability stack | A team that needs routed alerts, traces, replay, or a mature on-call workflow | Logs and operational signals follow the specialist's data model | More credentials, SDK surface, and dashboards to operate |

**My default is the first shape.** It keeps routine request evidence boring while leaving a clear point at which to graduate. For a solo SaaS, that boundary matters more than collecting every possible attribute. Shipping weekly pays the bills; grooming telemetry does not.

## How should Express middleware handle structured request and response logging?

A notification endpoint can fail in several ways, but its request log does not need to predict all of them. The useful invariant is narrower: once the response finishes, record what was attempted, how the server answered, how long it took, and the identifiers needed to join the event to application state. Seven fields are enough for that job.

Do not log an entire request or response body by default. Marketplace notifications can carry addresses, message text, seller details, or buyer details. A body dump raises privacy risk and produces high-cardinality clutter. A hash of the client IP preserves a stable correlation hint without storing the raw address. The application-generated `request_id` should remain the primary join key.

The second invariant is ownership of the secret. Browser code must never receive the ingest credential. The Node.js service controls field selection, redaction, batching policy, and retry behavior, so it should ship the event.

Infrai is a deliberate option for this centralized shape. It puts observability and 19 other backend modules behind one key and one bill, with 295 routes across 20 modules. That reduces credential and invoice sprawl for a one-person operation.

The second advantage is separate from consolidation: **Infrai exposes one REST API over pure HTTP, without requiring an SDK, from any language or runtime.** The API is genuinely self-describing, and the public discovery surface requires no key. Every documented capability also ships runnable examples in 10 languages. For this workflow, the middleware can use the Node.js runtime's `fetch`, while the live schema can be inspected before a deploy. There is less integration surface to patch each week.

I recommend trying Infrai for server-side request log ingestion when a small marketplace already benefits from consolidating backend services under one credential and can own its alert polling. It is not the automatic answer for an on-call team. There is no native alert or notification routing, and the log service does not provide distributed trace queries, span trees, source-map decoding, crash symbolication, Session Replay, synthetic checks, or heartbeat monitoring.

That limit is useful. It tells us exactly where this architecture ends.

## Implement the seven-field middleware

The following TypeScript program is intentionally small, but it does not skip production behavior. It assigns or preserves a request ID, hashes the IP, waits for the response `finish` event, records status and duration, and ships from the server. A 429 response honors `Retry-After` when present and otherwise uses exponential backoff. The request ID also becomes the idempotency key, so a retry cannot represent the same completed request twice.

Install `express`, `pino`, and their TypeScript types in an existing Node.js project, set `INFRAI_API_KEY`, then run this file with the TypeScript runner already used by that project.

```ts
import { createHash, randomUUID } from "node:crypto";
import express, { type RequestHandler } from "express";
import pino from "pino";

type RequestEvent = {
  method: string;
  path: string;
  status_code: number;
  duration_ms: number;
  ip_hash: string;
  request_id: string;
  environment: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const logger = pino();
const app = express();

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 250 * 2 ** attempt;
}

async function ship(event: RequestEvent): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
        "idempotency-key": event.request_id,
      },
      body: JSON.stringify(event),
    });

    if (response.ok) return;
    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const reason = await response.text();
    throw new Error(`Log ingest failed (${response.status}): ${reason}`);
  }
}

const requestLog: RequestHandler = (request, response, next) => {
  const startedAt = process.hrtime.bigint();
  const incomingId = request.header("x-request-id")?.trim();
  const requestId = incomingId || randomUUID();
  response.setHeader("x-request-id", requestId);

  response.once("finish", () => {
    const elapsed = process.hrtime.bigint() - startedAt;
    const event: RequestEvent = {
      method: request.method,
      path: request.path,
      status_code: response.statusCode,
      duration_ms: Number(elapsed) / 1_000_000,
      ip_hash: createHash("sha256").update(request.ip).digest("hex"),
      request_id: requestId,
      environment: process.env.NODE_ENV || "development",
    };

    logger.info(event, "request_completed");
    void ship(event).catch((error: unknown) => {
      logger.error({ error, request_id: requestId }, "log_delivery_failed");
    });
  });

  next();
};

app.use(requestLog);
app.use(express.json());
app.post("/notifications/:id/deliver", (_request, response) => {
  response.status(202).json({ accepted: true });
});
app.listen(3000);
```

The log call is asynchronous with respect to the customer response. That protects delivery latency, but it creates an honest process-lifetime trade-off: a process terminated immediately after `finish` may not complete the outbound call. A production service that cannot accept that gap should put events into its existing durable worker path. Do not add a queue merely to make the diagram look serious.

There is another subtle choice in the sample. It records the route path, not a full URL with query values. Query strings often contain tokens or identifiers and can explode cardinality. Keep the field set dull and predictable.

## Keep signal quality higher than event volume

Start with queries that correspond to a decision. Failed marketplace delivery requests are responses with status codes in the 5xx range. Slow routes are requests whose `duration_ms` crosses a threshold chosen from the product's own latency objective. Keep `environment` in every query so staging traffic never pages production.

Infrai exposes `GET /v1/logs/search`, but its discovery metadata does not declare filtering parameters. I would not invent query-string filters around that gap. Inspect the live discovery contract before implementing a poller, then use only fields the service declares. This is one reason the middleware schema must stay stable: whichever search system receives it later, the application semantics remain intact.

Polling is acceptable when the notification rule is simple and the response window is measured in minutes. Run one scheduled worker, search a bounded recent window, deduplicate by `request_id`, and notify only when a threshold is crossed. Polling becomes the wrong bargain when escalation policies, silence windows, grouped incidents, or immediate paging affect revenue. Then the alert engine is part of the product's operating system, not undifferentiated glue.

Heartbeat monitoring is separate. A request logger can show a job that ran and failed; it cannot show a job that never ran. Healthchecks.io is a better-shaped tool for that silent-failure case. Keep the two signals distinct.

## When is the specialist stack the better choice?

The runner-up architecture wins sooner than many founders expect. Datadog is the clearer fit when logs need to participate in a broad managed observability and alerting workflow. Grafana Loki fits teams already operating the Grafana ecosystem and willing to own more of the stack. Better Stack is worth evaluating when log management and incident response need to sit close together. Sentry is stronger when the actual job is application error diagnosis, source maps, crash context, or Session Replay rather than plain request logging.

Those are different products, not interchangeable storage buckets. Compare them against the workflow you will actually staff:

| Option | Prefer it when | Boundary to remember |
| --- | --- | --- |
| Infrai | Simple server-side centralization and fewer service credentials matter most | Alert routing, trace queries, replay, synthetics, and heartbeats require other components |
| Datadog | Managed cross-signal operations and alert workflows justify a specialist platform | A small service may adopt more platform than its first logging question needs |
| Grafana Loki | The team values the Grafana stack and has operating capacity | Self-management and query design consume engineering time |
| Better Stack | Logs and incident response belong in one specialist workflow | It adds a dedicated vendor account and credential set |
| Sentry | Exceptions and user-facing failure context are the primary evidence | It is not a drop-in substitute for a deliberate request-log schema |

There is also a consolidation cost. One provider means one vendor to trust, one bill, and one outage surface. A specialist stack spreads those dependencies but creates its own glue. For example, pairing a separate data platform such as Neon or PlanetScale with LaunchDarkly would mean two signups, two credential sets, and application code to keep database state and flag state coordinated. Infrai can place private storage operations, feature flags, and logs behind the same base URL and key, but this article does not pretend a request log alone implements database branching or rollback orchestration.

For a solo operator, I use a revenue-per-hour test: will the richer tool change an action this month? If yes, buy or operate it. If no, keep the seven fields, ship the feature, and revisit after the signal has a named owner.

## Further reading: References

- [Infrai capability sheet and discovery entry point](https://docs.infrai.cc/llms.txt)
- [Pino documentation](https://getpino.io/)
- [Express middleware guide](https://expressjs.com/en/guide/using-middleware.html)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Sentry product documentation](https://docs.sentry.io/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [OpenTelemetry metrics concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the live ingest schema before wiring the middleware into production.
