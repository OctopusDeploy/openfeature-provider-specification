# Octopus Feature Toggles OpenFeature Provider Specification

This specification defines the requirements for Octopus OpenFeature provider implementations. It sits on top of the [OpenFeature provider specification](https://openfeature.dev/specification/sections/providers/), providing complete requirements to implement an OpenFeature-compliant provider for Octopus Feature Toggles.

All requirements are mandatory. There is no SHOULD/MAY tiering; a conforming provider implements everything.

Requirements are numbered as `REQ-{section}.{index}` and are referenced in tests, PRs, and issues using that identifier.

## Deprecation

When a requirement is no longer applicable it is struck through and annotated inline with the version it was deprecated and the reason. Requirement numbers are never reused.

Example:

> ### ~~REQ-7.1: No telemetry~~ *(deprecated v1.2 — usage telemetry is now supported)*
> ~~Providers must not send evaluation data to any external endpoint.~~

---

## 1. Configuration

### REQ-1.1: Client identifier
The provider must accept a `clientIdentifier` string as a required configuration parameter. The client identifier is a JWT that encodes the Octopus project, environment, and optionally a tenant. Evaluation is scoped to these values.

Client identifiers are not validated.

### REQ-1.2: Product metadata
The provider must accept a `productMetadata` value as a required configuration parameter. The product metadata must contain a required string for `name` with the name of the product consuming the feature toggles, and an optional `version` string with the version of the project.

Both values must be sanitized to produce a valid RFC 9110 token. Any character that is not a valid token character must be removed.

If `name` is null, empty, or reduces to an empty string after sanitization, the provider must throw a configuration error. If `version` is supplied and reduces to an empty string after sanitization, the provider must throw a language-appropriate error.

### REQ-1.3: Cache duration
The provider must accept an optional `cacheDuration` configuration parameter. When not supplied, the cache duration must default to `60` seconds.

### REQ-1.4: API base URL
The provider must accept an optional API base URL configuration parameter to allow targeting non-public environments. When not supplied, the provider must use the production Octopus feature toggle value `https://features.octopus.com`.

### REQ-1.5: Release version override
The provider must accept an optional `releaseVersionOverride` configuration parameter. This is sent to the feature toggle service when fetching feature toggle evaluations to override the version specified in the client identifier.

## 2. Initialisation & Provider Lifecycle

### REQ-2.1: Initial flag fetch
On initialisation the provider must fetch flag configuration from the Octopus feature toggle API before transitioning to the READY state.

### REQ-2.2: Initial fetch retry
If the initial fetch fails, the provider must retry up to 3 times with a 5-second backoff between attempts.

### REQ-2.3: Startup failure fallback
If all startup attempts are exhausted without a successful response, the provider must transition to the ERROR state with an empty flag configuration and begin the background polling loop. The provider must continue polling at the configured interval and transition to READY if a subsequent poll succeeds.

### REQ-2.4: Background polling
After a successful initialisation, the provider must poll the Octopus feature toggle API at the configured cache duration interval to refresh flag configuration.

### REQ-2.5: Failed poll handling
If a background poll fails, the provider must transition to the STALE state immediately and continue serving the last successfully fetched flag configuration.

The last successfully fetched flag configuration must never be discarded.

### REQ-2.6: Poll retry behaviour
The background polling loop must not apply any additional retry behaviour between intervals. A failed poll is attempted again at the next interval.

## 3. Flag Evaluation

### REQ-3.1: Boolean-only evaluation
The provider must support boolean flag evaluation only. All other flag types defined by the OpenFeature SDK (`string`, `integer`, `float`, `object`) are not supported by Octopus feature toggles.

### REQ-3.2: Missing flag evaluation
If the flag slug cannot be found, return the caller's default value with the error code `FLAG_NOT_FOUND`.

### REQ-3.3: Non-boolean flag evaluation
Evaluating a non-boolean flag type (string, integer, float, object) must return the caller's default value with error code `TYPE_MISMATCH`. No coercion must be attempted.

### REQ-3.4: False flags
When evaluating a boolean flag with a `false` value, the provider must return `false` and not evaluate any rules.

## 4. Segments

### REQ-4.1: Segment structure
A toggle may have zero or more segments configured. Each segment is a key/value pair. Multiple segments with the same key but different values may exist on a single toggle.

### REQ-4.2: Segment matching
To satisfy segment matching, the evaluation context must contain at least one custom field value matching a segment value for every distinct key present in the toggle's segments. Within a key the match is OR (any value matches); across keys the match is AND (all keys must match). Comparisons are case-insensitive string matches.

### REQ-4.3: No segments configured
If a toggle has no segments configured, segment matching is considered satisfied for all evaluation contexts.

### REQ-4.4: Missing context field
If the evaluation context does not contain a custom field for a key referenced by any segment, that key's match fails and the overall segment match fails.

## 5. Fractional Rollout

### REQ-5.1: Deterministic rollout
Fractional rollout must be evaluated deterministically. Given the same flag key, targeting key, and percentage, all provider implementations must produce the same result.

### REQ-5.2: Hash input construction
The hash input must be constructed by concatenating the flag key and the targeting key with a `/` separator: `{flagKey}/{targetingKey}`.

### REQ-5.3: Hash algorithm
The hash algorithm must be SHA-256. The hash must be computed over the UTF-8 encoding of the input string.

### REQ-5.4: Bucket value derivation
The bucket value must be derived by taking the first 4 bytes of the SHA-256 digest, interpreting them as a big-endian unsigned 32-bit integer, and computing `value mod 100000`. This produces a bucket in the range `[0, 99999]`.

### REQ-5.5: Rollout percentage comparison
The configured rollout percentage must be expressed as a value between `0` and `100` inclusive, with up to 3 decimal places of precision. To compare against the bucket value, the percentage must be scaled to the same range: `threshold = percentage * 1000`. If `bucket < threshold`, the evaluation resolves to `true`.

### REQ-5.6: Test vector compliance
The shared test vectors in the specification repository under `test-data/` must all pass. These vectors are the authoritative compliance check for fractional rollout correctness.

### REQ-3.4: Targeting key requirement
The evaluation context must provide a `targetingKey` for any evaluation involving fractional rollout. If the `targetingKey` is absent and the rollout percentage is less than 100%, the provider must return `false`.

## 6. Error Handling

### REQ-6.1: Unexpected evaluation error
Errors must never propagate as exceptions to the caller if they are not handled by the OpenFeature SDK. Unhandled exceptions must not propagate to the caller.

## 7. Telemetry

### REQ-7.1: Client header
All calls to the feature toggle service must send an `X-Octopus-Client` header containing details about the product and library version.

Product name and version must be taken from the product metadata provider in configuration, library name must be the [OpenFeature metadata name](https://openfeature.dev/specification/sections/providers#requirement-211) and library version must be the public version number for this version of the library.

The value must be of the format `{product-name}/{product-version} {library-name}/{library-version}` if a product version was provided or `{product-name} {library-name}/{library-version}` if no product version was provided.

### REQ-7.2: Evaluation telemetry
Telemetry is not currently implemented. Providers must not send evaluation data to any external endpoint.

## 8. Testing

### REQ-8.1: Shared testing fixtures
Providers must implement automated test using the shared test fixtures in this repository. 