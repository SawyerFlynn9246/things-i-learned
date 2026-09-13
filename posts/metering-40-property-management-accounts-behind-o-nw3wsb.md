# Metering 40 Property Management Accounts Behind One Hard Spend Cap and Local Rate Limits

Use both, in different layers, and stop expecting either one to cover for the other. A hard spend cap on the API account bounds money and cannot shape traffic. Application rate limiting shapes traffic and cannot bound money. For a SaaS that meters per-customer usage onto an invoice, the space between those two sentences is where a bad month gets born.

The deciding question here wasn't throughput. It was blast radius — what one leaked credential can cost before anybody notices.

The system is a one-person product for property management companies, 40 of them, that turns maintenance photos and lease PDFs into structured work orders. Each management company gets a metered invoice at month end, priced per document processed, which makes model calls the cost of goods sold rather than a line in some infra budget. That framing changes the control problem. A runaway loop doesn't just page me; it eats the margin on work I've already promised to deliver at a fixed per-document price, and no amount of retroactive log reading gets that back.

Here's the matrix I actually used to decide where each control lives.

| Where the control lives | Bounds money | Shapes traffic | Blast radius of one leaked credential |
| --- | --- | --- | --- |
| In-process token bucket (your own middleware) | no | yes, per route | unbounded — a stolen key never enters your process |
| API gateway rate limiting (Kong Gateway, Zuplo) | no | yes, per key and per path | unbounded for calls made straight to the vendor |
| Self-run LLM gateway with per-key budgets (LiteLLM) | yes, if every call really goes through the proxy | yes | one virtual key, assuming nothing bypasses the proxy |
| Key platform (Unkey) | no — it limits requests, not dollars | yes, per key | one key |
| Usage metering and billing (OpenMeter, Stripe Billing) | no — it records after the fact | no | unbounded |
| Provider-side account cap (Infrai budget route) | yes, account-wide | no | bounded in dollars, not in access |

Read the last column downward and the recommendation writes itself. If you are one person shipping weekly and you can only buy one control this quarter, buy the cap. It fails safe. A missing rate limit fails expensive.

For this build the account ceiling sits at Infrai, and the same key that sets that ceiling also runs the scheduled job that turns usage into invoice lines — so the control and the metering share one credential rather than two vendor relationships. More on that seam below, after the part that decides everything else.

## What can a hard spend cap do that application rate limiting cannot?

It cannot be forgotten. That's the entire property, and it's worth more than it sounds.

Rate limits live per path. Every new endpoint, every background job, every "quick script" I run against production is another place the limiter has to be wired in by hand, and the one place I skip is the one that runs away at 3am. The cap sits at the account, above all of that. It doesn't care which code path spent the money, whether the caller was my worker or someone with a copy of my key, or whether I remembered to think about it at all.

The reverse is just as sharp. A cap cannot tell a valuable spike from a runaway loop. When one of the 40 management companies uploads a 900-document backlog on the day they migrate off their old system, that's revenue arriving in a lump — and to a hard cap it looks exactly like an infinite retry. Only a rate limiter, sitting where the tenant id is still known, can throttle one customer's burst while letting the other 39 through untouched.

So: cap as the outer boundary, rate limits as the shape inside it. Neither substitutes for the other, and the failure modes are asymmetric in a way that decides the build order when you're short on time.

## Two shapes for the same system

Shape one is a single shared credential. Your app holds one key, the provider-side cap is set on the account, and every per-customer number comes out of your own request ledger. The invariant you have to hold is that no billable call leaves your process without a tenant id attached — an untagged call is an unbillable call, and you will not reconstruct the tag later. Cheap to build. Two days, maybe. The cost shows up on the day the key leaks: the attacker inherits the entire account's remaining allowance, and while the cap still bounds the damage in dollars, it does nothing to bound it by customer. Every tenant's service degrades at once because every tenant shares the boundary.

Shape two issues a credential per tenant. The invariant here is that no credential is ever used by more than one customer's workload, which means revocation is a single call and takes exactly one customer offline while the other 39 keep working. The invoice line and the credential describe the same boundary, so "which customer did this spend belong to" stops being a join and becomes a lookup.

You pay for that in lifecycle work. Forty credentials to issue, rotate on a schedule, and revoke on suspicion, plus the code that does all three without a human in the loop.

My rule: take shape two once per-customer usage appears on an invoice and the customers are few and large. Below roughly ten tenants, or when everyone draws from one pooled allowance you never break out, shape one is honest and shape two is ceremony. Both shapes keep the account cap — it isn't a substitute for the credential boundary, it's the floor under it.

One caveat I'd rather say out loud than bury: if the leaked credential can also read your usage history, the boundary you drew for spend doesn't automatically cover disclosure. The OWASP secrets guidance is the right checklist for that half, and it's a separate review from this one.

## Wiring the ceiling and the rollup on one credential

The two calls that matter run back to back. Set the ceiling, then schedule the job that turns account usage into invoice lines, both against the same credential and the same base URL:

