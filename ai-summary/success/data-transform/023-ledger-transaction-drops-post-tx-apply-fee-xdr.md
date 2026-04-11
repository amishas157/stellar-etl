# 023: `ledger_transaction` drops protocol-23+ `PostTxApplyFeeChanges`

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformLedgerTransaction()` base64-encodes only `LedgerTransaction.UnsafeMeta` and `LedgerTransaction.FeeChanges` into the raw `ledger_transaction` export. On protocol 23+, the SDK surfaces fee-refund mutations separately in `LedgerTransaction.PostTxApplyFeeChanges`, so those post-apply fee changes are silently omitted from every exported row.

## Root Cause

The `ledger_transaction` schema still reflects the pre-protocol-23 layout: it has fields for `tx_meta` and `tx_fee_meta`, but no field for post-apply fee-processing XDR. `TransformLedgerTransaction()` therefore serializes the old blobs and drops the new `PostTxApplyFeeChanges` slice even though sibling transaction logic already knows that protocol-23+ refunds moved there.

## Reproduction

Use the existing P23+ fee-bump Soroban fixture from `makeTransactionTestInput()`, which includes both pre-apply `FeeChanges` and non-empty `PostTxApplyFeeChanges`. `TransformLedgerTransaction()` emits `tx_meta == MarshalBase64(tx.UnsafeMeta)` and `tx_fee_meta == MarshalBase64(tx.FeeChanges)`, but there is no output field that preserves `MarshalBase64(tx.PostTxApplyFeeChanges)`.

## Affected Code

- `internal/transform/ledger_transaction.go:TransformLedgerTransaction:17-35` — marshals only `Envelope`, `Result`, `UnsafeMeta`, `FeeChanges`, and ledger history
- `internal/transform/schema.go:LedgerTransactionOutput:86-94` — raw export schema has no field for post-apply fee changes
- `internal/transform/transaction.go:TransformTransaction:195-200` — sibling transform explicitly appends `PostTxApplyFeeChanges` for P23+ refund handling

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPostTxApplyFeeChangesDroppedFromLedgerTransaction`
- **Test language**: go
- **How to run**: Append the test body below to the target test file, then run `go build ./...` and `go test ./internal/transform/... -run TestPostTxApplyFeeChangesDroppedFromLedgerTransaction -v`.

### Test Body

```go
func TestPostTxApplyFeeChangesDroppedFromLedgerTransaction(t *testing.T) {
	transactions, headers, err := makeTransactionTestInput()
	if err != nil {
		t.Fatalf("makeTransactionTestInput returned error: %v", err)
	}
	if len(transactions) < 4 || len(headers) < 4 {
		t.Fatalf("expected the transaction test fixture to include a P23+ fee-bump case, got %d txs and %d headers", len(transactions), len(headers))
	}

	tx := transactions[3]
	lhe := headers[3]

	output, err := TransformLedgerTransaction(tx, lhe)
	if err != nil {
		t.Fatalf("TransformLedgerTransaction returned error: %v", err)
	}

	metaB64, err := xdr.MarshalBase64(tx.UnsafeMeta)
	if err != nil {
		t.Fatalf("Failed to marshal expected tx meta: %v", err)
	}
	feeMetaB64, err := xdr.MarshalBase64(tx.FeeChanges)
	if err != nil {
		t.Fatalf("Failed to marshal expected fee changes: %v", err)
	}
	postApplyB64, err := xdr.MarshalBase64(tx.PostTxApplyFeeChanges)
	if err != nil {
		t.Fatalf("Failed to marshal expected post-apply fee changes: %v", err)
	}

	if len(tx.FeeChanges) == 0 || len(tx.PostTxApplyFeeChanges) == 0 {
		t.Fatalf("fixture must contain both fee changes and post-apply fee changes")
	}

	if output.TxMeta != metaB64 {
		t.Fatalf("TransformLedgerTransaction changed tx_meta unexpectedly")
	}
	if output.TxFeeMeta != feeMetaB64 {
		t.Fatalf("TransformLedgerTransaction changed tx_fee_meta unexpectedly")
	}
	if output.TxFeeMeta == postApplyB64 || output.TxMeta == postApplyB64 {
		t.Fatal("PostTxApplyFeeChanges unexpectedly preserved in an existing output field — hypothesis disproven")
	}

	t.Errorf("BUG CONFIRMED: TransformLedgerTransaction exported tx_meta from UnsafeMeta and tx_fee_meta from FeeChanges, "+
		"but dropped PostTxApplyFeeChanges entirely. "+
		"fee_changes_b64_len=%d post_apply_b64_len=%d ledger_transaction_output_has_no_post_apply_field",
		len(feeMetaB64), len(postApplyB64))
}
```

## Expected vs Actual Behavior

- **Expected**: The raw `ledger_transaction` export preserves every transaction-processing XDR blob needed to reconstruct the final fee lifecycle, including protocol-23+ post-apply fee refunds.
- **Actual**: `tx_meta` contains only `UnsafeMeta`, `tx_fee_meta` contains only pre-apply `FeeChanges`, and `PostTxApplyFeeChanges` is not exported anywhere.

## Adversarial Review

1. Exercises claimed bug: YES — the test uses the project's existing protocol-25 fee-bump fixture with populated `PostTxApplyFeeChanges` and calls the production `TransformLedgerTransaction()` path.
2. Realistic preconditions: YES — the fixture is already used to test P23+ Soroban fee handling and mirrors a real fee-bump transaction shape exported by the CLI.
3. Bug vs by-design: BUG — `TransformTransaction()` explicitly handles `PostTxApplyFeeChanges` for P23+ refunds, showing the new field is semantically required and the raw ledger-transaction path is simply stale.
4. Final severity: High — exported rows stay parseable but silently omit a structurally important raw XDR blob, breaking downstream fee-lifecycle reconstruction.
5. In scope: YES — this is silent transform/export data corruption in a production output path.
6. Test correctness: CORRECT — it checks exact output blobs against the transaction's real `UnsafeMeta`, `FeeChanges`, and `PostTxApplyFeeChanges`, so the failure is specifically the omission of the post-apply blob.
7. Alternative explanations: NONE — `LedgerTransactionOutput` has no other field that can carry post-apply fee-processing XDR, and `tx_ledger_history` only serializes ledger-header history.
8. Novelty: NOVEL

## Suggested Fix

Add a dedicated raw output field for `PostTxApplyFeeChanges` to `LedgerTransactionOutput` (and any matching export schemas), then serialize `transaction.PostTxApplyFeeChanges` alongside the existing `tx_meta` and `tx_fee_meta` fields instead of overloading either existing column.
