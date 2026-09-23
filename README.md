# Octopus Deploy OpenFeature provider specification

Behavioural specifications for all Octopus Deploy OpenFeature providers, and shared test data so we can ensure consistent evaluation across all providers.

Each fixture in `Fixtures/*.json` holds one simulated `GET api/feature-flags/evaluations/v4` response and the cases evaluated against it. `schema/fixtures.schema.json` describes the format.

## Fixture layout

The filename prefix says what a fixture covers. Put a new case in the file that matches the behaviour you are testing.

`response-*` — reading the payload, before any rule is evaluated.

- `response-flag-lookup.json` — finding a flag by slug, and the default value when there is none
- `response-unknown-fields.json` — extra fields a newer server might send
- `response-unknown-condition-types.json` — condition types this library does not recognise
- `response-malformed-flags.json` — responses no server should send

`condition-*` — one condition type on its own, one file per condition class in the providers.

- `condition-percentage-by-context.json` — bucketing a targeting key
- `condition-context-attribute-is-one-of.json` — matching an attribute
- `condition-context-attribute-is-not-one-of.json` — excluding an attribute

`evaluation-*` — conditions and rules combined.

- `evaluation-conditions-within-a-rule.json` — a rule needs all of its conditions
- `evaluation-rules-within-a-flag.json` — a flag needs any one of its rules

## Deliberately invalid fixtures

`response-malformed-flags.json` and `response-unknown-condition-types.json` describe responses no server should send, so they do not validate against the schema. Neither does the `"region": null` case in `evaluation-conditions-within-a-rule.json`.
