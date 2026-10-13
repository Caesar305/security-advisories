# Bypassing route-protection middleware in @clerk/nextjs via path-normalization desync

**Author:** Caesar

## Research context

This case study describes a vulnerability discovered by the author during an authorized security assessment of a consented production deployment running a vulnerable release of `@clerk/nextjs`, Clerk's route-protection middleware for Next.js applications. The customer deployment, its hostname, the engagement identifiers, and the client name are intentionally withheld.

The vulnerability was reported privately to Clerk, and Clerk shipped a fix in `@clerk/nextjs` 6.39.2 (matcher normalization lands in `@clerk/shared` 3.47.4). Dates and internal references are withheld to limit identification of the affected system.

This is a sanitized first-party account. It does not claim independent validation by an external security program. Supporting evidence is not included in this publication.

## Summary

`@clerk/nextjs` middleware protects routes by matching the raw request path against developer-configured matchers, while the Next.js router resolves the same request after its own path normalization. Requests crafted so that the two views of the path disagree reach protected route handlers without the middleware having applied its auth protection.

Two request-shaping classes were reproduced against a live unpatched deployment: percent-encoding an unreserved character inside a protected path prefix, and case-varying that prefix. On the tested deployment, middleware-gated routes answered with the middleware's session-protection response stamp absent, for both classes.

## Root cause

Two components normalize the same path differently:

- The route matcher (in `@clerk/shared` before 3.47.4) tested matchers against the **raw** request path: no percent-decoding of unreserved characters, no collapsing of consecutive slashes, case-sensitive comparison.
- The Next.js router and the serving edge layer decode percent-encoded characters, tolerate case differences in path segments, and collapse duplicate slashes when resolving a route.

A path whose encoded or cased form fails every configured matcher, while its normalized form resolves to a protected route, therefore skips the middleware entirely. The vendor's fix aligns the two views by normalizing the matcher's input (`decodeURI` of unreserved characters, collapsing of consecutive slashes, rejection of malformed percent-encoding). Byte-level comparison of the 6.39.0 and 6.39.1 distributions showed identical compiled output, so both releases behave the same under the vectors.

The primary weakness is [CWE-436: Interpretation Conflict](https://cwe.mitre.org/data/definitions/436.html) between the security-control input and the routing input; the consequence is [CWE-863: Incorrect Authorization](https://cwe.mitre.org/data/definitions/863.html) for applications that treat the middleware as the sole enforcement point.

## Proof-of-concept form

Reproduced vectors operate on the *protected prefix* the application's matcher config names (typically an `api`-style first segment):

1. **Encoded unreserved character.** Percent-encode one unreserved character inside the protected prefix (any character that `decodeURI` will restore). The matcher sees the raw encoded form, finds no match, and skips protection; the router decodes and resolves the protected route.
2. **Case variant.** Rewrite the prefix with mixed case. The matcher's comparison is case-sensitive; the path resolution tolerates it. Same desync.
3. Ruled out on the tested topology: an extra leading slash collapses at the edge layer *before* the middleware runs, so that variant never reached the desync window. Vector viability is deployment-topology dependent.

The detection signal used was the middleware's protection stamp: gated responses on the unpatched release carry the Clerk auth-status headers when the middleware fires, and requests through the crafted path returned without them.

Runnable request strings as used against the consented target are deliberately withheld, along with hosts and endpoint paths; the mechanism above is sufficient to reproduce the class locally against an unpatched release.

## Impact and boundaries

Confirmed on the tested deployment: protected API routes and protected page routes answered with the middleware's protection stamp absent, for the reproduced vector classes. The middleware's protection enforcement was bypassed. Not confirmed, and not claimed: unauthorized disclosure of customer or production data. Every handler reached still performed its own authorization, and application surfaces configured with a fail-closed matcher pattern (protect everything, allow-list the public routes) resisted all vectors tested.

The practical severity depends on enforcement depth. Middleware-only protection means real exposure of otherwise-authenticated routes; the bypass is not, by itself, a data breach, and defense-in-depth held where handlers independently checked sessions.

No evidence of historical exploitation by others was established or looked for beyond the engagement scope; this is not a claim that none occurred.

## Remediation

Upgrade `@clerk/nextjs` to 6.39.2 or later (matcher normalization ships in `@clerk/shared` 3.47.4). The upgrade is a drop-in for the affected application patterns. Guidance published with the disclosure also recommended not stopping at the middleware: keep session checks inside route handlers, because the bypass class exists wherever two components disagree about path equivalence, and prefer fail-closed matcher patterns so a missed matcher protects rather than exposes.

## Validation

Post-upgrade re-testing on the consented deployment showed the encoded and case-variant vectors matching their protected routes again and receiving the middleware's protection behavior, and malformed percent-encoding rejected. Exact requests, responses, and retest logs remain in the engagement's private evidence set, consistent with this repository's limits.

These results address the specific issue described here. They are not a comprehensive assessment of the framework or of the tested deployment.

## Evidence and disclosure limits

Live hosts, endpoint paths, exact payload strings, engagement dates, and all request/response evidence are withheld. Anonymization limits external verifiability. Any reviewer requiring identifying evidence would need a separately agreed private review channel and authorization for the specific materials shared.
