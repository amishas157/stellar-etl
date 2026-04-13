# 023: `export_assets --captive-core` panics on zero `LedgerCloseMeta`

**Date**: 2026-04-13
**Severity**: Medium
**Impact**: Operational correctness / broken asset export path
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_assets` takes a different reader path when `--captive-core` is enabled, and that path constructs each `AssetTransformInput` with `xdr.LedgerCloseMeta{}` instead of real ledger-close metadata. The original hypothesis was wrong about silent `closed_at=1970-01-01` / `ledger_sequence=0` corruption: the export fails earlier because `TransformAsset()` dereferences the zero-value XDR union and panics before any asset row is written.

This is still a real production bug because the deprecated flag remains registered and reachable from the CLI. Any qualifying payment or manage-sell-offer operation on that path crashes the export instead of producing output.

## Root Cause

`GetPaymentOperationsHistoryArchive()` reads the archive ledger for each sequence but discards usable close metadata and hard-codes `LedgerCloseMeta: xdr.LedgerCloseMeta{}`. `TransformAsset()` unconditionally calls `utils.GetCloseTime(lcm)` and `utils.GetLedgerSequence(lcm)`, which call `lcm.LedgerHeaderHistoryEntry()`. For a zero-value `LedgerCloseMeta`, the union discriminant is `V=0` while `V0` is nil, so the XDR helper dereferences a nil pointer and panics.

## Reproduction

During normal operation, running `stellar-etl export_assets --captive-core --start-ledger <s> --end-ledger <e>` over any range containing a `Payment` or `ManageSellOffer` operation reaches the history-archive asset reader. That reader emits `AssetTransformInput` values with zero `LedgerCloseMeta`, and the first attempt to transform one of those operations crashes the command before the exporter can write JSON or Parquet output.

## Affected Code

- `cmd/export_assets.go:assetsCmd.Run:30-45` — routes `--captive-core` to the history-archive asset reader and passes its `LedgerCloseMeta` directly into `TransformAsset`
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:21-39` — hard-codes `LedgerCloseMeta: xdr.LedgerCloseMeta{}`
- `internal/transform/asset.go:TransformAsset:45-50` — unconditionally derives close time and ledger sequence from `LedgerCloseMeta`
- `internal/utils/main.go:GetCloseTime/GetLedgerSequence:968-975` — calls `LedgerHeaderHistoryEntry()` on the supplied `LedgerCloseMeta`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestAssetTransformPanicsWithZeroLedgerCloseMeta`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./internal/transform/... -run TestAssetTransformPanicsWithZeroLedgerCloseMeta -v`.

### Test Body

```go
func TestAssetTransformPanicsWithZeroLedgerCloseMeta(t *testing.T) {
	paymentOp := xdr.Operation{
		SourceAccount: nil,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypePayment,
			PaymentOp: &xdr.PaymentOp{
				Destination: testAccount2,
				Asset:       usdtAsset,
				Amount:      350000000,
			},
		},
	}

	panicked := false
	func() {
		defer func() {
			if recover() != nil {
				panicked = true
			}
		}()
		_, _ = TransformAsset(paymentOp, 0, 0, 1, xdr.LedgerCloseMeta{})
	}()

	if !panicked {
		t.Fatal("expected zero-value LedgerCloseMeta to panic")
	}

	output, err := TransformAsset(paymentOp, 0, 0, 1, genericLedgerCloseMeta)
	if err != nil {
		t.Fatalf("expected valid LedgerCloseMeta to succeed, got error: %v", err)
	}
	if output.AssetCode != "USDT" || output.LedgerSequence != 2 {
		t.Fatalf("unexpected valid output: %+v", output)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_assets --captive-core` should emit asset rows with the real close time and ledger sequence for the originating ledger.
- **Actual**: the history-archive reader supplies a zero `LedgerCloseMeta`, and `TransformAsset()` panics while trying to read close metadata from it.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the exact production `TransformAsset()` path with the same zero `LedgerCloseMeta` value constructed by the captive-core/history-archive reader.
2. Realistic preconditions: YES — the deprecated `--captive-core` flag is still registered on the public CLI, and any range containing a qualifying asset-producing operation reaches this path.
3. Bug vs by-design: BUG — nothing in the CLI contract or code comments indicates that `export_assets --captive-core` is expected to panic instead of exporting data.
4. Final severity: Medium — this is an operational exporter failure, not silent field corruption, because the command crashes before producing rows.
5. In scope: YES — this is a concrete, reproducible correctness bug in the shipped export path.
6. Test correctness: CORRECT — the final PoC proves the bad path panics and contrasts it with the valid path using a real `LedgerCloseMeta`, so it does not pass for tautological reasons.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Populate `AssetTransformInput.LedgerCloseMeta` with usable ledger-close metadata in `GetPaymentOperationsHistoryArchive()`, or change the asset transform interface so close time and ledger sequence are supplied directly from archive data instead of forcing callers to fabricate an `xdr.LedgerCloseMeta`. At minimum, reject zero-value `LedgerCloseMeta` before dereferencing it so the command returns a handled error rather than panicking.