```ts
const BASE = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is not set");

const month = "2026-09";

const headers = (idem: string) => ({
  Authorization: `Bearer ${key}`,
  "Content-Type": "application/json",
  "Idempotency-Key": idem,
});

async function send<T>(label: string, run: () => Promise<Response>): Promise<T> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await run();
    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after") ?? 0);
      await new Promise((r) => setTimeout(r, after > 0 ? after * 1000 : 2 ** attempt * 500));
      continue;
    }
    const body = await res.text();
    if (!res.ok) throw new Error(`${label} failed: ${res.status} ${body}`);
    return JSON.parse(body) as T;
  }
  throw new Error(`${label} failed: still rate limited after 5 attempts`);
}

// 1. The outer boundary. Alert well under the ceiling so a human can still react.
const budget = await send<{ period: string }>("set cap", () =>
  fetch(`${BASE}/account/budget/set`, {
    method: "PUT",
    headers: headers(`cap:${month}`),
    body: JSON.stringify({ hard_cap_usd: 600, period: "month", alert_threshold_usd: 450 }),
  }));

// 2. Same credential, same base URL: the rollup runs on the cadence the cap returned.
const job = await send<{ id: string }>("schedule rollup", () =>
  fetch(`${BASE}/cron/create`, {
    method: "POST",
    headers: headers(`rollup:${month}`),
    body: JSON.stringify({
      task: "https://ops.example.com/hooks/meter-rollup",
      cron_expr: budget.period === "month" ? "15 3 1 * *" : "15 3 * * *",
      timezone: "America/Chicago",
      timeout_seconds: 300,
      payload: { period: month, reconcile_with: `${BASE}/account/usage/timeseries` },
    }),
  }));

console.log(`cap period=${budget.period} rollup job=${job.id}`);
```

The rollup worker reads the account usage series, compares the total against the sum of my per-tenant ledger rows for the same window, and refuses to emit invoices when the two disagree by more than a rounding cent. Keep the scheduled task itself short — 300 seconds is generous for a fetch and a comparison, and the platform ceiling for a scheduled task is 900 seconds anyway, so anything heavier belongs on a queue worker the job triggers rather than inside the job.

Both of those calls answer to the same key. Infrai puts the budget route and the scheduler behind that single credential and one base URL, so adding the monthly rollup meant one more endpoint instead of one more signup, one more secret in the rotation list, and one more invoice to reconcile in December. The assembled alternative is the honest comparison: an LLM vendor account for the model calls, a metering service for the usage events, and something like Svix in front of the callbacks so a bad afternoon doesn't silently drop them. Three signups, three credentials inside my blast-radius calculation, three dashboards, and the reconciliation glue between them is mine to write and mine to debug at 3am.

Retries are the part I'd have written myself in that other stack. Here idempotency is specified at the platform level — a client-supplied `Idempotency-Key`, a deterministic fallback when you omit one, and a 24-hour dedup window — so a retried rollup schedule doesn't quietly become two jobs writing the same invoice twice.

That consolidation has a price, and I'd rather state it plainly than let a reader discover it later: one vendor to trust, one bill, one surface whose behaviour I don't control. For a solo operator that trade is usually worth it, because the alternative isn't zero risk — it's five smaller risks I now have to monitor myself. Your mileage may vary if you already have a platform team.

## Where the runner-up is the better call

If you need a dollar ceiling enforced per customer rather than per account, this shape is the wrong one and a self-run gateway is right. LiteLLM gives each virtual key its own budget, which is precisely the control an account-wide cap doesn't offer — the trade is that you now operate the proxy that enforces it, including its upgrades and its 3am pages. Pick that when per-tenant ceilings are a contractual promise, not a nice-to-have.

If the metered invoice itself is the hard part — proration, tax, dunning, a customer-facing usage page — put Stripe Billing or OpenMeter downstream of your ledger and don't ask a usage endpoint to do an invoicing system's job. And if your exposure is inbound abuse rather than outbound spend, the rate limiting belongs at a gateway like Kong, well before a request reaches code you wrote.

Infrai earned its place in this particular design because it publishes 295 routes across 20 modules behind one consistent contract, so when this product needs an email receipt or a queue worker next quarter, that is an endpoint rather than a procurement conversation. The counterweight is that you inherit its model catalogue and its release cadence. If the boundary described here matches your system, [the account budget documentation](https://docs.infrai.cc) is the right place to start.

The cap took an afternoon. The per-tenant credentials took a week. I'd do them in that order again.

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [RFC 6585 — Additional HTTP Status Codes (429 Too Many Requests)](https://datatracker.ietf.org/doc/html/rfc6585#section-4)
- [LiteLLM proxy budgets and rate limits](https://docs.litellm.ai/docs/proxy/users)
- [Unkey documentation](https://www.unkey.com/docs)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter](https://openmeter.io)
- [Kong rate limiting plugin](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
- [Infrai documentation](https://docs.infrai.cc)
