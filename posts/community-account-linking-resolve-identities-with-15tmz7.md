# Community Account Linking — Resolve Identities Without Accidental Merges During Recovery

Short answer: link accounts only after the signed-in member proves control of both identities; if that proof is unavailable, recover one account without merging anything.

| Situation | Decision | Why |
|---|---|---|
| Signed in, both identities freshly verified | Offer an explicit link | The member controls both sides |
| Same email, only one identity verified | Keep separate | Email equality is a hint, not proof |
| Recovery request after a stolen session | Revoke sessions, recover, then reconsider linking | Recovery must not become a merge shortcut |
| Conflicting ownership or uncertain history | Escalate or leave separate | A false negative is reversible; a false merge may not be |

For a content community, the recommendation is conservative: model a person, their login identities, and their sessions as different records. Put a uniqueness constraint on each external identity, require a recent authentication ceremony before linking, and record every link or unlink as a security event. This costs a little conversion. It protects posts, moderation history, subscriptions, and property-management conversations from landing in the wrong account.

Don't optimize the prompt first. Optimize the invariant.

## What should a community account linking flow prove before resolving identities?

It should prove control, intent, and exclusivity. Control means the member has recently authenticated to the currently signed-in account and successfully authenticated to the identity being added. Intent means the member clicked a clear “Link account” action rather than merely choosing another sign-in button. Exclusivity means the incoming identity isn't already attached to a different person record.

The data model should make those claims visible. A `member` owns community data. A `login_identity` holds an issuer-scoped subject or another stable credential identifier. A `session` belongs to the member but has its own lifecycle. Email addresses can help discovery and recovery, yet they shouldn't serve as the join key. They can change, be recycled, arrive unverified, or be shared operationally. Even when two providers report the same email, silently merging the corresponding accounts turns a weak correlation into an ownership decision.

That distinction matters in property management. Imagine a manager who posts from a work account, later signs in with a personal identity, and also has access to a shared leasing inbox. The string in an email field cannot tell the system which identity owns the manager's old posts. Nor can it tell whether a shared mailbox represents one person. An automatic merge could expose tenant discussions to the wrong colleague, move moderation actions, and make the recovery screen lie about which account is being restored. The safer result is mundane: show both accounts, ask for proof on both, and leave them separate when proof stops halfway.

Small friction wins here.

## The two criteria that decide the design

The first criterion is the **strength of the linking ceremony**. Linking changes an account's future authentication surface, so treat it like changing a password or recovery factor. OWASP recommends reauthentication for sensitive account changes and after risk events. In practice, expire the linking challenge quickly, bind it to the current member and browser session, require a fresh sign-in for the identity being attached, and consume the challenge once. Don't accept an identity assertion copied from an earlier tab or a bare email claim.

The second criterion is the **quality of the recovery path**. Recovery is where a clean account-linking model gets pressure-tested. A useful design can restore access to one known member record without guessing that two records represent the same human. It also gives support enough evidence to resolve a dispute without granting support agents a magic “merge” button. Useful evidence includes the identity identifiers involved, verification times, the initiating session, the final decision, and who approved an exceptional action. Store the minimum needed for that audit, restrict access, and set a retention rule.

These criteria pull in different directions. A strict two-sided ceremony creates more abandoned links and more support work. A loose email match converts better today but creates an authorization problem that may surface months later. For a small team shipping weekly, support volume is visible; silent ownership corruption isn't. The revenue-per-hour choice is still to outsource undifferentiated authentication mechanics while keeping the linking policy and member-data transaction under application control.

The catch is that strict linking is not suitable when the community has no usable recovery channel for either account. In that case, ship recovery first. Don't compensate with automatic merges.

## One transaction, three outcomes

The application service needs only three outcomes: linked, already linked to this member, or conflict. Keep that decision inside one transaction, where the uniqueness constraint can settle races. The following TypeScript sketch uses generic interfaces on purpose; the policy should survive a database or authentication-provider change.

