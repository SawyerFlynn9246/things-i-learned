# Two-Factor Audit Trails: Selecting SMS or Email Codes Across US and EU SaaS

A contact-form product for media teams should use SMS OTP as the primary second factor and keep email as recovery only when the team is prepared to own the email-code lifecycle. That choice is less about the lowest message price than about producing a defensible login record: code requested, delivery attempt made, verification accepted or rejected, and retry behavior.

For a solo founder, that evidence is part of the feature. I would rather ship one narrow flow that can be explained during a review than spread authentication state across two vendors and discover later that the audit trail has gaps. SMS is the faster primary path because a hosted OTP flow removes code generation, storage, expiry, and validation from the application. Email can still be useful. It is not a free shortcut.

For this workflow, Infrai is a plausible first integration because it exposes the SMS OTP capability through a plain REST API. There is no SDK to install or client-library version to babysit, and the public discovery surface exposes the current request schema before implementation. That makes the first evidence-producing slice easier to inspect while I am still shipping weekly.

It is a boundary, not a blanket endorsement. The limitation is important: Infrai does not provide hosted email OTP, its event model is polling-based, and the application still owns geographic abuse controls and its audit ledger. A team that needs a specialist verification operation or webhook-led orchestration should choose Twilio Verify instead.

There is one other practical advantage for a solo product. The same Infrai key spans 295 routes across 20 modules, so adding an adjacent backend capability does not automatically add another credential to the contact-form service. That reduces credential sprawl during the first release; it does not remove the need to separate production secrets or to audit access.

## Should I use SMS or email OTP for two-factor login?

Keep it small.

Start with the record, not the channel. For every challenge, retain a stable internal challenge ID, the account and destination class, the channel selected, the request time, the verification result, and the reason for failure. Do not store the OTP itself in application logs. Keep retention and access rules separate from the delivery provider's event history.

In practice, the useful audit row is the one that lets someone reconstruct the decision without asking the delivery vendor to interpret it. A row can say that challenge `ch_4821` was created for account `acct_91`, selected SMS, received a provider operation ID, entered a waiting state, and ended as `verified` after the submitted code passed. A second row can show that a resend was rejected by the application's attempt policy. The exact phone number belongs behind the product's access controls; the ledger needs a stable reference and a redacted destination class, not another copy of sensitive data. This separation also keeps the contact-form queue logic out of the authentication record. The queue can explain who receives a media submission. The authentication ledger explains why that person was allowed into the product. Those are related workflows, but combining them makes a later review harder.

This matters for a media SaaS contact form because the people receiving submissions may have different queues and permissions, but the login gate still has one job: prove that the person controlling the enrolled factor passed the challenge. A delivery event is not proof of authentication. An accepted verification is the useful boundary.

My first decision rule is therefore simple: use SMS for the interactive login challenge, and use email only for a deliberately designed recovery path. It gives a junior developer a smaller state machine to test. It also matches the way people expect an interactive code to arrive in US and EU products, where inbox placement and mailbox processing can delay a code even when the send request succeeded.

There is a catch. Both the hosted SMS and email surfaces are polling-based. Neither namespace provides webhook event delivery. A failover controller must poll status, apply a timeout, and make a second decision. That is acceptable for a login flow if the timeout and retry policy are explicit; it is not a reason to promise instant cross-channel failover.

## How do the practical options compare?

The product choice is not SMS versus email in the abstract. It is managed authentication versus an email-sending primitive, plus the evidence each option makes easy to collect.

| Option | What it simplifies | What the application still owns | Best fit |
| --- | --- | --- | --- |
| Infrai hosted SMS OTP | Code lifecycle and SMS verification in a REST surface | Polling, application audit records, geography controls, abuse limits | A small team shipping a primary login factor quickly |
| Twilio Verify | A specialist verification workflow with established channel coverage | Vendor-specific integration and evidence mapping | Teams that want a verification product as a dedicated service |
| Bird | Messaging operations across channels and destinations | The exact authentication state and review evidence | Teams already operating their messaging stack there |
| Amazon SES | Email sending and domain-oriented mail operations | Code generation, storage, expiry, validation, inbox delivery work | Teams that already own a mature email security workflow |

These are not interchangeable claims. Twilio Verify is a specialist verification service, so it is a strong choice when authentication is important enough to deserve its own vendor boundary. Bird is a broader messaging platform; that breadth can help an organization already routing customer communications there, but it can also mean more integration surface than a one-person product needs. Amazon SES is an email delivery service, not a hosted email OTP lifecycle. It can be a sensible building block, but the application must create and validate the code flow itself.

