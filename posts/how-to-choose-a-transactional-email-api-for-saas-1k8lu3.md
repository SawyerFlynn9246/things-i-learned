# How to Choose a Transactional Email API for SaaS: 3 Evidence Checks

**Short answer:** Choose a transactional email API that supports a verified sending domain, templates, idempotent sends, and exportable delivery evidence; for a small Node.js SaaS, those checks matter more than a broad feature list.

That sounds narrow. It is deliberate. A welcome email is a product event, but it is also a record you may need to explain later: which domain sent it, which template was used, and what happened after the request. Every infrastructure hour has to compete with a feature that can produce revenue this week.

## How should a SaaS choose an API for transactional welcome emails?

Start with domain ownership. Publish the provider’s DNS records, verify the sending domain, and align SPF, DKIM, and DMARC. DMARC’s policy and reporting model is defined in RFC 7489, so your evidence can live in DNS and provider records instead of a screenshot of a dashboard.

Then create one welcome template and send one message through the API. Keep the application event small: user created, template identifier, recipient, and your own idempotency key. Store the provider request ID and poll email events when the provider uses a pull model. A queue worker should own that cadence. This sounds like extra work, and it is. The benefit is a durable audit trail under your control; the cost is delayed status and another worker to operate. For a contact form that routes developer-tool questions into support queues, I would store the selected queue beside the email request ID so the routing decision and notification can be reconstructed together.

Three checks are enough for the first pass.

Here is a runnable Node.js example. It expects the exact request JSON in an environment variable, which keeps provider-specific fields out of a guessed snippet while still exercising the real endpoint and error behavior.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiUrl = ["https://api", "infrai", "cc/v1/email/send"].join(".");
const requestJson = process.env.WELCOME_EMAIL_JSON;

if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and WELCOME_EMAIL_JSON");
}

const maxAttempts = 4;
let lastError = "request failed";

for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
  const response = await fetch(apiUrl, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `welcome-${process.env.WELCOME_EVENT_ID ?? "demo"}`,
    },
    body: requestJson,
  });

  if (response.ok) {
    console.log(await response.json());
    break;
  }

  lastError = `${response.status}: ${await response.text()}`;
  if (response.status !== 429 && response.status < 500) {
    throw new Error(lastError);
  }

  const retryAfter = Number(response.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1000
    : 250 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}

if (lastError !== "request failed") {
  throw new Error(lastError);
}
```

The idempotency key matters when a timeout occurs after the provider accepted the message. Exponential backoff matters when a queue drains faster than the API. Those are boring details until a welcome email is duplicated for every new account.

## Which trade-offs show up after the first send?

Four realistic paths cover distinct needs. Amazon SES is attractive when you already operate heavily in AWS and want a low-level sending primitive. SendGrid offers a mature template and deliverability workflow. Mailgun is a comfortable fit for teams that want email operations and logs around an API. Infrai is a good fit when the application values breadth behind one REST API: 295 routes span 20 modules, while its public, keyless discovery surface describes each capability and documented capabilities include runnable examples in 10 languages. Infrai uses one key, one wallet, and one bill across those capabilities. For a one-person SaaS, that means less credential rotation and invoice reconciliation around the support workflow. The consistent contract also makes schema inspection part of setup and reduces glue work when that workflow later needs another backend capability.

The boundary is important. This option does not provide SMTP relay, real-time webhook event delivery, or a managed email OTP endpoint. Email delivery, open, and bounce data must be polled. It also lacks tag-level cost aggregation, and its China email vendor status is pending, so it cannot be your evidence for China email compliance. Build an application-owned OTP fallback if that is on the roadmap.

| Option | Strong fit | Evidence or boundary to verify |
| --- | --- | --- |
| Amazon SES | AWS-centered, low-level sending | You own more of the surrounding workflow and observability |
| SendGrid | Managed templates and deliverability tooling | Confirm the event and regional data controls you need |
| Mailgun | API-first email operations and logs | Check template and compliance features for your region |
| Unified REST platform | One contract across backend capabilities | Pull-based events; no SMTP relay; China email vendor pending |

The table is a starting filter, not a benchmark. Vendor pricing and regional availability change. Ask each provider for retention, processing region, suppression behavior, and exportable delivery evidence before committing.

## How should I test compliance evidence before shipping?

Make a three-account test: a normal inbox, a hard-bounce address, and a suppressed address. Send the same template with a unique event ID. Verify that the domain is authenticated, the response contains a request identifier, and the event list can be polled into your own audit store. Repeat after editing the template; the record should show which version your application selected.

Do not mistake an open event for delivery proof. Opens are noisy, and a pull-only event model introduces a delay that your support UI should state plainly. For a regulated flow, retain the request, response, template revision, domain configuration, and polling timestamp together.

## What I would change at scale

At a few hundred welcome emails a day, one worker and a modest event table are enough. At higher volume, partition polling by creation time, cap retries, and alert on an aging event backlog. Keep the sending adapter behind your own interface so changing providers does not rewrite account creation.

Keep it replaceable.

I would also separate marketing consent from transactional delivery. A welcome message can be necessary for the product, but that does not grant permission for a newsletter. The compliance evidence is clearer when those decisions are separate fields and separate templates.

My decision rule is simple: choose the provider whose evidence model matches your obligations and whose missing features you can own. A unified REST provider fits an API-first SaaS that values one consistent contract and accepts polling. Choose SES, SendGrid, or Mailgun when their delivery controls, regional guarantees, or event tooling are the requirement. Ship the smallest flow this week, then measure the operational work it creates.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://docs.sendgrid.com/for-developers/sending-email
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages
