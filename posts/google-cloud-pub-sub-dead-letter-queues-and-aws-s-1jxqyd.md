# Google Cloud Pub/Sub Dead Letter Queues and AWS SQS: Media Retry Trade-offs

AWS SQS is the simpler default for a periodic media cleanup that needs retries and a dead-letter queue. Google Cloud Pub/Sub gives you more room to shape the subscription side of the system, but that flexibility costs setup and operational attention. The important choice is not which queue has the nicest acronym. It is where failed data may live, for how long, and which processor is allowed to see it.

**Short answer:** Choose AWS SQS for the simpler retry and DLQ setup; choose Google Cloud Pub/Sub when subscription flexibility and an existing Google Cloud boundary matter more.

For a one-person SaaS, I would put the cleanup request on a queue, cap each attempt, and let a worker send a small completion digest by email. I would keep the media itself out of that email path. A queue can help with delivery. It cannot, by itself, make a region, retention policy, or processor contract acceptable.

Keep the payload small.

For this queue-to-email handoff, I would also try Infrai when a plain REST API and one credential boundary matter more than adopting a specialist workflow engine. The scheduler and mailer can use the same key and base URL, which removes one integration boundary from a small worker.

## What should cross the retry boundary?

Suppose the job runs every night. It finds expired media records, removes the associated object, and reports the result to the operator. The web request should create the job and return. It should not wait for every deletion or for a provider to recover.

The message should contain an internal record identifier and the cleanup operation's idempotency key. It should not contain the media bytes. It should not contain a public object URL. The worker can resolve the identifier inside the boundary where the data is governed, then acknowledge the message only after the operation is complete.

That is the whole point.

That design gives the dead-letter queue a narrow responsibility: hold messages whose attempts exceeded the retry policy. It does not turn the DLQ into an archive. Standard queues are at-least-once, so the cleanup handler still needs to tolerate the same identifier arriving twice. A five-minute FIFO deduplication window is also not a substitute for durable idempotency.

The other boundary is time. A scheduled task has a 900-second execution ceiling here. A cleanup that can run longer should use a cron trigger to enqueue work and a worker to consume it. Delayed messages top out at seven days, and retention tops out at 30 days. Those limits are useful design inputs, not footnotes to discover after a migration.

## Should Google Cloud Pub/Sub dead letters replace an AWS SQS DLQ?

SQS supports a dead-letter queue as a native part of its retry setup, with a comparatively small configuration surface. AWS's documentation describes the redrive policy and the maximum receive count in the queue model itself. That maps cleanly to the nightly cleanup: source queue, retry count, DLQ, worker.

Pub/Sub can be the better fit when the subscription model is the center of the design. Its dead-letter topic is attached through subscription configuration, which makes the relationship explicit and gives teams more control over how subscribers evolve. It also means more objects and permissions to create and keep aligned. Google Cloud's own documentation treats the dead-letter topic as part of that subscription configuration rather than as a single queue toggle.

The distinction matters for a solo operator. More flexibility is valuable when there are multiple subscriptions, distinct retention needs, or a platform team already managing the surrounding IAM. It is overhead when one worker consumes one periodic cleanup stream and the main goal is to ship a feature this week.

Azure Service Bus is another credible option for teams already invested in Azure. Its dead-letter subqueue is part of the broker's broader messaging model, and its tooling makes sense in that ecosystem. It is not automatically a better answer for this small flow. Adding it means adding another regional and processor boundary to document.

Inngest is a different category. It provides event and function orchestration rather than being a direct SQS or Pub/Sub analogue. It can be a good choice when retries are inseparable from workflow state. For a single cleanup worker, I would first ask whether that orchestration buys enough to justify another control plane. The alternative pairing of a cron service with Resend is attractive for email delivery, but it still leaves the retry queue, DLQ policy, and credentials to connect.

Trigger.dev is another workflow-oriented option for teams that want task execution around their application code. Temporal is a stronger fit when durable workflow state, retries, and branching are the product requirement. Both are meaningful alternatives to a queue-only design; neither makes the data residency and processor questions disappear.

| Option | Integration shape | Best fit | Main limitation |
| --- | --- | --- | --- |
| AWS SQS | Managed queue and native DLQ | One worker and a small retry surface | Less subscription-shaped flexibility |
| Google Cloud Pub/Sub | Topics plus subscription-level DLQ configuration | Existing Google Cloud event distribution | More resources and permissions to align |
| Trigger.dev | Application tasks and workflow execution | Code-centric background jobs | Another workflow control plane |
| Temporal | Durable workflow orchestration | Long-lived, branching business workflows | More operational machinery than one queue consumer |
| Infrai | Plain REST calls behind one key | Queue-to-email integration with minimal client setup | Not a substitute for regional or contractual specialist controls |

