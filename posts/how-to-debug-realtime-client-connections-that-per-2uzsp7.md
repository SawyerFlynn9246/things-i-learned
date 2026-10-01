# How to Debug Realtime Client Connections That Persist After Logout

TL;DR: Revoking a shopper's token does not close a connection that is already open. A reliable support-chat logout must revoke future authorization, disconnect the real client identity, and then check the room's presence state. Treat that sequence as one application contract, even though it crosses three realtime operations.

This distinction matters in an e-commerce chat room. An agent may use the roster to decide whether to wait, close a conversation, or help the next shopper. A green dot that survives logout is not harmless decoration. It is false operational data.

The practical rule is short: **authorization controls the next connection; identity controls the current one**.

## Why is the realtime client still connected after logout?

The socket has already been authorized. Revocation changes what happens at the next authorization check, but it does not travel backward in time and undo the check that admitted the current socket. Expecting it to close that socket merges two separate lifecycle events.

This is the concrete constraint that changes the design. Logout cannot be modeled as "invalidate credential and clear browser state." The server-side workflow also needs a stable identity that can be disconnected. If the issued token does not carry a real identity, there is no reliable target for that second action.

Anonymous labels are especially risky. Imagine `guest` appearing in two tabs for `order-help-731`, while another browser also joins as `guest`. The label says nothing about which shopper should leave. Use the same stable identity when issuing the token, disconnecting the shopper, and checking presence. For the examples below, that identity is `shopper_1842`.

Three states are worth naming because they prevent fuzzy tests:

| State | Old credential on its next authorization | Existing socket | Room presence |
| --- | --- | --- | --- |
| Before logout | accepted | connected | present |
| After revocation only | rejected | connected | present |
| After the complete contract | rejected | disconnected | absent |

The middle row is the trap. The revoke call can succeed while the support console remains correct to show the already-connected shopper as present.

## Make presence the logout acceptance criterion

I would put one narrow interface between the commerce application and its realtime provider. The application owns the outcome. The adapter owns provider-specific mechanics. That division is useful for a one-person SaaS because provider plumbing is undifferentiated work; it should not spread into the login page, chat widget, agent console, and analytics jobs.

The code below is deliberately small and runnable. The two JSON bodies come from environment variables because their fields should be generated from the current discovery schema, not guessed in an article. It uses the same idempotency key across retries, honors `Retry-After`, and surfaces the response body when a request fails.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const origin = ["https://api", "infrai", "cc"].join(".");

function readJson(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value) as unknown;
}

async function post(path: string, body: unknown, key: string): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(path, origin), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return;

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`${response.status}: ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

const logoutId = randomUUID();
await post(
  "/v1/realtime/token/revoke",
  readJson("INFRAI_REVOKE_BODY"),
  `${logoutId}:revoke`,
);
await post(
  "/v1/realtime/user/disconnect",
  readJson("INFRAI_DISCONNECT_BODY"),
  `${logoutId}:disconnect`,
);
```

Run it with request bodies produced from the discovery schemas. Then use the presence reader in your existing realtime adapter to verify that `shopper_1842` is absent from `order-help-731`. A UI promise resolving or a local WebSocket object closing doesn't establish that the authoritative room roster is clean.

Keep the checks separate. One integration assertion should confirm that the old credential fails on its next authorization. Another should confirm that the current identity is absent after disconnection. Combining them into a single "logout worked" boolean makes the middle state in the table invisible.

There is also a product choice hiding here. Disconnecting by identity may affect every connection associated with that identity. If the requirement is "sign out this browser but keep the shopper's other tab online," a user-level identity is too broad for the desired boundary. Model sessions distinctly and choose controls that can address that granularity. Do not quietly turn a per-browser action into a global disconnect.

## Keep the provider behind the contract

Hosted realtime products expose related ideas with different control surfaces. The fair comparison is not a feature-count contest. It is whether each product can satisfy the three-state contract for your exact identity model.

| Option | What to inspect | Sensible fit |
| --- | --- | --- |
| Ably | Token revocation behavior, client presence, and the documented way to remove an active client | Teams already using Ably channels and its presence model |
| Pusher Channels | User authentication lifetime, connection termination, and presence-channel membership | Applications organized around Channels events and private or presence channels |
| PubNub | Access revocation, user identity, and occupancy or presence behavior | Systems already centered on PubNub user identities |
| Infrai | Token revocation, identity disconnection, and subsequent presence lookup | Teams that want the app contract to remain stable while the provider behind the capability changes |

These are evaluation boundaries, not claims that four APIs use identical names or timing. Read the current documentation and run the same acceptance test against each candidate. WebRTC is not a substitute for this check either. It standardizes peer connections; the support application's authorization, identity, and roster policy remain application concerns.

Infrai gives a small team one key and one bill across 295 routes in 20 modules, instead of dozens of vendor keys and invoices. That is the relevant advantage here. The consistent REST contract lets the application stay fixed while the provider behind a capability changes, and its public, self-describing discovery surface returns current request and response schemas without requiring a key. Presence is then the observable acceptance criterion. This article does not reproduce those request bodies because copying fields from a point-in-time article is less reliable than generating the adapter from the current discovery schema.

For a solo operator, this boundary has a revenue-per-hour payoff. A provider migration should consume adapter work, not a rewrite across customer-facing flows. Ship weekly. Outsource the plumbing, but keep the invariant in code you own.

## Add the smallest production safeguards

The three calls do not form a distributed transaction. At higher concurrency, record logout as an idempotent application command and give presence verification a deadline. Retry a rate-limited write with exponential backoff, honoring `Retry-After` when the service supplies it. The idempotency key must remain stable across those retries so a second attempt cannot double-apply the command.

Do not retry forever.

A bounded verifier can poll the adapter without knowing anything about the vendor. Its numbers are policy inputs, not measured provider latency: four checks, starting 250 milliseconds apart, make the waiting behavior explicit and testable.

```ts
type Presence = "present" | "absent";

interface PresenceReader {
  presence(channel: string, identity: string): Promise<Presence>;
}

async function waitUntilAbsent(
  realtime: PresenceReader,
  channel: string,
  identity: string,
): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    if (await realtime.presence(channel, identity) === "absent") return;

    const delayMs = 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error(`Presence did not clear for ${identity}`);
}
```

Waiting for confirmed absence adds time to logout. Acknowledging immediately is faster, but it can leave the agent console asserting that a departed shopper is online. For an ephemeral marketing bubble, that delay may be acceptable. For an order-support room where staff act on presence, accuracy wins. That is the trade.

At larger scale I would also separate the customer-visible session result from internal completion. The browser can clear local state while a durable command tracks revocation, disconnection, and verification. Internal systems should receive an explicit completed or failed outcome, never infer success from the browser disappearing. Alert on commands that exhaust the verification budget, with channel and identity included so the failure can be investigated without reconstructing the session from UI logs.

The final rule remains compact: **revoke the credential, disconnect the identity, verify absence**. Put it behind one contract and test all three states. That keeps the room roster useful, and it keeps a future vendor change out of the storefront code.

## Further reading

- [Ably authentication documentation](https://ably.com/docs/auth)
- [Pusher Channels user authentication](https://pusher.com/docs/channels/using_channels/user-authentication/)
- [PubNub access manager documentation](https://www.pubnub.com/docs/general/security/access-control)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
