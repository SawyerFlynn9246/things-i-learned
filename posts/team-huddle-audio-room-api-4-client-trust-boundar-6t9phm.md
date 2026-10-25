# Team Huddle Audio Room API: 4 Client-Trust Boundaries for Property Polls

**TL;DR:** For a property operations huddle with audio and a live poll, use a server-authored session record, short-lived capability tokens, and an outsourced media plane. Keep poll state and authorization in your application. The browser may request actions; it never gets to declare its room, role, or permissions.

| Choice | Ship-weekly fit | Trust boundary | Operational load |
|---|---|---|---|
| Managed media plane | Strong when audio is undifferentiated work | Application still issues narrow capabilities | Lowest media burden |
| Self-hosted forwarding server | Useful when media control is differentiating | Application plus media infrastructure | Highest |
| Browser peer mesh | Acceptable for a deliberately tiny, fixed room | More client coordination | Low server load, more edge cases |

**Recommendation:** choose the first row for a one-person SaaS unless control of media routing is part of the product. It protects revenue-per-hour: ship the tenant and poll rules yourself, outsource the undifferentiated transport, and preserve a boundary that can be replaced later.

## How should a team huddle audio room API trust the client?

Almost none of the authority-bearing state. A building manager can click "open huddle," but the server decides which property the session belongs to, who may join, and whether that person can speak or submit a vote. The client can own device selection, mute state, and presentation. Those are local preferences, not authorization facts.

Keep that line hard.

This distinction matters in a property management product because the same user may manage Building A while only observing Building B. A client-supplied `propertyId`, `role`, or `canPublish` field turns a UI convenience into an escalation path. Derive them from the authenticated account, membership records, and the active session instead.

Treat the session as one application aggregate with four boundaries:

1. Tenant boundary: one property cannot address another property's session.
2. Participation boundary: membership is checked when a token is minted, not accepted from browser state.
3. Capability boundary: audio publication and poll submission are separate grants.
4. Time boundary: access expires and must be re-evaluated against current server state.

Four boundaries are enough. Do not build a general policy language for a weekly feature unless the product already needs one.

## Scope the token to one job

A huddle token should be narrow enough that theft has a bounded consequence. Put the session identifier, participant identifier, capabilities, and expiration in signed claims. Do not put a reusable account credential in a WebSocket query string or hand a browser a secret that can mint new claims.

For example, an example policy can issue a 90-second token and refresh it only while the server still sees valid property membership and an open session. Ninety seconds is a design choice, not a universal constant. Pick a window longer than ordinary reconnects but short enough for removal from a property to take effect promptly. Measure refresh failures before changing it.

The audio and poll paths should consume the same identity but different capabilities. `publish_audio` lets the media admission layer accept an audio publisher. `submit_poll` lets the application accept one vote for the active question. A user who only listens needs neither write capability.

Consider the ordinary Monday handoff. A regional manager opens a huddle for one property, six staff members join, and the manager asks which maintenance request should go first. The audio layer carries discussion while the application opens one poll. A staff member transferred away from that property should fail at token refresh even if the tab remains open; an observer may hear the discussion but cannot publish audio or vote; a repeated poll submission after a reconnect resolves to the existing response rather than creating another one. One session identifier connects these checks, but it does not collapse them into one broad permission. This concrete split is more useful than a generic "room member" role because each write can be denied independently and logged with a reason.

WebRTC supplies the browser media primitives, including `RTCPeerConnection`; it does not define the application's signaling protocol. That leaves room for a small application-owned admission exchange rather than coupling business membership to whatever signaling transport happens to be in use.

## A small Node.js authorization core

The useful abstraction is not "create any room." It is "mint current capabilities for this authenticated member of this open property session." The following TypeScript keeps that decision in one function and rejects client-authored scope. The media adapter and poll handler can both verify the resulting claims.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

type Capability = "join_audio" | "publish_audio" | "submit_poll";

type Session = {
  id: string;
  propertyId: string;
  status: "open" | "closed";
  pollOpen: boolean;
};

