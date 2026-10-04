# Fintech Media Contracts — White-Label Image Delivery, Presets, Watermarks, Formats

Short answer: for a fintech brand portal, generate the small set of approved image variants at upload time, then keep on-demand transforms for exceptional crops. That choice makes the contract testable: every asset has known presets, a watermark policy, and a format fallback before a partner ever sees it.

The short version is simple. Upload-time processing wins when the portal has a finite set of placements and strict review rules. On-demand processing wins when placements change often or users can art-direct a crop. Most solo teams should use a hybrid: produce canonical derivatives once, and allow a tightly bounded dynamic path for new aspect ratios.

## The decision matrix I use before touching code

| Constraint | Process at upload | Process on demand |
| --- | --- | --- |
| Placements | A known catalog such as 1:1, 4:3, and 16:9 | A growing or user-defined catalog |
| Review | Easy to inspect and approve every derivative | Review happens at request time |
| Latency | First view reads a ready object | First view pays transform latency |
| Storage | More objects and lifecycle rules | Fewer stored derivatives, more compute |
| Crop changes | Requires a re-run job | New policy applies to the next request |

I optimize for revenue per hour, not for the smallest storage bill. A failed hero image in a partner pitch costs more time than a few extra objects in a bucket. I don't need another bespoke image tool to maintain. Ship weekly. Outsource the undifferentiated work to a queue and a repeatable policy.

Keep it finite.

The catch is that upload-time processing is not suitable when editors invent new placements every day. In that case, keep the original immutable and use on-demand transforms behind a cache. Stick with a pure dynamic service when experimentation is the product; use the hybrid when compliance and repeatable branding are the product.

## What should presets, watermarks, and formats mean in a white-label contract?

Treat a preset as a versioned contract, not a bag of query parameters. A preset should name its target dimensions, aspect-ratio policy, focal-point behavior, watermark rule, and output negotiation. `card-v2` is more useful than `width=640&height=480` because a policy can be reviewed, rolled back, and referenced in an audit.

For smart crops, define what happens when the requested ratio differs from the source. A face or a logo may be a focal point, but the system still needs a deterministic fallback for ordinary product photography. Center crop is a reasonable default; it is not a promise that every image will look good. Store the crop metadata with the derivative so a reviewer can reproduce the result.

Watermarks need the same discipline. Record whether a mark is required, where it is anchored, its opacity range, and whether it is baked into an export or applied only to a preview. Never let a browser toggle turn a required mark off. A signed transformation request can select a preset, while authorization decides which preset names a partner may use.

Formats are content negotiation, not a popularity contest. The `Accept` header tells a server what a client can decode; the server should still provide a stable fallback when the header is absent or overly broad. MDN's media format guide is a useful reference for browser support and trade-offs. Keep the original in a lossless or source format, then choose a web delivery format per derivative and document the fallback.

## How do image delivery, presets, watermarks, and formats fit a fintech portal?

Start with an asset record that is independent of any partner's URL. It should include an opaque asset id, source checksum, policy version, and derivative status. Partner-facing URLs can then be revoked or rotated without changing the source record. This separation matters when a bank changes its brand kit while an older statement still needs to render exactly as approved.

Here is the shape of a small TypeScript policy function. It does not depend on a vendor SDK, and it keeps the decision in code that can be unit-tested.

```ts
type OutputFormat = "avif" | "webp" | "jpeg";

type ImageRequest = {
  preset: "square" | "landscape" | "portrait";
  accepts: readonly string[];
  preview: boolean;
};

type TransformPlan = {
  width: number;
  height: number;
  format: OutputFormat;
  watermark: "required" | "optional";
};

const sizes = {
  square: [800, 800],
  landscape: [1200, 675],
  portrait: [675, 900],
} as const;

function planTransform(request: ImageRequest): TransformPlan {
  const [width, height] = sizes[request.preset];
  const format: OutputFormat = request.accepts.includes("image/avif")
    ? "avif"
    : request.accepts.includes("image/webp")
      ? "webp"
      : "jpeg";

  return {
    width,
    height,
    format,
    watermark: request.preview ? "required" : "optional",
  };
}
```

The important part is the boundary around this function. A queue worker can run the plan for each approved preset after upload. An HTTP handler can run it for a cache miss when a new placement is explicitly allowed. Both paths use the same policy version, so a partner cannot receive one watermark rule from the upload path and another from the dynamic path.

I also keep a manifest beside each derivative: source checksum, preset id, policy version, output format, dimensions, and created timestamp. A manifest turns “why is this image different?” into a lookup instead of a forensic exercise. It gives support a useful answer without exposing internal storage layout to the partner.

## Where this pipeline fails in production

The first failure is a race. The portal publishes an asset record before every required derivative is ready, and a partner gets a 404 for the landscape card. Model derivative readiness explicitly. The publish transition should require the mandatory set, while optional formats can be marked pending. In practice, that means the upload transaction writes an `asset_id` and a `pending` manifest, then a worker claims each preset with an idempotency key. If the worker is retried after a timeout, it can compare the source checksum and policy version before writing again. Only the final state change exposes the asset to a partner. This extra state feels fussy on a Friday afternoon, but it prevents a support ticket on Monday when a campaign link is already in a bank review deck.

The second failure is cache identity. If the URL omits the preset version, watermark state, or negotiated format, a cached response can outlive the policy that produced it. Put those inputs into a stable cache key, or make the derivative id include a content hash. Invalidate by version, not by hoping every edge cache forgets at the same time.

That key is part of the public contract.

The third failure is unbounded input. A caller can request a 20,000-pixel image or hundreds of ratios and turn a helpful dynamic path into an expensive job farm. Bound dimensions, preset names, and concurrency. Return a clear validation response before any decoder or compositor starts work.

There is a quieter failure: the crop is technically valid but visually wrong. Automated checks should catch dimensions, MIME type, byte limits, and watermark presence. A small human review set should catch faces cut in half, unreadable marks, and text rendered too close to an edge. I am not sure a single saliency algorithm can cover every brand style; your mileage may vary, so keep a manual override for the rare high-value asset.

## A weekly operating loop for a one-person SaaS

I keep the operating loop boring. On Monday, review which presets were actually requested and retire names nobody uses. During the week, sample a few manifests and compare source checksums with derivative metadata. Before a release, run a matrix of `Accept` headers against the same asset and assert that the fallback is deterministic.

Use metrics that point to a decision: queue age for upload work, cache-hit rate for dynamic work, rejection counts by validation rule, and bytes delivered by format. Alert on a missing required derivative, not on every transient retry. Keep the original and manifests under a lifecycle policy that matches retention obligations; derivatives can usually be rebuilt, while an approved source may need a longer audit trail.

The runner-up approach is better in two cases. If a portal is a creative sandbox with unknown placements, on-demand transforms preserve momentum and avoid precomputing a combinatorial set. If legal requires every public pixel to be reviewed before release, upload-time derivatives are the safer default even when they consume more storage. The decision is about control and change rate, not about a universal “best” image service.

A compact rule closes the loop: finite placements plus strict approval means upload-time; volatile placements plus rapid experimentation means on-demand; both together means versioned upload presets with a bounded dynamic escape hatch.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation
- https://www.rfc-editor.org/rfc/rfc9110.html
