# Telemetry Measurability

Use this reference when the user wants counts, percentages, trends, or distinct-entity conclusions from metrics or other retained telemetry.

## Event Counts vs Distinct Entities

- A metric can support event counts even when it cannot support distinct-customer counts.
- Do not infer distinct customers, orders, users, or entities unless the telemetry retains a trustworthy identity dimension for that exact concept.
- If the metric dimensions do not retain `customerId`, `orderId`, `userId`, or another unique identifier, state explicitly that the metric supports event counts, not distinct-entity counts.

## Retained Dimension Check

Before using a dimension in analysis:

1. verify that the dimension is retained on the telemetry source
2. verify that the dimension values are populated in the time window being analyzed
3. confirm that the intended filter actually narrows the data as expected

If a dimensioned breakdown does not sum to the full total:

- do not hide the mismatch
- call out likely missing-dimension coverage, partial tagging, or telemetry-shape changes over time

## Fixed Windows vs One Long Scalar

For historical reporting:

- prefer fixed calendar months for monthly reporting
- prefer fixed days for daily reporting
- prefer the sum of fixed windows over one giant scalar rollup when they disagree materially

If fixed-window sums and one-shot long-range scalars differ:

- state that fixed windows are being treated as the source of truth
- state the likely reason if known, such as rollup instability, retention behavior, or partial dimension coverage

## Environment Discipline

- Default to production unless the user asks for another environment.
- Confirm that the production filter actually exists on the telemetry surface.
- Prefer exact environment filters over name fragments or fuzzy matching.

## Measurability Warnings To Reuse

Use language like this when needed:

- `This telemetry supports event counts, not distinct-customer counts.`
- `This dimensioned breakdown does not cover the full raw total, which suggests partial tag coverage or telemetry-shape drift.`
- `Fixed-window sums are being treated as the source of truth because one-shot long-range rollups were not stable enough to trust.`
