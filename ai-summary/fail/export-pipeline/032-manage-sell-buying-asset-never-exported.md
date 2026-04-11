# H002: `export_assets` drops the buy-side asset from `ManageSellOffer` discovery

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Once the asset export pipeline chooses to treat `ManageSellOffer` as an
asset-bearing operation, the exported row should surface the asset that the
operation newly introduces to the range. A sell offer that sells native XLM in
order to buy a new issued asset should not emit only the already-known native
asset and silently miss the novel buy-side asset.

## Mechanism

Both asset readers explicitly admit `ManageSellOffer` operations, but
`TransformAsset()` always selects `opSellOf.Selling` and never inspects
`opSellOf.Buying`. That means a valid range where the only first-seen reference
to asset B appears on the buy side of a sell offer will export either asset A
(the selling side) or nothing at all after `seenIDs` deduplication, even though
asset B is the new asset the offer introduces.

## Trigger

1. Find or construct a `ManageSellOffer` where:
   1. `Selling = native` (or some already-seen asset A)
   2. `Buying =` a distinct issued asset B not otherwise referenced earlier in the range
2. Run `stellar-etl export_assets` over a range that contains this offer but no
   earlier payment/offer referencing asset B.
3. The export emits native/A or skips the row after dedupe, while asset B never
   appears in `history_assets`.

## Target Code

- `internal/input/assets.go:40-49` — datastore reader intentionally includes `ManageSellOffer` in asset discovery
- `internal/input/assets_history_archive.go:30-39` — history-archive reader includes the same operation type
- `internal/transform/asset.go:17-36` — `ManageSellOffer` branch hard-codes `asset = opSellOf.Selling`
- `cmd/export_assets.go:53-58` — post-transform dedupe can suppress the already-known selling asset and leave no row for the new buy-side asset

## Evidence

The current pipeline clearly treats `ManageSellOffer` as in-scope input: both
readers append those operations, and `TransformAsset()` has an explicit
`ManageSellOffer` branch. Within that active path, however, only one of the two
referenced assets is exported. For offers that sell a common asset in order to
acquire a new one, the only asset written is the already-known selling side.

## Anti-Evidence

The README still describes `export_assets` in payment-operation terms, so a
reviewer may argue that the entire `ManageSellOffer` branch is legacy or broader
than the intended contract. Even under that interpretation, the current code is
internally inconsistent: if `ManageSellOffer` stays enabled, the one-sided
`Selling` extraction is an observable data-loss choice.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Both asset readers (`GetPaymentOperations` in `assets.go:42` and `GetPaymentOperationsHistoryArchive` in `assets_history_archive.go:32`) filter for `OperationTypeManageSellOffer`, feeding each matching operation to `TransformAsset()`. The transform function's `ManageSellOffer` branch (asset.go:24-29) unconditionally assigns `asset = opSellOf.Selling` and returns a single `AssetOutput`. The `Buying` field of the operation is never accessed. The export command's `seenIDs` dedupe map (export_assets.go:40-58) then suppresses already-seen selling-side assets, but cannot compensate for a buying-side asset that was never extracted. The function signature (`TransformAsset → AssetOutput`) structurally returns one asset per call, so even the caller loop cannot produce two rows from one operation.

### Code Paths Examined

- `internal/input/assets.go:42` — datastore reader includes `OperationTypeManageSellOffer` in the filter predicate
- `internal/input/assets_history_archive.go:32` — history-archive reader includes the same filter predicate
- `internal/transform/asset.go:24-29` — `ManageSellOffer` case extracts only `opSellOf.Selling`; `opSellOf.Buying` is never referenced
- `internal/transform/asset.go:14` — function returns a single `AssetOutput`, not a slice
- `cmd/export_assets.go:44-56` — caller iterates one transform result per operation; dedupe map can only suppress, not add, rows

### Findings

The bug is confirmed through direct code reading:

1. **Both readers admit ManageSellOffer** — `assets.go:42` and `assets_history_archive.go:32` both check `op.Body.Type == xdr.OperationTypeManageSellOffer`.

2. **Transform extracts only the selling side** — `asset.go:29` assigns `asset = opSellOf.Selling`. The `Buying` field from `ManageSellOfferOp` is never accessed anywhere in the function.

3. **Single-return architecture prevents implicit fix** — `TransformAsset` returns `(AssetOutput, error)`, not a slice. Even if a caller wanted both assets, the current signature cannot deliver them.

4. **Dedupe amplifies the loss** — When the selling asset is already known (common for native XLM), the `seenIDs` check at `export_assets.go:54` skips the row entirely. The buying-side asset was never extracted, so it produces zero rows total for that operation.

5. **Neither `ManageBuyOffer` nor `PathPayment*` are in scope** — The filter at `asset.go:18` rejects everything except `Payment` and `ManageSellOffer`. This is not a broader asset-discovery tool; the partial ManageSellOffer support is specifically what creates the inconsistency.

**Impact**: Any asset whose first (or only) appearance in an exported ledger range is on the buying side of a `ManageSellOffer` will be silently absent from the `history_assets` export. Downstream BigQuery tables that join on `asset_id` will have no matching row for that asset.

### PoC Guidance

- **Test file**: `internal/transform/asset_test.go` (or create a new test in `cmd/` if integration-level)
- **Setup**: Construct a `ManageSellOfferOp` with `Selling = native` and `Buying = CreditAlphanum4{AssetCode: "TEST", Issuer: <some account>}`. Wrap it in an `xdr.Operation` with `Body.Type = OperationTypeManageSellOffer`.
- **Steps**: Call `TransformAsset(op, 0, 0, 100, lcm)` and inspect the returned `AssetOutput`.
- **Assertion**: Assert that the returned `AssetOutput.AssetCode` equals `"TEST"` (the buying asset). The current code will return the native asset instead, confirming the bug. Alternatively, demonstrate that calling `TransformAsset` once per operation can never yield both assets — showing the structural gap.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestManageSellOfferBuyingAssetNeverExported"
**Test Language**: Go

### Demonstration

The test constructs a `ManageSellOffer` operation selling native XLM and buying a "TEST" credit asset, then calls `TransformAsset`. The function returns only the native (selling-side) asset with `AssetCode=""`, proving the buying-side "TEST" asset is silently dropped. This confirms that any asset whose only reference in a ledger range is on the buy side of a `ManageSellOffer` will be absent from the `history_assets` export.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestManageSellOfferBuyingAssetNeverExported demonstrates that TransformAsset
// only extracts the Selling side of a ManageSellOffer, silently dropping the
// Buying asset. If a novel asset's only reference in a ledger range is on the
// buy side of a ManageSellOffer, it will be absent from the export.
func TestManageSellOfferBuyingAssetNeverExported(t *testing.T) {
	// Construct a ManageSellOffer: selling native XLM, buying a custom asset "TEST"
	testIssuerAddress := "GBVVRXLMNCJQW3IDDXC3X6XCH35B5Q7QXNMMFPENSOGUPQO7WO7HGZPA"
	testIssuerID, err := xdr.AddressToAccountId(testIssuerAddress)
	if err != nil {
		t.Fatalf("failed to parse test issuer address: %v", err)
	}

	buyingAsset := xdr.Asset{
		Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
		AlphaNum4: &xdr.AlphaNum4{
			AssetCode: xdr.AssetCode4([4]byte{0x54, 0x45, 0x53, 0x54}), // "TEST"
			Issuer:    testIssuerID,
		},
	}

	sellingAsset := xdr.MustNewNativeAsset()

	op := xdr.Operation{
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeManageSellOffer,
			ManageSellOfferOp: &xdr.ManageSellOfferOp{
				Selling: sellingAsset,
				Buying:  buyingAsset,
				Amount:  1000000,
				Price:   xdr.Price{N: 1, D: 1},
			},
		},
	}

	result, err := TransformAsset(op, 0, 0, 2, genericLedgerCloseMeta)
	if err != nil {
		t.Fatalf("TransformAsset returned unexpected error: %v", err)
	}

	// The buying asset is "TEST" — a novel asset introduced by this operation.
	// The current code extracts only opSellOf.Selling (native), so the buying
	// asset is silently lost.
	if result.AssetCode == "TEST" {
		t.Log("TransformAsset correctly returned the buying asset — bug is fixed")
	} else {
		// Bug confirmed: the output contains only the selling-side asset (native),
		// and the buying-side asset "TEST" is never exported.
		t.Errorf("BUG CONFIRMED: ManageSellOffer buying asset lost — got AssetType=%q AssetCode=%q, "+
			"but the operation's Buying asset is TEST issued by %s. "+
			"The buying asset is silently dropped from the export.",
			result.AssetType, result.AssetCode, testIssuerAddress)
	}

	// Additionally demonstrate the structural limitation: TransformAsset returns
	// a single AssetOutput, so it can never yield both assets from one operation.
	// Even if the caller iterates, each ManageSellOffer operation produces at most
	// one asset row — always the selling side.
	if result.AssetType != "native" {
		t.Errorf("Expected selling-side native asset, got type=%q code=%q", result.AssetType, result.AssetCode)
	}
}
```

### Test Output

```
=== RUN   TestManageSellOfferBuyingAssetNeverExported
    data_integrity_poc_test.go:56: BUG CONFIRMED: ManageSellOffer buying asset lost — got AssetType="native" AssetCode="", but the operation's Buying asset is TEST issued by GBVVRXLMNCJQW3IDDXC3X6XCH35B5Q7QXNMMFPENSOGUPQO7WO7HGZPA. The buying asset is silently dropped from the export.