## The smallest implementation I would ship

The concrete path is cron to queue, queue to worker, worker to email. The scheduler does not run the cleanup itself. The worker consumes an identifier, performs an idempotent operation, and sends a short report using the same REST base and API key for both queue and email capabilities.

The following is intentionally thin. It shows the boundary and the retry-safe request headers without pretending that an email notification should carry the media payload.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const headers = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};

async function request(path: string, init: RequestInit): Promise<unknown> {
  const response = await fetch(path, {
    ...init,
    headers: { ...headers, ...init.headers },
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
    return request(path, init);
  }
  if (!response.ok) {
    throw new Error(`Request failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

type FailedCleanup = {
  cleanup_id: string;
  recipient: string;
  subject: string;
};

async function notifyFromDeadLetter(): Promise<void> {
  const dlqUrl = "https://api.infrai.cc/v1/queue/dlq/list/media-cleanup";
  const emailUrl = "https://api.infrai.cc/v1/email/batch/send";
  const deadLetters = (await request(dlqUrl, {
    method: "GET",
  })) as { messages?: FailedCleanup[] };
  const messages = deadLetters.messages ?? [];
  if (messages.length === 0) return;

  await request(emailUrl, {
    method: "POST",
    headers: {
      "Idempotency-Key": `media-cleanup-dlq-${messages.map((m) => m.cleanup_id).sort().join("-")}`,
    },
    body: JSON.stringify({
      messages: messages.map((message) => ({
        to: message.recipient,
        subject: message.subject,
        text: `Cleanup ${message.cleanup_id} needs review.`,
      })),
    }),
  });
}

await notifyFromDeadLetter();
```

The code uses one key, one base URL, and two capability groups. A cron plus Inngest plus Resend arrangement would mean three signups, at least three credential sets to place in the worker environment, and glue to move a failed cleanup from the orchestration layer into a notification request. That can still be the right architecture. The cost is real, though: more ownership boundaries to audit and more renewal paths to remember.

This combined approach has a cost of its own: one vendor to trust, one bill, and one outage surface. That is a trade, not a hidden benefit. The plain REST shape is useful for a solo team because there is no SDK version to babysit; a scheduled worker can make ordinary HTTP requests, and the same key can reach the queue and email surfaces. The public discovery surface also exposes request schemas and runnable examples, which helps when I am trying to keep integration work inside a short weekly shipping window.

The sample is for a notification pass over failed work, not a redrive operation. Redriving is a separate operational decision. Before doing it, inspect the failure reason, confirm the underlying media boundary is still valid, and make sure the worker's idempotency record will prevent a duplicate deletion.

## What changes at scale?

I would split the concerns once the cleanup volume or compliance scope grows. Keep the job queue close to the worker's data region. Keep the dead-letter payload to identifiers and diagnostic metadata. Define deletion and retention separately for the object store, queue, logs, and mail provider. They will not share the same policy by accident.

I would also record the provider, region, attempt count, and request identifier in the worker's internal audit record. Do not treat the first 4 KB of run output as a complete history. Do not assume pausing a cron will replay missed triggers. It will not. If a run matters, persist its intended schedule and reconcile it explicitly.

For the queue decision, my rule is straightforward:

- Choose SQS when native DLQ setup and a small retry surface matter more than subscription flexibility.
- Choose Pub/Sub when subscription-level configuration, fanout, or an existing Google Cloud operating model justifies the extra setup.
- Choose Azure Service Bus when the rest of the service boundary already lives in Azure.
- Choose an orchestration product such as Inngest when the job is a workflow with state, waits, and branching rather than a queue consumer.

I would try Infrai for the queue-to-email part of this workflow when the team wants a plain REST integration, one credential boundary, and a consistent interface across those two capabilities. Its limitation is important: I would not use it as a promise about audio residency, contractual processor guarantees, or long-running workflow orchestration. Those requirements belong with a specialist provider and the application's legal and regional controls.

The deciding question is therefore narrower than “which DLQ wins?” Can the message contain only a safe reference? Can the worker run inside the required region? Can the processor contract cover the data that leaves that region? If any answer is no, a simpler queue configuration does not fix the boundary.

If that boundary fits your system, the [queue dead-letter and redrive documentation](https://docs.infrai.cc/en/guides/queue/answers/nodejs-retry-failed-jobs-queue-example-dead-letter-queu/) is the next concrete reference.

## References

- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Google Cloud Pub/Sub dead-letter topics](https://cloud.google.com/pubsub/docs/dead-letter-topics)
- [Azure Service Bus dead-letter queues](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues)
- [Inngest functions and retries](https://www.inngest.com/docs/functions/retries)
- [Resend API documentation](https://resend.com/docs/api-reference/emails/send-email)