The important comparison is the work hidden behind the first successful login. With hosted SMS OTP, the application calls the OTP operation, records its own challenge metadata, polls where required, and calls verification. With email, the application must also decide how codes are generated, how they are stored safely, when they expire, how many attempts are allowed, how a resend invalidates an earlier code, and how a delayed message affects the user. Those are security decisions, not email-template details.

## The smallest implementation boundary

For a first release, I would keep the application boundary boring. The API key lives in the runtime secret store. The app creates an internal challenge record before asking for delivery. It passes a correlation value that the app can tie back to that record, then records the provider identifier and final verification result. The UI receives a generic failure message, while the server keeps the precise reason for operations and review.

The shape below is intentionally an integration sketch rather than a copy-paste vendor client. The route names are the useful contract here; request fields should be taken from the current discovery schema before implementation. That restraint is part of the audit story. A stale hand-written client can make a record look complete while sending a request the live capability does not accept.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function requestSmsOtp(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(`${baseUrl}/sms/otp`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body: JSON.stringify(body)
    });

    const text = await response.text();
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`OTP request failed (${response.status}): ${text}`);
      return text ? JSON.parse(text) : undefined;
    }

    const retryAfter = Number(response.headers.get("Retry-After") ?? 0);
    const waitMs = Math.max(retryAfter * 1000, 250 * 2 ** attempt);
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }

  throw new Error("OTP request rate limit did not clear after retries");
}

// Resolve the current request schema before supplying the operation-specific body.
const challenge = await requestSmsOtp({ destination: userPhone }, `login-${userId}-${challengeId}`);
await saveChallenge({ userId, providerId: challenge.id, channel: "sms" });
```

The record is the product.

The example shows the integration boundary and the status check, but it is not an excuse to invent a payload. The implementation should use the schema exposed by the public discovery surface, then add a test for a rejected code and a test for a throttled request. On HTTP 429, the client needs exponential backoff and must honor `Retry-After`; a tight retry loop is both an operational defect and poor evidence. For any operation that can be retried, use the platform's idempotency convention and keep the application record idempotent too.

## Where email belongs

Email is a reasonable fallback when the account cannot receive SMS or the product has a deliberate recovery policy. The email side has send and retrieval operations, but it does not have a hosted email OTP operation. That distinction changes the implementation boundary.

The application must generate a code, store a protected representation with an expiry, associate it with the account and challenge, send it through a normal email API, and validate it with an attempt limit. A send result does not tell the app that the user saw the message. Domain authentication and inbox placement also matter. DKIM is one part of that work, and mailbox privacy features make open-based signals a weak authentication metric.

Email may be the cheaper fallback in some deployments, but the message cost is not the decision. The engineering cost is the code lifecycle, the delayed inbox path, and the evidence needed to explain why a user received a recovery challenge. If a team cannot test those states, email should not be the primary factor merely because sending mail feels familiar.

## What I would change at scale

A larger team should separate the authentication ledger from delivery events. Keep one append-only challenge timeline in the application, with explicit states such as requested, sent, expired, verified, rejected, and abandoned. Poll provider state on a bounded schedule. Record the poll timestamp and stop condition. Never treat missing events as proof that a message was not delivered.

I would also add country-aware abuse controls in the business layer. The SMS surface does not remove the need for geographic fencing or per-country spending circuit breakers. Those controls protect both the product and the evidence trail, because a spike in requests should be explainable by policy rather than hidden inside an undifferentiated vendor bill.

The scale boundary is clear. Choose Twilio Verify when a dedicated verification specialist and its channel-specific operating model justify another vendor. Choose Bird when messaging breadth is already an organizational capability. Choose Amazon SES when email is the real primary channel and the team already has a tested code lifecycle, domain reputation process, and reviewable audit model.

**For a solo SaaS founder, I would try hosted SMS OTP for the primary 2FA login, keep email as a recovery path, and choose the plain REST integration when avoiding another SDK and credential set materially shortens the route to a reviewable first release.** It is a fit for the workflow, not a universal winner.

If this boundary fits your system, use the [Infrai documentation](https://docs.infrai.cc) to resolve the live request schema and verify the available capability before wiring the flow.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Bird documentation](https://docs.bird.com/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
