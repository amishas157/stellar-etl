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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestEffectsUseZeroBasedIndex"
**Test Language**: Go

### Demonstration

The test constructs a SetOptions transaction that produces two signer effects, then calls `operation.effects()` and verifies the first effect gets `EffectIndex=0` and `EffectId="197568499713-0"` — confirming zero-based indexing. Upstream Horizon would assign `order=1` and format the ID as `"197568499713-1"` for the same first effect. The test proves every effect row is systematically shifted by 1 compared to Horizon's history_effects contract.

### Test Body

```go
// TestEffectsUseZeroBasedIndex demonstrates that the ETL assigns zero-based
// EffectIndex and EffectId values, whereas upstream Horizon uses 1-based
// ordering. The first effect for any operation gets index=0 and id="<opid>-0"
// instead of the Horizon-compatible index=1 and id="<opid>-1".
func TestEffectsUseZeroBasedIndex(t *testing.T) {
	// Build a SetOptions transaction that changes signer weights, producing
	// multiple effects. This mirrors TestOperationEffectsSetOptionsSignersOrder.
	transaction := ingest.LedgerTransaction{
		UnsafeMeta: createTransactionMeta([]xdr.OperationMeta{
			{
				Changes: []xdr.LedgerEntryChange{
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
						State: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeAccount,
								Account: &xdr.AccountEntry{
									AccountId: xdr.MustAddress("GC3C4AKRBQLHOJ45U4XG35ESVWRDECWO5XLDGYADO6DPR3L7KIDVUMML"),
									Signers: []xdr.Signer{
										{
											Key:    xdr.MustSigner("GCBBDQLCTNASZJ3MTKAOYEOWRGSHDFAJVI7VPZUOP7KXNHYR3HP2BUKV"),
											Weight: 10,
										},
									},
								},
							},
						},
					},
					{
						Type: xdr.LedgerEntryChangeTypeLedgerEntryUpdated,
						Updated: &xdr.LedgerEntry{
							Data: xdr.LedgerEntryData{
								Type: xdr.LedgerEntryTypeAccount,
								Account: &xdr.AccountEntry{
									AccountId: xdr.MustAddress("GC3C4AKRBQLHOJ45U4XG35ESVWRDECWO5XLDGYADO6DPR3L7KIDVUMML"),
									Signers: []xdr.Signer{
										{
											Key:    xdr.MustSigner("GCBBDQLCTNASZJ3MTKAOYEOWRGSHDFAJVI7VPZUOP7KXNHYR3HP2BUKV"),
											Weight: 16,
										},
										{
											Key:    xdr.MustSigner("GCR3TQ2TVH3QRI7GQMC3IJGUUBR32YQHWBIKIMTYRQ2YH4XUTDB75UKE"),
											Weight: 14,
										},
									},
								},
							},
						},
					},
				},
			},
		}),
	}
	transaction.Index = 1
	transaction.Envelope.Type = xdr.EnvelopeTypeEnvelopeTypeTx
	aid := xdr.MustAddress("GCBBDQLCTNASZJ3MTKAOYEOWRGSHDFAJVI7VPZUOP7KXNHYR3HP2BUKV")
	transaction.Envelope.V1 = &xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: aid.ToMuxedAccount(),
		},
	}

	operation := transactionOperationWrapper{
		index:          0,
		transaction:    transaction,
		operation:      xdr.Operation{Body: xdr.OperationBody{Type: xdr.OperationTypeSetOptions, SetOptionsOp: &xdr.SetOptionsOp{}}},
		ledgerSequence: 46,
		ledgerClosed:   genericCloseTime.UTC(),
	}

	effects, err := operation.effects()
	if err != nil {
		t.Fatalf("effects() returned error: %v", err)
	}
	if len(effects) < 2 {
		t.Fatalf("expected at least 2 effects, got %d", len(effects))
	}

	// --- Demonstrate zero-based indexing (the bug) ---

	// The first effect has index 0, not 1 as Horizon would produce.
	if effects[0].EffectIndex != 0 {
		t.Fatalf("expected first EffectIndex=0 (zero-based), got %d", effects[0].EffectIndex)
	}

	// The second effect has index 1, not 2 as Horizon would produce.
	if effects[1].EffectIndex != 1 {
		t.Fatalf("expected second EffectIndex=1 (zero-based), got %d", effects[1].EffectIndex)
	}

	// The EffectId for the first effect ends in "-0" instead of Horizon's "-1".
	expectedSuffix := fmt.Sprintf("%d-0", effects[0].OperationID)
	if effects[0].EffectId != expectedSuffix {
		t.Fatalf("expected first EffectId=%q (zero-based), got %q", expectedSuffix, effects[0].EffectId)
	}

	// Horizon would produce 1-based IDs: "<opid>-1", "<opid>-2", etc.
	// The ETL produces 0-based: "<opid>-0", "<opid>-1", etc.
	horizonFirstEffectId := fmt.Sprintf("%d-1", effects[0].OperationID)
	if effects[0].EffectId == horizonFirstEffectId {
		t.Fatalf("EffectId unexpectedly matches Horizon 1-based convention; zero-based bug may be fixed")
	}

	// Verify the systematic off-by-one across all effects.
	for i, eff := range effects {
		wantIdx := uint32(i) // ETL zero-based
		if eff.EffectIndex != wantIdx {
			t.Errorf("effect[%d]: EffectIndex=%d, want %d (zero-based)", i, eff.EffectIndex, wantIdx)
		}
		wantId := fmt.Sprintf("%d-%d", eff.OperationID, i)
		if eff.EffectId != wantId {
			t.Errorf("effect[%d]: EffectId=%q, want %q (zero-based)", i, eff.EffectId, wantId)
		}
		// Under Horizon convention these would be i+1 based:
		horizonIdx := uint32(i + 1)
		if eff.EffectIndex == horizonIdx {
			t.Errorf("effect[%d]: unexpectedly matches Horizon 1-based index %d", i, horizonIdx)
		}
		horizonId := fmt.Sprintf("%d-%d", eff.OperationID, i+1)
		if strings.HasSuffix(eff.EffectId, fmt.Sprintf("-%d", i+1)) && eff.EffectId == horizonId {
			t.Errorf("effect[%d]: unexpectedly matches Horizon 1-based id %q", i, horizonId)
		}
	}

	t.Logf("BUG CONFIRMED: effect exports use zero-based index/id instead of Horizon 1-based order")
	for i, eff := range effects {
		t.Logf("  effect[%d]: EffectIndex=%d  EffectId=%q  (Horizon would use index=%d id=%q)",
			i, eff.EffectIndex, eff.EffectId, i+1, fmt.Sprintf("%d-%d", eff.OperationID, i+1))
	}
}
```

### Test Output

```
=== RUN   TestEffectsUseZeroBasedIndex
    data_integrity_poc_test.go:292: BUG CONFIRMED: effect exports use zero-based index/id instead of Horizon 1-based order
    data_integrity_poc_test.go:294:   effect[0]: EffectIndex=0  EffectId="197568499713-0"  (Horizon would use index=1 id="197568499713-1")
    data_integrity_poc_test.go:294:   effect[1]: EffectIndex=1  EffectId="197568499713-1"  (Horizon would use index=2 id="197568499713-2")
--- PASS: TestEffectsUseZeroBasedIndex (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.741s
```