```ts
type LinkResult =
  | { kind: "linked" }
  | { kind: "already_linked" }
  | { kind: "conflict" };

type VerifiedIdentity = {
  issuer: string;
  subject: string;
  verifiedAt: Date;
};

interface LinkStore {
  transaction<T>(work: (tx: LinkTransaction) => Promise<T>): Promise<T>;
}

interface LinkTransaction {
  findOwner(issuer: string, subject: string): Promise<string | null>;
  attach(memberId: string, identity: VerifiedIdentity): Promise<void>;
  recordSecurityEvent(event: {
    memberId: string;
    action: "identity_linked";
    issuer: string;
    subject: string;
  }): Promise<void>;
}

async function linkVerifiedIdentity(
  store: LinkStore,
  currentMemberId: string,
  identity: VerifiedIdentity,
  now: Date,
): Promise<LinkResult> {
  const verificationAgeMs = now.getTime() - identity.verifiedAt.getTime();
  const fiveMinutesMs = 5 * 60 * 1000;

  if (verificationAgeMs < 0 || verificationAgeMs > fiveMinutesMs) {
    throw new Error("Fresh authentication required");
  }

  return store.transaction(async (tx) => {
    const owner = await tx.findOwner(identity.issuer, identity.subject);

    if (owner === currentMemberId) return { kind: "already_linked" };
    if (owner !== null) return { kind: "conflict" };

    await tx.attach(currentMemberId, identity);
    await tx.recordSecurityEvent({
      memberId: currentMemberId,
      action: "identity_linked",
      issuer: identity.issuer,
      subject: identity.subject,
    });
    return { kind: "linked" };
  });
}
```

The five-minute window is an example policy, not a universal standard. Your risk model and authentication mechanism should determine it. The nonnegotiable part is freshness plus two-sided proof, checked before a single atomic attach. A production implementation should also let the database enforce uniqueness on `(issuer, subject)`; an application-only check can lose a race when two requests arrive together.

Test the ugly paths. Send two link attempts concurrently. Replay a consumed challenge. Try an identity already owned by another member. Change the email while the ceremony is open. Confirm that none of those cases moves posts or expands access. Then test the boring success path and the idempotent retry.

Observe decisions, not secrets — count link attempts, conflicts, expirations, recovery starts, session revocations, and support escalations. Alert on unusual bursts by account or identity, but don't put tokens or raw authentication assertions in logs. A weekly review of conflict and abandonment rates is enough for a one-person operation to spot friction without building a security data warehouse.

## Recovery and stolen sessions must stay separate from merging

Suppose a property manager reports a stolen session. The first job is containment: invalidate the affected session and, when confidence is low, invalidate the member's other sessions too. Rotate refresh credentials as part of issuing the replacement session, require reauthentication before sensitive account changes, and notify the member through an established channel. OWASP's authentication guidance treats reauthentication after suspicious activity and session rotation after reauthentication as core defenses.

Then recover exactly one member record.

Do not use recovery answers, matching profile fields, or access to a shared email inbox as permission to merge another account. Once the member has a trusted session again, the normal two-sided linking ceremony can run. If the second identity cannot be proven, the accounts remain separate and the member can use the normal ownership-dispute process. I'm not sure any automated rule can safely resolve a case where both sides present credible, conflicting ownership evidence; a restricted manual review with a written decision rule is the honest boundary.

The runner-up design — suggesting a possible link when a verified email matches — is better when duplicate accounts are common and the community can tolerate an extra prompt. Keep it as discovery only: “You may have another account” can lead to the two-sided ceremony, but the match itself grants nothing. Stick with fully manual linking when identities often use shared mailboxes, pseudonyms, delegated property-management access, or weak upstream verification. Those environments make even a suggestion sensitive because it may reveal that another account exists.

Account linking is successful when it is boring. The system can explain why an identity belongs to a member, recovery restores one account without changing ownership boundaries, and a stolen session can be revoked without improvising a merge. Ship those invariants before polishing the button copy.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
