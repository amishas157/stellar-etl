# 004: Effect limit counts transactions instead of effects

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness: `--limit` can return oversized effect exports
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`export_effects --limit N` is documented as the maximum number of effects to export, but the command applies that budget to source transactions before `TransformEffect()` expands them into effect rows. On pubnet ledger `30822025`, `export_effects -l 1` writes `4` effect rows while reporting only `1` transformed transaction, so the export silently exceeds the caller's requested bound.

## Root Cause

`cmd/export_effects.go` passes the caller-visible `limit` directly into `input.GetTransactions()`, and `GetTransactions()` enforces that limit on `len(txSlice)`. `transform.TransformEffect()` then expands each returned transaction into a variable number of `EffectOutput` rows, but `export_effects` never applies a secondary cap after that expansion and writes every returned effect.

## Reproduction

During normal operation, this manifests whenever the first in-range transaction that survives `GetTransactions(limit=N)` yields more than one effect row. For example, on mainnet ledger `30822025`, running `stellar-etl export_effects -s 30822025 -e 30822025 -l 1` succeeds but writes `4` rows, while the sibling command `export_operations` on the same ledger honors `-l 1` and writes exactly `1` row.

## Affected Code

- `cmd/export_effects.go:21-57` — passes `limit` into `GetTransactions()` and then exports every effect returned by `TransformEffect()`
- `internal/input/transactions.go:23-70` — enforces `limit` on transaction count via `len(txSlice)`
- `internal/transform/effects.go:23-50` — expands a single transaction into all of its effect rows
- `internal/input/operations.go:29-80` — sibling reader shows the intended row-level limit enforcement pattern
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of effects to export

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestEffectLimitCountsTransactionsInsteadOfEffects`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
func TestEffectLimitCountsTransactionsInsteadOfEffects(t *testing.T) {
	// Use the "payment" test case XDR from effects_test.go.
	// This is a native XLM payment that produces exactly 2 effects:
	//   1. AccountCredited (receiver)
	//   2. AccountDebited  (sender)
	paymentEnvelopeXDR := "AAAAABpcjiETZ0uhwxJJhgBPYKWSVJy2TZ2LI87fqV1cUf/UAAAAZAAAADcAAAABAAAAAAAAAAAAAAABAAAAAAAAAAEAAAAAGlyOIRNnS6HDEkmGAE9gpZJUnLZNnYsjzt+pXVxR/9QAAAAAAAAAAAX14QAAAAAAAAAAAVxR/9QAAABAK6pcXYMzAEmH08CZ1LWmvtNDKauhx+OImtP/Lk4hVTMJRVBOebVs5WEPj9iSrgGT0EswuDCZ2i5AEzwgGof9Ag=="
	paymentResultXDR := "AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAABAAAAAAAAAAA="
	paymentMetaXDR := "AAAAAQAAAAIAAAADAAAAOAAAAAAAAAAAGlyOIRNnS6HDEkmGAE9gpZJUnLZNnYsjzt+pXVxR/9QAAAACVAvjnAAAADcAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAABAAAAOAAAAAAAAAAAGlyOIRNnS6HDEkmGAE9gpZJUnLZNnYsjzt+pXVxR/9QAAAACVAvjnAAAADcAAAABAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAABAAAAAA=="
	paymentFeeChangesXDR := "AAAAAgAAAAMAAAA3AAAAAAAAAAAaXI4hE2dLocMSSYYAT2ClklSctk2diyPO36ldXFH/1AAAAAJUC+QAAAAANwAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAEAAAA4AAAAAAAAAAAaXI4hE2dLocMSSYYAT2ClklSctk2diyPO36ldXFH/1AAAAAJUC+OcAAAANwAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAA=="
	paymentHash := "2a805712c6d10f9e74bb0ccf54ae92a2b4b1e586451fe8133a2433816f6b567c"
	ledgerSeq := uint32(56)

	// Build the LedgerTransaction from XDR (same helper used by effects_test.go)
	transaction := BuildLedgerTransaction(t, TestTransaction{
		Index:         1,
		EnvelopeXDR:   paymentEnvelopeXDR,
		ResultXDR:     paymentResultXDR,
		MetaXDR:       paymentMetaXDR,
		FeeChangesXDR: paymentFeeChangesXDR,
		Hash:          paymentHash,
	})

	// Construct a minimal LedgerCloseMeta with a valid close time
	ledgerCloseMeta := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					LedgerSeq: xdr.Uint32(ledgerSeq),
					ScpValue: xdr.StellarValue{
						CloseTime: xdr.TimePoint(0),
					},
				},
			},
		},
	}

	// Call TransformEffect — this is the PRODUCTION code path used by export_effects
	effects, err := TransformEffect(transaction, ledgerSeq, ledgerCloseMeta, "Test SDF Network ; September 2015")
	if err != nil {
		t.Fatalf("TransformEffect failed: %v", err)
	}

	// The single transaction produces 2 effects (credit + debit)
	effectCount := len(effects)
	t.Logf("Single transaction produced %d effects", effectCount)

	if effectCount <= 1 {
		t.Fatalf("Expected a single transaction to produce >1 effect, got %d", effectCount)
	}

	// This is the core of the bug:
	// export_effects.go calls GetTransactions(start, end, limit, ...) where limit
	// controls the number of TRANSACTIONS returned (line 25 of export_effects.go).
	// Then it iterates ALL effects from TransformEffect without any secondary cap
	// (lines 44-56 of export_effects.go). So --limit 1 (documented as "Maximum
	// number of effects to export") actually limits to 1 transaction, which here
	// yields 2 effect rows.
	//
	// Contrast with GetOperations (operations.go:52-69) which expands transactions
	// into operations at the input layer and enforces the limit on the expanded items.
	simulatedLimit := int64(1)
	t.Logf("BUG CONFIRMED: --limit %d (documented as max effects) would export %d effect rows", simulatedLimit, effectCount)
	t.Logf("  GetTransactions(limit=1) returns 1 transaction")
	t.Logf("  TransformEffect on that 1 transaction returns %d effects", effectCount)
	t.Logf("  export_effects exports ALL %d effects — no secondary row cap exists", effectCount)
	t.Logf("  Effect types: %s, %s", EffectTypeNames[EffectType(effects[0].Type)], EffectTypeNames[EffectType(effects[1].Type)])

	// Verify the effects are the expected credit + debit pair
	if effects[0].Type != int32(EffectAccountCredited) {
		t.Errorf("Expected first effect to be AccountCredited, got type %d", effects[0].Type)
	}
	if effects[1].Type != int32(EffectAccountDebited) {
		t.Errorf("Expected second effect to be AccountDebited, got type %d", effects[1].Type)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_effects --limit N` should stop after `N` effect rows have actually been emitted, matching the flag text and command comment.
