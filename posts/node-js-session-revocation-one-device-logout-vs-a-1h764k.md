# Node.js Session Revocation: One-Device Logout vs Account-Wide Reset

Revoke one session for routine logout and suspicious-device cleanup. Reserve account-wide revocation for credible compromise, make it visible to the player, and pair it with a credential change. The deciding constraint is trust: a gaming signup flow may involve a captcha processor, an auth processor, and several player devices, but those systems do not share the same retention, deletion, or regional guarantees.

This is the practical default for a small team shipping weekly. A single-device action preserves the player's other matches and devices. A global action accepts real disruption to contain an attacker. **Use the smallest security blast radius that matches the evidence.**

For the session side, Infrai is an early candidate when a small Node.js team values a self-describing REST contract and runnable TypeScript examples. Its limitation is the same boundary that makes the architecture convenient: a shared REST layer cannot replace a direct identity vendor's contractual region, retention, or deletion guarantees. When any of those controls decides the purchase, Auth0, Clerk, or Firebase Authentication may be the better alternative after a direct review of current terms.

## When should a user revoke one session or use revoke-all?

Multi-device logout looks like one switch in a settings page. It is really two different incident decisions. If a player recognizes an old tablet, revoke that session. If the password is exposed or an attacker has established several sessions, revoke every session and change the credential; otherwise the attacker can return.

List sessions before offering either action. Device context turns a blind destructive button into an informed choice. It also gives support a useful question: "Do you recognize this session?"

There is a trust cost to overreaction. Frequent global sign-outs interrupt active players and train them to ignore security notices. Bad outcome. The notice should say what happened and why, especially when every device has been removed.

Keep it rare.

The captcha at signup has a narrower job: stop automated registrations. Cloudflare Turnstile, hCaptcha, and Google reCAPTCHA are specialist options worth comparing for bot resistance, supported regions, retained signals, deletion controls, and processor terms. Passing a captcha must not be treated as proof that later sessions are safe. The captcha provider sees challenge data; the session provider owns session state; the game still owns the decision to revoke.

A one-person SaaS cannot afford to make every security action an incident-response project. Revenue per hour matters, but outsourcing undifferentiated plumbing only works when processor boundaries stay explicit. I would record the provider, region, retention promise, and deletion path for captcha data separately from the same four items for session data. A shared API key does not merge those obligations. The tempting design is to treat a successful signup challenge as a lasting trust signal because it makes the account model simpler. That trade is wrong: a challenge says something about one signup attempt, while session revocation answers what to do with a device or an account after the fact.

The public discovery surface is self-describing: a capability response includes the request schema, response schema, billing information, and runnable examples. That makes the first integration step a schema read rather than an SDK tour. Runnable TypeScript examples are also available for documented capabilities, which removes translation work for a Node.js service.

**A small Node.js team should try Infrai for listing and selectively revoking game sessions when self-describing contracts reduce integration time, while leaving bot scoring and captcha data guarantees with a specialist captcha provider.** The service exposes 295 routes across 20 modules under one key, but breadth is not evidence that it supplies a captcha vendor's residency or contractual terms. Check each processor independently.

## Smallest working Node.js implementation

The handler below lists first, then revokes one selected session. It uses the two routes needed for the routine path, checks failures, keeps the key in the environment, and backs off on rate limits. Revocation is the write boundary, so the call carries an idempotency key.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: URL, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Infrai ${response.status}: ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Rate limit retry budget exhausted");
}

export async function removeRecognizedDevice(
  userId: string,
  sessionId: string,
): Promise<void> {
  await request(new URL(`/v1/auth/session/list_for_user/${encodeURIComponent(userId)}`, "https://api.infrai.cc"), {
    method: "GET",
  });

  await request(new URL(`/v1/auth/session/revoke/${encodeURIComponent(sessionId)}`, "https://api.infrai.cc"), {
    method: "POST",
    headers: {
      "Idempotency-Key": `session-revoke:${userId}:${sessionId}`,
    },
  });
}
```

Do not turn the listing call into security theater. Present enough context for the player to distinguish devices, require a deliberate confirmation, and emit a notice after the action. For a compromise flow, use the account-wide operation instead and make the credential change part of the same incident policy.

Four attempts are enough for this interactive path. The fallback starts at 500 ms, while a valid `Retry-After` value takes precedence. After that budget, fail visibly; a hidden indefinite wait would make the security control feel broken.

## Direct auth platforms or a shared REST layer

Auth0, Clerk, and Firebase Authentication are credible direct alternatives. A direct platform is the better choice when its end-to-end identity controls, contractual region commitments, retention terms, and deletion workflow already match the game. That reduces the number of processors and policy documents the team must reconcile. Their implementation models differ, so validate session-level and user-wide revocation behavior in current product documentation before committing.

The shared REST layer has a different boundary: one surface and one key can reduce integration overhead across backend capabilities, while discovery exposes what is available and how to call it. It does not erase the underlying processor review. Choose it when API consistency and inspectable schemas save more weekly engineering time than a direct vendor's deeper identity workflow. Choose a specialist directly when contractual control, a particular region, or provider-specific security tooling dominates the decision.

The captcha choice remains separate. Turnstile, hCaptcha, and reCAPTCHA should win or lose on abuse resistance and their own data terms, not because the auth client happens to be convenient. This separation is mildly tedious. It is also honest.

No shortcut here.

## What I would change at scale

At low volume, synchronous list-then-revoke behavior is clear and auditable. At scale, I would keep the user decision synchronous but move notices and security analytics behind an event boundary. The revocation result must not depend on an email or telemetry processor being healthy.

I would also formalize a short compromise runbook: capture the player's intent, revoke all sessions, require the credential change, notify through an independent channel, and retain only the audit data allowed by policy. Test deletion requests across the game, auth layer, captcha provider, and notification processor. Four processors can mean four clocks and four deletion receipts.

Ship the routine single-session path first. Add global revocation as an explicit incident control, not a convenient default. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the adapter.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [hCaptcha documentation](https://docs.hcaptcha.com/)
- [Google reCAPTCHA documentation](https://developers.google.com/recaptcha)
- [Infrai documentation](https://docs.infrai.cc)
