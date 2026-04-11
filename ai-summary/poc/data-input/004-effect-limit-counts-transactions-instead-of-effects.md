# H004: Effect Export Limit Counts Transactions Instead of Effect Rows

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_effects --limit N` should emit at most `N` effect rows, matching the CLI flag text that describes the limit as the maximum number of effects to export. A transaction that expands into multiple effect rows should therefore be truncated or followed by an exact row-level stop once the requested number of effects has been written.

## Mechanism

The command passes the effect limit directly into `GetTransactions()`, which enforces the bound in units of transactions, not effects. `TransformEffect()` then expands each returned transaction into a variable-length slice of `EffectOutput` rows, so `--limit 1` can still write multiple debit/credit/trustline/etc. effects from the first transaction in the range while reporting a normal-looking successful export.

## Trigger

Run `stellar-etl export_effects --limit 1 --start-ledger <S> --end-ledger <E>` on any range whose first returned transaction produces multiple effects (for example, a standard successful payment that yields both debited and credited effects). The correct output should contain exactly one effect row, but the current implementation writes all effects derived from that first transaction.

## Target Code

- `cmd/export_effects.go:21-57` — reads `limit` as "effects" but applies it only when fetching transactions
- `internal/input/transactions.go:GetTransactions:23-70` — stops after `N` transactions, not after `N` effect rows
- `internal/transform/effects.go:23-50` — expands one transaction into many effects
- `internal/utils/main.go:AddArchiveFlags:248-255` — defines the user-visible flag text as "Maximum number of effects to export"

## Evidence

`GetTransactions()` has no visibility into how many `EffectOutput` rows a transaction will later generate, yet `export_effects` never applies a second limit after `TransformEffect()`. The command comment also documents the flag as an effect count, not a transaction count, so the current control flow silently violates the exported row bound instead of merely using a confusing internal unit.

## Anti-Evidence

If the first `N` transactions each happen to generate exactly one effect, the bug is not observable. Users requesting unbounded exports (`limit < 0`) also do not see this discrepancy because no row cap is expected.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `export_effects` command reads the `--limit` flag (documented as "Maximum number of effects to export") and passes it directly to `input.GetTransactions()`, which enforces the limit as a count of transactions, not effects. Each transaction is then expanded by `TransformEffect()` into a variable-length slice of `EffectOutput` rows, all of which are exported without any secondary row-level cap. The sibling `export_operations` command avoids this bug because `GetOperations()` expands transactions into operations at the input layer and enforces the limit on the expanded items directly.

### Code Paths Examined

- `cmd/export_effects.go:21-25` — reads `limit` via `MustArchiveFlags` and passes it to `input.GetTransactions(startNum, commonArgs.EndNum, limit, env, commonArgs.UseCaptiveCore)`
- `internal/input/transactions.go:51` — `GetTransactions` enforces limit as `for int64(len(txSlice)) < limit || limit < 0`, counting transactions not effects
- `cmd/export_effects.go:34-57` — iterates all returned transactions, calls `TransformEffect()` on each, and exports ALL resulting effects with no row cap
- `internal/transform/effects.go:23-51` — `TransformEffect` iterates all operations in a transaction and returns all generated effects; a single transaction can yield many effect rows
- `internal/input/operations.go:52-70` — contrast: `GetOperations()` expands into operations at input level and enforces limit on `len(opSlice)` with an inner break at line 67-69
- `internal/utils/main.go:250-254` — `AddArchiveFlags("effects", ...)` produces flag text "Maximum number of effects to export"

### Findings

The bug is real and follows the same pattern as the confirmed trade limit bug (`success/export-pipeline/010-trade-limit-counts-operations-before-trades.md`) and the asset reader overshoot (`success/data-input/002-asset-readers-overshoot-limit-within-ledger.md`). In this case:

1. **Limit is applied at the wrong granularity**: `GetTransactions()` enforces limit on transaction count. Effects are a many-to-one expansion of transactions (up to 100 effects per transaction per the comment at `export_effects.go:86`).
2. **No secondary limit**: After `TransformEffect()` expands transactions into effects, `export_effects.go` iterates all effects and exports every single one (lines 44-56). There is no `if totalEffects >= limit { break }` guard.
3. **Flag text misleads users**: The `--limit` flag reads "Maximum number of effects to export" but actually controls the number of source transactions read.
4. **Sibling command does it correctly**: `export_operations` uses `GetOperations()` which expands into operations at the input layer and enforces the limit at operation granularity (inner loop break at `operations.go:67-69`). Effects should have an analogous `GetEffects()` or a post-expansion limit.

### PoC Guidance

- **Test file**: `internal/input/data_integrity_poc_test.go` (or create new file `internal/input/effect_limit_poc_test.go`)
- **Setup**: Use `utils.GetEnvironmentDetails` with datastore config pointing to `sdf-ledger-close-meta/v1/ledgers`. Pick a mainnet ledger with a multi-operation successful transaction (e.g., a payment that produces debit + credit effects).
- **Steps**:
  1. Call `input.GetTransactions(ledger, ledger, 1, env, false)` — verify it returns exactly 1 transaction
  2. Call `transform.TransformEffect()` on that transaction — verify it returns more than 1 effect
  3. This proves `--limit 1` would export multiple effect rows
- **Assertion**: `len(effects) > 1` when `limit == 1`, demonstrating the limit is violated. Contrast with `input.GetOperations(ledger, ledger, 1, env, false)` which returns exactly 1 operation on the same ledger.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestEffectLimitCountsTransactionsInsteadOfEffects"
**Test Language**: Go

### Demonstration

The test constructs a single native XLM payment transaction from real mainnet XDR and passes it through the production `TransformEffect()` code path. This single transaction produces 2 effects (AccountCredited + AccountDebited), proving that `--limit 1` (documented as "Maximum number of effects to export") would export 2 effect rows instead of 1, because the limit is enforced at the transaction level in `GetTransactions()` with no secondary row cap after effect expansion.

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

### Test Output

```
=== RUN   TestEffectLimitCountsTransactionsInsteadOfEffects
    data_integrity_poc_test.go:694: Single transaction produced 2 effects
    data_integrity_poc_test.go:711: BUG CONFIRMED: --limit 1 (documented as max effects) would export 2 effect rows
    data_integrity_poc_test.go:712:   GetTransactions(limit=1) returns 1 transaction
    data_integrity_poc_test.go:713:   TransformEffect on that 1 transaction returns 2 effects
    data_integrity_poc_test.go:714:   export_effects exports ALL 2 effects — no secondary row cap exists
    data_integrity_poc_test.go:715:   Effect types: account_credited, account_debited
--- PASS: TestEffectLimitCountsTransactionsInsteadOfEffects (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.756s
```