--- FAIL: TestManageSellOfferBuyingAssetNeverExported (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.816s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC actually exercise the claimed issue?** Partially. It correctly exercises the `TransformAsset()` branch for `ManageSellOffer` and proves that the function returns the selling-side asset, but it does not establish that the export is required to emit the buying-side asset.
2. **Are the preconditions realistic?** Yes. A `ManageSellOffer` that sells native XLM and buys a credit asset is a normal Stellar operation shape.
3. **Is the behavior a bug or by design?** By design. The command, reader names, comments, and README consistently describe `export_assets` as exporting assets created from payment operations, not as a complete asset-discovery export over both sides of offer operations.
4. **Does the impact match the claimed severity?** No. Because the finding never establishes that buy-side offer assets are in the command's intended output contract, there is no demonstrated structural corruption to rate as High.
5. **Is the finding in scope?** Only if it were a real correctness defect. The current evidence shows an implementation/documentation scope mismatch theory, not proven wrong output.
6. **Is the test itself correct?** Not as bug proof. The test fails by asserting a stronger contract than the repository documents; it proves the implementation chooses `ManageSellOffer.Selling`, but not that this choice violates a requirement.
7. **Can the results be explained WITHOUT the claimed issue?** Yes. A benign explanation is that `export_assets` is intentionally a narrow payment-oriented asset-reference export and that the `ManageSellOffer` support is legacy or limited behavior rather than a promise to enumerate both assets in an offer.
8. **Is this finding novel?** Possibly, but novelty does not matter once the semantics do not support the claimed bug.

### Rejection Reason

The PoC shows what the code does, but it does not prove that the code is wrong. In this repository, `export_assets` is documented and named around payment-operation asset export, and nothing in the product contract says a `ManageSellOffer` must emit its buying asset or both offer-side assets. Without that requirement, the observed result is explainable as intentionally narrow behavior rather than silent data corruption.

### Failed Checks

- 3. Behavior is by design rather than a demonstrated correctness bug
- 4. Claimed High impact is unsupported because no documented output requirement was violated
- 6. Test proves implementation behavior, not incorrect output against the command contract
- 7. The observed result has a benign explanation consistent with the command's documented scope