type Membership = {
  userId: string;
  propertyId: string;
  canSpeak: boolean;
  canVote: boolean;
};

type Claims = {
  sessionId: string;
  participantId: string;
  capabilities: Capability[];
  exp: number;
};

function encode(value: string): string {
  return Buffer.from(value, "utf8").toString("base64url");
}

function signClaims(claims: Claims, secret: string): string {
  const payload = encode(JSON.stringify(claims));
  const signature = createHmac("sha256", secret)
    .update(payload)
    .digest("base64url");
  return `${payload}.${signature}`;
}

function verifyClaims(token: string, secret: string, nowSeconds: number): Claims {
  const [payload, supplied] = token.split(".");
  if (!payload || !supplied) throw new Error("invalid token");

  const expected = createHmac("sha256", secret).update(payload).digest();
  const actual = Buffer.from(supplied, "base64url");
  if (actual.length !== expected.length || !timingSafeEqual(actual, expected)) {
    throw new Error("invalid token");
  }

  const claims = JSON.parse(
    Buffer.from(payload, "base64url").toString("utf8")
  ) as Claims;
  if (!Number.isInteger(claims.exp) || claims.exp <= nowSeconds) {
    throw new Error("expired token");
  }
  return claims;
}

function issueSessionToken(
  session: Session,
  membership: Membership,
  authenticatedUserId: string,
  secret: string,
  nowSeconds: number
): string {
  if (session.status !== "open") throw new Error("session closed");
  if (membership.userId !== authenticatedUserId) throw new Error("wrong member");
  if (membership.propertyId !== session.propertyId) throw new Error("wrong property");

  const capabilities: Capability[] = ["join_audio"];
  if (membership.canSpeak) capabilities.push("publish_audio");
  if (membership.canVote && session.pollOpen) capabilities.push("submit_poll");

  return signClaims(
    {
      sessionId: session.id,
      participantId: authenticatedUserId,
      capabilities,
      exp: nowSeconds + 90
    },
    secret
  );
}
```

`verifyClaims` proves integrity and expiry, but each consumer must still check the required capability and exact `sessionId`. The poll write also needs an application-level uniqueness rule such as one response per participant and poll. That rule belongs in durable storage; a reconnect must not turn the same click into a second vote.

Keep lifecycle transitions equally plain: `scheduled` is unnecessary for an ad hoc huddle. Create `open`, allow joins, close explicitly when the host ends the session, and run a server-side cleanup check when the last participant leaves. A short grace period can absorb a network handoff, but the server remains the clock and source of truth.

## When is the runner-up the better choice?

Self-host the media forwarding layer when media behavior itself creates product value or a hard deployment constraint demands control over where it runs. That choice is defensible, but price is not the main test. Count the engineering hours spent on upgrades, capacity, packet-level diagnosis, deployment, and on-call work. For a solo operator, those hours compete directly with the next weekly release.

The managed option has a real limitation: transport behavior and deployment placement are constrained by the chosen service boundary. This approach is not suitable when custom packet processing, a mandatory private deployment, or direct control of media routing is a product requirement. Self-host the forwarding layer instead, and accept its operational load as part of the feature.

A browser peer mesh is the runner-up for a tightly capped internal huddle where every participant sends media directly to every other participant. The application still needs server-side membership, signaling, token expiry, poll ordering, and cleanup. Removing a media server does not remove the trust problem. It changes the network topology.

That is the trade-off.

Test the boundaries before testing polish. Attempt to reuse a token in another session, submit a vote after poll closure, publish audio with a listen-only token, refresh after membership removal, and reconnect after session closure. Log denials by reason without recording token contents. Track open sessions, participants, token refresh outcomes, and time from the last departure to closure; these signals expose lifecycle drift without collecting audio.

This is the decision rule: **own tenant authorization and shared application state; rent media transport until controlling it earns its maintenance cost.** It keeps client trust narrow, gives the live poll a durable source of truth, and leaves the audio implementation replaceable.

## Further reading

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
