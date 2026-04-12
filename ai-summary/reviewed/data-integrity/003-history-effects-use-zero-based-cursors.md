# H003: Effect exports use zero-based `index` / `id` instead of Horizon history order

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_effects.index` and `history_effects.id` should follow the same ordering contract as Horizon/history effects: the first effect for an operation should have order `1`, and the effect identifier/paging token should be derived from that one-based order. A successful operation's first effect should not export as `index = 0` and `id = "<operation_id>-0"`.

## Mechanism

After building the effect slice, `effects()` assigns `EffectIndex = uint32(i)` directly from Go's zero-based slice index and formats `EffectId` from that same value. Upstream Horizon initializes effect order at `1` and formats IDs from the one-based order, so the ETL emits systematically shifted cursor fields for every effect row while still producing plausible-looking data.

## Trigger

1. Export effects for any successful effect-producing operation, such as a payment or trustline-flags update.
2. Inspect the first effect row emitted for an operation.
3. Observe `index = 0` and `id = "<operation_id>-0"` even though Horizon/history order for the first effect starts at `1`.

## Target Code

- `internal/transform/effects.go:161-165` — assigns zero-based `EffectIndex` and formats `EffectId` from it
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/services/horizon/internal/ingest/processors/effects_processor.go:91-95` — upstream processor starts effect order at `1`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/services/horizon/internal/db2/history/effect.go:58-71` — upstream ID / paging-token formatting uses the one-based order
- `testdata/effects/one_ledger_effects.golden:1-8` — checked-in fixtures currently expose the zero-based export (`...-0`, `index:0`)

## Evidence

The repository labels these rows as `history_effects`, but the exported cursor fields do not match Horizon's underlying history model. The mismatch is visible in both code and fixtures: local rows start at zero, whereas upstream ingestion stores first-effect order as `1` and derives IDs from that value.

## Anti-Evidence

Existing tests and fixtures currently lock in the zero-based behavior, so some downstream consumers may already have adapted to it. Even so, the exported `index` / `id` no longer match the history-effect contract suggested by the table naming and upstream source of truth.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The ETL's `EffectOutput` struct is explicitly documented as aligning with the BigQuery table `history_effects` (schema.go:360). Upstream Horizon's `effectsWrapper` initializes its `order` field to `1` (effects_processor.go:94), uses it when inserting each effect (line 210), and increments it afterward (line 216). The upstream `Effect.ID()` and `PagingToken()` methods format IDs using this 1-based order. The ETL's `effects()` function instead assigns `EffectIndex = uint32(i)` from the zero-based Go slice index (effects.go:164), producing a systematic off-by-one in both the `index` and `id` fields for every effect row.

### Code Paths Examined

- `internal/transform/effects.go:161-166` — Post-hoc loop assigns `EffectIndex = uint32(i)` and formats `EffectId = fmt.Sprintf("%d-%d", operationID, i)` using zero-based slice index
- `internal/transform/schema.go:360-371` — `EffectOutput` struct comment states "aligns with the BigQuery table history_effects"; fields are `EffectIndex uint32 json:"index"` and `EffectId string json:"id"`
- Upstream `effects_processor.go:91-95` — `effectsWrapper` initialized with `order: 1`
- Upstream `effects_processor.go:200-217` — `add()` uses `e.order` for DB insertion, then increments `e.order++`
- Upstream `history/effect.go:58-71` — `Effect.ID()` formats as `fmt.Sprintf("%019d-%010d", operationID, Order)` and `PagingToken()` formats as `fmt.Sprintf("%d-%d", operationID, Order)`, both using the 1-based order
- `testdata/effects/one_ledger_effects.golden:1-7` — Golden fixtures confirm zero-based output (`"index":0`, `"id":"...-0"`)
- `internal/transform/effects_test.go:1760-1761,1908-1909` — Tests replicate the zero-based assignment pattern (`EffectIndex = uint32(i)`)

### Findings

The discrepancy is confirmed and systematic:

1. **Upstream Horizon**: First effect per operation gets `order=1`, ID formatted as `<opid>-1`
2. **ETL export**: First effect per operation gets `index=0`, ID formatted as `<opid>-0`
3. **Schema contract**: The `EffectOutput` struct explicitly claims alignment with BigQuery `history_effects`
4. **Impact scope**: Every single effect row exported by the ETL has `index` off by 1 and `id` with a shifted suffix compared to what Horizon would produce

This means:
- Joining ETL effect data with Horizon effect data on the `id` field will produce zero matches
- The `index` field cannot be used as a Horizon-compatible paging cursor
- Downstream systems that assume Horizon-compatible ordering will have silent cursor mismatches

The fix would be to change line 164 from `uint32(i)` to `uint32(i + 1)`, which would also fix the `EffectId` derived from it on line 165. Golden test fixtures and unit test expectations would need updating.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go`
- **Setup**: Use the existing `TransformEffect` test infrastructure (e.g., `TestTransformEffect` or any test case that exercises a payment or trustline-flags operation)
- **Steps**: Call `TransformEffect()` for a transaction with at least one effect-producing operation. Inspect the first `EffectOutput` in the returned slice.
- **Assertion**: Assert that `effects[0].EffectIndex == 0` to demonstrate the zero-based indexing (current behavior). Then show that upstream Horizon would produce `order=1` for the same operation. The PoC should verify the golden fixture `testdata/effects/one_ledger_effects.golden` contains `"index":0` and `"id":"...-0"` entries, confirming the systematic mismatch.
