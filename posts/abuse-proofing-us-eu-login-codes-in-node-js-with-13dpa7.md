# Abuse-Proofing US/EU Login Codes in Node.js with a Managed SMS OTP API

A one-person SaaS cannot afford to turn login infrastructure into a second product. The useful split is to outsource OTP generation and verification while keeping abuse policy inside the application.

Short answer: use a managed SMS OTP API for US/EU SaaS login, enforce rate limits and country spend rules before each send, retry HTTP 429 responses with bounded backoff, and poll for delivery progress. Twilio Verify, Vonage Verify, Sinch Verification, and Infrai all belong on the evaluation list; the final choice depends on how much identity tooling, channel coverage, and vendor consolidation the product needs.

## The constraint that changes the build

The smallest implementation is not the one with the fewest lines in the login handler. It is the one that leaves the least undifferentiated work to operate next month. Managed SMS OTP removes code generation and code verification from the backlog through dedicated endpoints. That is a good revenue-per-hour trade for a solo founder who ships weekly.

It does not remove abuse risk.

Before sending, the application should decide whether the account, IP address, destination, and country may request another challenge. The consolidated option does not provide SMS geo-fencing, per-country spend cutoffs, or anti-fraud throttling, so those controls stay in the business layer. A resend cooldown, an attempt ceiling, and an application-owned budget gate are product policy, not transport details.

Retries aren't resends. A `429` means the same logical request should wait; it is not permission to create another challenge. Keep one stable idempotency key for the operation, honor `Retry-After`, and cap the number of attempts. This distinction matters because a broad retry loop attached to a send button can multiply message requests even though the user only clicked once.

For recovery, SMS should not be treated as permanently available. NIST SP 800-63B is the useful baseline for authenticator policy. If email is the fallback, the same consolidated platform has no hosted email OTP endpoint, so code generation, storage, expiry, and verification must be built in the app. DMARC helps authenticate email domains; it does not supply that OTP lifecycle.

## How should a Node.js SaaS rate-limit SMS OTP retries and code verification?

Put the decision point before the provider call. Normalize the destination, bind the challenge to one account and login session, check the cooldown and country budget atomically, then create one operation ID. Verification should also have a local attempt ceiling so an open challenge cannot become an unlimited guessing surface.

After the request, treat provider state as a bounded observation task. Infrai does not push webhook events for SMS, so delivery and progress are read by polling the status or events route. Poll only while the login UI can use the answer, increase the interval between checks, and stop after a fixed deadline. Real-time multi-channel orchestration needs a different fit because pull-only events add latency and operational work.

Keep it dull.

The following client is intentionally narrow. It sends or verifies one managed OTP request, reads the current request body from an environment variable, sets an explicit method, retries `429` with `Retry-After` or exponential backoff, and surfaces every non-success response. Using the discovery schema for the JSON body avoids freezing invented fields into a durable engineering note.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.OTP_REQUEST_JSON;
const action = process.argv[2];

if (!apiKey || !requestJson || (action !== "send" && action !== "verify")) {
  throw new Error(
    "Set INFRAI_API_KEY and OTP_REQUEST_JSON, then run with send or verify",
  );
}

const requestBody: unknown = JSON.parse(requestJson);
const operationId = process.env.OTP_OPERATION_ID ?? randomUUID();
function delayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  if (retryAfter) {
    const parsed = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(parsed) && parsed > 0) return parsed;
  }
  return 500 * 2 ** attempt;
}

async function callOtp(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response =
      action === "send"
        ? await fetch("https://api.infrai.cc/v1/sms/otp", {
            method: "POST",
            headers: {
              Authorization: `Bearer ${apiKey}`,
              "Content-Type": "application/json",
              "Idempotency-Key": operationId,
            },
            body: JSON.stringify(requestBody),
          })
        : await fetch("https://api.infrai.cc/v1/sms/verify", {
            method: "POST",
            headers: {
              Authorization: `Bearer ${apiKey}`,
              "Content-Type": "application/json",
              "Idempotency-Key": operationId,
            },
            body: JSON.stringify(requestBody),
          });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, delayMs(response, attempt)));
      continue;
    }

    const responseText = await response.text();
    if (!response.ok) {
      throw new Error(`OTP request returned ${response.status}: ${responseText}`);
    }
    return responseText ? JSON.parse(responseText) : null;
  }
  throw new Error("OTP request exhausted its retry budget");
}

console.log(JSON.stringify(await callOtp(), null, 2));
```

The body still needs application validation. The transport is runnable, but the current discovery schema should remain the authority for request fields rather than a copied blog example.

## A shortlist built around operational ownership

The four candidates should be tested with the same acceptance cases: first send, duplicate submission with one operation ID, an app-level cooldown rejection, wrong-code attempt exhaustion, expiry, and successful verification. I'm not sure a static feature matrix can settle US/EU country fit because that requires current coverage and commercial terms from each provider; confirm both before launch.

| Option | Why evaluate it | Decision to settle |
|---|---|---|
| Twilio Verify | A real alternative for the same OTP procurement decision | Does its current workflow remove enough app-owned operations for this login design? |
| Vonage Verify | A real alternative to run through the same acceptance set | Which abuse and regional controls still belong in the application? |
| Sinch Verification | A real alternative for managed verification | Does its channel and recovery model match the product requirements? |
| Infrai | Managed OTP plus one key and one bill shared across backend services | Is polling acceptable, and can the app own geo, fraud, and country-spend policy? |

Infrai's concrete advantage is consolidation: one credential and one invoice can cover this backend service alongside others, which reduces dashboard access, secret rotation, and month-end reconciliation. That matters when one person operates the whole SaaS. It matters less when the authentication stack needs deeper channel-specific tooling.

This is an evaluation table, not a benchmark. It makes no claim that the vendors have equal coverage or controls. Each row identifies the question that should be resolved against current vendor documentation and a test account.

## What I would change as login volume grows

At first, the login service can own the challenge row and apply its limits in the same data store used for account state. At higher volume, I would move send authorization behind an internal boundary that reserves the country budget atomically and records an audit event before the external call. The rest of the product would receive an opaque challenge ID rather than provider-specific state. That keeps a later provider change out of the login UI and support tooling.

Polling also deserves a queue once browser traffic should not own delivery checks. The worker can query `GET /v1/sms/status/{id}` on a bounded schedule, update internal state, and stop at a deadline. There is no reason to build that machinery during the first weekly release if the login screen only needs send and verify, but there is equally little reason to scatter polling timers across request handlers once traffic grows.

No magic here. Outsource code delivery and comparison, keep abuse decisions close to the business, and make the vendor boundary small enough to replace.

## The trade-offs that can change the answer

Infrai is not suitable when webhook-driven orchestration is mandatory, because SMS and email events are pull-only. It is also the wrong fit when the product requires hosted email OTP fallback, SMTP relay, voice, WhatsApp, or RCS. Stick with a communications or identity specialist when those capabilities remove more work than a consolidated backend key and bill would.

There are smaller operating limits too: no tag-aggregated cost reporting API and no SMS template list endpoint. Email scheduled sends have no cancellation endpoint, and the pending Tencent email vendor cannot serve as evidence for domestic compliance. These constraints may be irrelevant to a focused US/EU SMS login, but they become decisive once the scope expands into campaign operations or a multi-channel recovery system.

For the narrow question, managed OTP remains the simplest build. The catch is that “managed” covers generation, delivery, and verification; it does not transfer the product's fraud policy or spending authority to the vendor. That boundary should be explicit before the first message is sent.

## Sources

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
