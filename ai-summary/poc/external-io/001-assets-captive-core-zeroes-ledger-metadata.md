# H001: `export_assets --captive-core` zeroes `closed_at` and `ledger_sequence`

**Date**: 2026-04-13
**Subsystem**: external-io
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_assets` is run with `--captive-core`, each exported asset row should still carry the real ledger metadata for the operation that introduced the asset: `closed_at` should match that ledger's close time and `ledger_sequence` should match the originating ledger sequence.

## Mechanism

The `--captive-core` branch does not supply a real `LedgerCloseMeta` to `TransformAsset()`. `GetPaymentOperationsHistoryArchive()` hard-codes `LedgerCloseMeta: xdr.LedgerCloseMeta{}`, but `TransformAsset()` unconditionally derives `ClosedAt` and `LedgerSequence` from that value. As a result, the row keeps the correct asset identifiers but silently exports `closed_at=1970-01-01T00:00:00Z` and `ledger_sequence=0`.

## Trigger

Run `stellar-etl export_assets --captive-core --start-ledger <s> --end-ledger <e>` over any range containing at least one `Payment` or `ManageSellOffer` operation that introduces an asset.

## Target Code

- `cmd/export_assets.go:30-45` — routes `--captive-core` to the history-archive reader and passes the returned struct directly into `TransformAsset()`
- `internal/input/assets_history_archive.go:21-39` — populates `LedgerSeqNum` but hard-codes `LedgerCloseMeta: xdr.LedgerCloseMeta{}`
- `internal/transform/asset.go:45-50` — derives `ClosedAt` and `LedgerSequence` exclusively from `LedgerCloseMeta`
- `internal/utils/main.go:41-46` — converts a zero XDR `TimePoint` into the Unix epoch instead of rejecting it

## Evidence

The history-archive asset reader already knows the real ledger sequence (`seq`) and reads the archive ledger itself, but it discards ledger-close metadata by filling `LedgerCloseMeta` with the zero value. `TransformAsset()` then uses `utils.GetCloseTime(lcm)` and `utils.GetLedgerSequence(lcm)` with no alternate source, so the exported metadata necessarily comes from the zero LCM rather than the actual ledger.

## Anti-Evidence

The asset identity fields (`asset_code`, `asset_issuer`, `asset_type`, `asset_id`) are still sourced from the operation and should remain correct. This only corrupts the ledger/timestamp columns, so the trigger needs a caller that relies on `--captive-core` output rather than the default datastore path.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `--captive-core` flag (deprecated, `internal/utils/main.go:238`) routes `export_assets` to `GetPaymentOperationsHistoryArchive()` which constructs `AssetTransformInput` with `LedgerCloseMeta: xdr.LedgerCloseMeta{}` (zero value). The zero-value `LedgerCloseMeta` has `V=0` but `V0=nil`. When `TransformAsset()` calls `utils.GetCloseTime(lcm)` → `lcm.LedgerHeaderHistoryEntry()` → `MustV0()` → `GetV0()`, the code attempts `result = *u.V0` where `V0` is nil, causing a **nil pointer dereference panic** — not silent zeroes as the hypothesis claims. The actual impact is worse operationally: the entire `export_assets --captive-core` command crashes.

### Code Paths Examined

- `cmd/export_assets.go:30-34` — confirms `UseCaptiveCore` routes to `GetPaymentOperationsHistoryArchive()`
- `internal/input/assets_history_archive.go:33-39` — confirms `LedgerCloseMeta: xdr.LedgerCloseMeta{}` hard-coded zero value
- `internal/input/assets.go:43-49` — contrast: the non-captive-core path passes real `ledger` as LCM
- `internal/transform/asset.go:45-50` — `GetCloseTime(lcm)` and `GetLedgerSequence(lcm)` called unconditionally on the zero LCM
- `internal/utils/main.go:968-976` — `GetCloseTime()` and `GetLedgerSequence()` call `lcm.LedgerHeaderHistoryEntry()`
- `go-stellar-sdk xdr/xdr_generated.go:20065-20070` — `LedgerCloseMeta` struct: V0 is `*LedgerCloseMetaV0` (pointer, nil at zero)
- `go-stellar-sdk xdr/xdr_generated.go:20123-20131` — `MustV0()` calls `GetV0()`
- `go-stellar-sdk xdr/xdr_generated.go:20135-20144` — `GetV0()` does `result = *u.V0` — nil pointer dereference when V0 is nil
- `go-stellar-sdk xdr/ledger_close_meta.go:8-18` — `LedgerHeaderHistoryEntry()` switches on V, case 0 calls `MustV0()`
- `internal/utils/main.go:238` — `--captive-core` flag is deprecated but still registered and functional

### Findings

**Mechanism correction**: The hypothesis claims silent export of `closed_at=1970-01-01T00:00:00Z` and `ledger_sequence=0`. This is incorrect. The actual behavior is a **nil pointer dereference panic** because:

1. `xdr.LedgerCloseMeta{}` zero value has `V=0` and `V0=nil` (pointer field)
2. `LedgerHeaderHistoryEntry()` matches `case 0:` and calls `l.MustV0()`
3. `MustV0()` → `GetV0()` → `result = *u.V0` dereferences nil → **PANIC**

The code never reaches `TimePointToUTCTimeStamp()` (hypothesis target `internal/utils/main.go:41-46`) because the panic occurs earlier in the call chain.

**Core finding is valid**: The `--captive-core` code path for `export_assets` is completely broken. `GetPaymentOperationsHistoryArchive()` has real ledger data available (from `backend.GetLedgerArchive(ctx, seq)`) but discards it, substituting a zero `LedgerCloseMeta` that cannot be used by `TransformAsset()`. The non-captive-core path (`GetPaymentOperations()`) correctly passes the real `LedgerCloseMeta` obtained from `backend.GetLedger()`.

**Impact**: Running `export_assets --captive-core` with any range containing Payment or ManageSellOffer operations will crash with an unrecoverable panic. No output is produced. This is a High severity finding because it renders the entire `--captive-core` export path non-functional, though the flag is deprecated.

### PoC Guidance

- **Test file**: `internal/transform/asset_test.go` (or `cmd/export_assets_test.go` if it exists)
- **Setup**: Construct an `AssetTransformInput` with a valid Payment operation but `LedgerCloseMeta: xdr.LedgerCloseMeta{}` (zero value)
- **Steps**: Call `transform.TransformAsset()` with the zero LCM and a valid Payment operation
- **Assertion**: Assert that the call panics with a nil pointer dereference (use `recover()` or `assert.Panics`). This demonstrates that the `--captive-core` path is non-functional. Alternatively, demonstrate that the non-captive-core path (with a real LCM) succeeds, confirming the asymmetry.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-13
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestAssetTransformPanicsWithZeroLedgerCloseMeta"
**Test Language**: Go

### Demonstration

The test confirms that calling `TransformAsset()` with a zero-value `xdr.LedgerCloseMeta{}` (exactly as constructed by `GetPaymentOperationsHistoryArchive`) causes an unrecoverable nil pointer dereference panic. The same call with a valid `LedgerCloseMeta` (as used by the non-captive-core path) succeeds and produces correct output, proving the asymmetry between the two code paths.

### Test Body

```go
func TestAssetTransformPanicsWithZeroLedgerCloseMeta(t *testing.T) {
	// 1. Construct a valid Payment operation (same as existing tests)
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

	// 2. Use a zero-value LedgerCloseMeta — exactly what
	//    GetPaymentOperationsHistoryArchive hard-codes at
	//    internal/input/assets_history_archive.go:38
	zeroLCM := xdr.LedgerCloseMeta{}

	// 3. Demonstrate that calling TransformAsset with the zero LCM panics
	panicked := false
	var panicValue interface{}
	func() {
		defer func() {
			if r := recover(); r != nil {
				panicked = true
				panicValue = r
			}
		}()
		_, _ = TransformAsset(paymentOp, 0, 0, 1, zeroLCM)
	}()

	if !panicked {
		t.Errorf("Expected TransformAsset to panic with zero LedgerCloseMeta, but it did not")
	}
	t.Logf("TransformAsset panicked as expected: %v", panicValue)

	// 4. Contrast: the same call with a valid LCM succeeds (non-captive-core path)
	validLCM := genericLedgerCloseMeta
	output, err := TransformAsset(paymentOp, 0, 0, 1, validLCM)
	if err != nil {
		t.Fatalf("TransformAsset with valid LCM should succeed, got error: %v", err)
	}
	if output.AssetCode != "USDT" {
		t.Errorf("Expected AssetCode=USDT, got %s", output.AssetCode)
	}
	t.Logf("Valid LCM path succeeded: AssetCode=%s, LedgerSequence=%d, ClosedAt=%v",
		output.AssetCode, output.LedgerSequence, output.ClosedAt)

	t.Logf("BUG CONFIRMED: export_assets --captive-core path panics due to zero LedgerCloseMeta")
}
```

### Test Output

```
=== RUN   TestAssetTransformPanicsWithZeroLedgerCloseMeta
    data_integrity_poc_test.go:338: TransformAsset panicked as expected: runtime error: invalid memory address or nil pointer dereference
    data_integrity_poc_test.go:349: Valid LCM path succeeded: AssetCode=USDT, LedgerSequence=2, ClosedAt=1970-01-01 00:00:10 +0000 UTC
    data_integrity_poc_test.go:352: BUG CONFIRMED: export_assets --captive-core path panics due to zero LedgerCloseMeta
--- PASS: TestAssetTransformPanicsWithZeroLedgerCloseMeta (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.697s
```
