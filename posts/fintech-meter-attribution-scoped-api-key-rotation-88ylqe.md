# Fintech Meter Attribution: Scoped API Key Rotation for Least Privilege CI Pipelines

A fintech CI pipeline has an awkward constraint: it needs a scoped API key with least privilege to publish billable tenant artifacts, yet that key can leak into retained build logs if a script becomes noisy. The useful choice is a separate, revocable key for each tenant and pipeline purpose, with the tenant identity bound on the server rather than accepted from the request body.

**TL;DR:** issue a high-entropy opaque token, store only its digest, attach one tenant and one narrow action set to its database record, and write the credential ID into every billing event. Rotation means overlapping old and new credentials briefly, proving the new one works, then revoking the old record. A leaked log becomes one bounded revocation, not an account-wide migration.

## How should a scoped API key enforce least privilege in a CI pipeline?

Least privilege is usually described as a security property. Here it is also an accounting invariant. If one CI secret can submit usage for several tenants, the server must trust a caller-supplied `tenantId`; a typo, copied fixture, or altered request can charge the wrong ledger even when authentication succeeds.

Bind the tenant during issuance. On each authenticated call, derive `tenant_id`, `credential_id`, and allowed actions from the credential record. Ignore any tenant assertion in the payload. The resulting billing event can carry both identifiers, which makes two different questions answerable: which customer owns the event, and which automation credential produced it?

No exceptions.

That separation matters. A tenant may have a release publisher, a reconciliation job, and an emergency repair task. Giving all three the same secret saves a few minutes today but makes later attribution ambiguous. For a one-person SaaS, ambiguous invoices create support work at exactly the wrong time. I would spend the extra table row and ship the narrower boundary.

## The constraint that rules out a shared secret

A CI credential should do one job. For this example, the job is `billing-artifact:publish` for tenant `tn_7F3`, with no read access and no authority to issue another credential. The build can submit the artifact. It cannot inspect balances, cross tenant boundaries, or administer keys.

The log is hostile storage. Even if the CI system masks configured secrets, derived values, transformed output, archived test reports, and third-party steps enlarge the places that must be reviewed. OWASP recommends limiting secret scope, centralizing lifecycle management, auditing use, and rotating exposed credentials. Those controls reinforce one another; masking alone does not establish containment. Consider the full failure chain rather than the first visible mistake: a debug flag prints a request object, the authorization value lands in a retained build log, a retry job downloads the same artifact, and the shared credential continues to authorize several tenants. Redaction might remove the first appearance while missing an encoded copy. Rotation might replace the secret while leaving the old one valid for a queued worker. A narrow record changes that chain. The operator revokes one credential ID, queued calls fail at the common authorization layer, and audit queries identify every accepted billing event produced by that credential before revocation. The tenant binding still comes from the database, so possession of the token cannot redirect a publish to another ledger by changing JSON. This is why I treat log hygiene, server-side scope, attribution, audit, and revocation as one design decision. Dropping any one of them leaves an accounting gap, even if the other controls look tidy in a security checklist.

Keep the secret out of command arguments and generated configuration. Pass it through the step environment, disable shell tracing, and make the HTTP client redact authorization headers before errors reach the logger. Also keep pull-request code from untrusted contexts away from environments that hold production credentials. These are layers. None is permission to print a token.

## Smallest working credential boundary

The implementation below shows the security boundary, not a vendor SDK. It issues a 256-bit opaque secret, persists a SHA-256 digest, and returns the plaintext once. SHA-256 is appropriate here because the input is a randomly generated 32-byte value rather than a human password. The database must enforce uniqueness on `tokenHash` and treat revocation as an atomic update.

```ts
import { createHash, randomBytes, timingSafeEqual } from "node:crypto";

type Action = "billing-artifact:publish";

type Credential = {
  id: string;
  tenantId: string;
  tokenHash: Buffer;
  actions: readonly Action[];
  createdAt: Date;
  revokedAt: Date | null;
};

const digest = (token: string): Buffer =>
  createHash("sha256").update(token, "utf8").digest();

export function issueCredential(
  tenantId: string,
  save: (row: Credential) => void,
): { credentialId: string; token: string } {
  const credentialId = crypto.randomUUID();
  const token = `ci_${randomBytes(32).toString("base64url")}`;

  save({
    id: credentialId,
    tenantId,
    tokenHash: digest(token),
    actions: ["billing-artifact:publish"],
    createdAt: new Date(),
    revokedAt: null,
  });

  return { credentialId, token };
}

export function authenticate(
  presentedToken: string,
  candidates: readonly Credential[],
): Credential | null {
  const presentedHash = digest(presentedToken);
  const row = candidates.find(
    (candidate) =>
      candidate.tokenHash.length === presentedHash.length &&
      timingSafeEqual(candidate.tokenHash, presentedHash),
  );

  if (!row || row.revokedAt) return null;
  return row;
}

export function authorizePublish(row: Credential): {
  tenantId: string;
  credentialId: string;
} {
  if (!row.actions.includes("billing-artifact:publish")) {
    throw new Error("credential lacks billing-artifact:publish");
  }
  return { tenantId: row.tenantId, credentialId: row.id };
}
```

In production, lookup should use an indexed, non-secret credential selector instead of scanning candidates. Put a short public identifier before the secret, fetch that one row, digest the presented full token, then compare fixed-length buffers. Keep the selector non-authoritative: possession of it grants nothing.

The billing write should copy the authenticated values into an append-only event record along with an idempotency key supplied for that publish attempt. Do not overwrite attribution later from mutable tenant metadata. Authentication failures get a stable category in logs, never the presented token.

Test the negative paths first: a valid key with the wrong action, a revoked key, a valid key paired with another tenant in the payload, and two retries carrying the same idempotency key. The third test proves the important rule: request data cannot move usage between tenants.

## Rotation is a two-key state transition

Rotation should not mean editing one secret in place. Issue a second credential with the same tenant and action, add it to the protected CI environment, and run a harmless authenticated check or one idempotent publish. Only after evidence shows the new credential ID in the audit stream should the old record receive `revokedAt`.

Fast rollback remains possible during the overlap because the old credential still exists. Keep that overlap short and explicit; otherwise it becomes permanent privilege duplication. Once revoked, the old token must fail every authorization path, including queued jobs and retry workers. Test that behavior against the same authentication layer used by the live endpoint.

For a suspected build-log leak, skip the leisurely schedule. Revoke the exposed credential, stop or rerun jobs that may still carry it, issue a replacement, and search retained artifacts for the exact secret only through a controlled process that will not echo it again. Then inspect audit events by credential ID and tenant ID. The goal is both containment and a defensible billing trail.

## What I would change at scale

The single-record design is enough to ship weekly while the credential count is modest. At larger scale, I would add a public selector for indexed lookup, a transactional outbox for audit delivery, expiration policy, and automated alerts for use after revocation or from an unexpected pipeline context. I would also separate issuance authority from publish authority so a compromised build cannot mint its successor.

There is a cost: more records, overlap logic, and operational tests. I accept it because the alternative moves complexity into invoice disputes and incident reconstruction, where every hour is more expensive. Outsource commodity secret storage to the CI environment or a dedicated secret manager, but keep tenant binding and billing attribution in the application database. Those are product rules.

The decision rule is compact. **Choose the design that lets one credential be revoked without changing another tenant, and lets every billable event name the server-bound tenant plus the credential that created it.** If a proposed setup cannot do both, it is too broad for this pipeline.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