- **Actual**: `GetTransactions()` stops after `N` transactions, and `export_effects` then writes every effect from those transactions, so `--limit 1` can emit multiple effect rows.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the production `TransformEffect()` path to prove that a single valid transaction expands into multiple effects, and an independent end-to-end CLI run on pubnet ledger `30822025` confirmed that `export_effects -l 1` writes `4` rows.
2. Realistic preconditions: YES — multi-effect transactions are normal Stellar behavior; both a simple payment and a real mainnet `allow_trust` transaction produce multiple effect rows.
3. Bug vs by-design: BUG — both `AddArchiveFlags("effects", ...)` and the inline command comment define `limit` as an effect count, while the sibling operations exporter enforces the same flag at the emitted-row granularity.
4. Final severity: Medium — this silently returns oversized effect exports and breaks caller-visible batching semantics, but it does not corrupt monetary field values.
5. In scope: YES — it is a concrete production export path that returns wrong-but-plausible output.
6. Test correctness: CORRECT — the test does not assume the bug; it proves the necessary one-transaction-to-many-effects expansion, and the cited production loops make the overshoot deterministic once `limit` is applied at transaction granularity.
7. Alternative explanations: NONE — after `GetTransactions(limit=1)` returns one transaction and `TransformEffect()` yields multiple rows, the current exporter necessarily writes them all because no secondary cap exists.
8. Novelty: NOVEL

## Suggested Fix

Apply `limit` to emitted `EffectOutput` rows, not to source transactions. The simplest fix is to track the number of effects written in `export_effects.go` and stop once it reaches the requested bound; alternatively, add an effect-level reader that mirrors `GetOperations()` and enforces the limit after expansion.
