# Octopus Deploy OpenFeature provider specification

Behavioural specifications for all Octopus Deploy OpenFeature providers, and shared test data so we can ensure consistent evaluation across all providers.

Each fixture in `Fixtures/*.json` holds one simulated `GET api/feature-flags/evaluations/v4` response and the cases evaluated against it. `schema/fixtures.schema.json` describes the format. The evaluation rules themselves are not restated here — the fixtures are the specification.

## Deliberately invalid fixtures

`malformed-evaluations.json` and `unrecognised-conditions.json` describe responses no server should send, so they do not validate against the schema. Neither does the `"region": null` case in `evaluate-for-multiple-context-attributes.json`, which pins how a null context value is treated.
