# H062: Offer `pricen` / `priced` names looked malformed, but they are the established export contract

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Low
**Impact**: schema naming
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the offer exporter had silently drifted from the intended schema, the rational price columns would likely appear as `price_n` and `price_d`, matching the more descriptive naming used in other tables such as trades.

## Mechanism

`OfferOutput` and `OfferOutputParquet` name the numerator and denominator columns `pricen` and `priced`, which initially looked like a typo or broken schema migration. That would be a structural data bug if downstream consumers were supposed to receive `price_n` / `price_d` instead.

## Trigger

Inspect any exported offer row or the offer Parquet schema. Both formats expose `pricen` and `priced`.

## Target Code

- `internal/transform/schema.go:260-275` — JSON schema defines `PriceN`/`PriceD` as `json:"pricen"` / `json:"priced"`
- `internal/transform/schema_parquet.go:193-208` — Parquet schema uses the same `pricen` / `priced` names
- `testdata/offers/bucket_read_offset.golden:1-2` — committed offer fixtures already assert `pricen` / `priced` in emitted rows

## Evidence

The names differ from the trade schema's `price_n` / `price_d`, so the offer table initially looks inconsistent with its nearest sibling. That kind of cross-table mismatch is often a sign of schema drift.

## Anti-Evidence

The JSON schema, Parquet schema, and committed golden fixtures are all internally consistent on `pricen` / `priced`. I found no local code or tests expecting `price_n` / `price_d` for offers, so there is no evidence of current data being exported under the wrong column names.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is a long-standing schema contract, not a silent export regression. The unusual names may be awkward, but the offer exporter, Parquet schema, and golden fixtures all agree on them, so downstream consumers are not seeing internally contradictory data.

### Lesson Learned

Cross-table naming inconsistency alone is not enough for a data-integrity finding. To make a schema-name hypothesis viable, you need proof that the same table's own code, fixtures, or documented contract disagree about the expected column name.
