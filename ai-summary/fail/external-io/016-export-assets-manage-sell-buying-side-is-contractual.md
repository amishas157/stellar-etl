# H003: `export_assets` only discovers the selling side of `manage_sell_offer`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Structural data corruption: offer-based asset discovery can miss legitimate asset rows
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Once `manage_sell_offer` is included as an asset-discovery source, a successful offer should be able to contribute both assets named by that operation. A range where asset `Y` only appears as the buying side of an in-scope `manage_sell_offer` should still export an asset row for `Y`, because the command has already decided that this offer type participates in `history_assets` discovery.

## Mechanism

The input readers admit the entire `manage_sell_offer` operation, but `TransformAsset()` narrows that operation to `opSellOf.Selling` and never emits `opSellOf.Buying`. As a result, an asset that enters the selected range only as the quote/buying side of included sell-offer activity is silently omitted from `history_assets`, even though the same operation would export the base/selling asset from the identical ledger event.

## Trigger

1. Find or construct a ledger range where a successful `manage_sell_offer` references assets `X` and `Y`, and `Y` does not appear anywhere else in-range as a payment asset or as the selling side of another included offer.
2. Run `stellar-etl export_assets --start-ledger <S> --end-ledger <E> -o /tmp/assets.json`.
3. Inspect the exported asset rows.
4. The command will export `X` from the offer but omit `Y`, even though both assets were referenced by the included operation.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:40-49` — treats the full `manage_sell_offer` operation as an in-scope asset-discovery input.
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:30-39` — repeats the same offer admission on the history-archive path.
- `internal/transform/asset.go:17-38` — `ManageSellOffer` branch extracts only `opSellOf.Selling` and never visits `opSellOf.Buying`.
- `cmd/export_assets.go:44-69` — writes exactly one transformed asset row per admitted operation before deduplicating by `asset_id`.

## Evidence

The transform layer is one-sided by construction: the `ManageSellOffer` arm binds one local `asset` variable and assigns only the selling asset into it. There is no sibling append or second transform call for the buying asset anywhere in `export_assets`, so the omission persists all the way to output.

## Anti-Evidence

The overall `export_assets` command is intentionally narrow and not an exhaustive asset inventory, so a reviewer may argue that the exact offer-side semantics are part of that undocumented scope. But the code has already crossed the boundary from pure payment-based discovery into offer-based discovery by admitting `manage_sell_offer`; once that choice is made, exporting only one leg of the included offer is an internal inconsistency that can change which asset rows appear for legitimate market activity.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full export_assets pipeline from input filtering through transform to output. Both `GetPaymentOperations` (assets.go:42) and `GetPaymentOperationsHistoryArchive` (assets_history_archive.go:32) admit `OperationTypeManageSellOffer` into the processing scope alongside `OperationTypePayment`. However, `TransformAsset` (asset.go:24-29) extracts only `opSellOf.Selling` from the ManageSellOffer body — there is no code path for `opSellOf.Buying`. The export command (export_assets.go:44-69) calls `TransformAsset` once per operation and writes one row, with no mechanism to emit multiple assets per operation.

### Code Paths Examined

- `internal/input/assets.go:42` — `op.Body.Type == xdr.OperationTypeManageSellOffer` admits the full operation
- `internal/input/assets_history_archive.go:32` — identical filter on the history-archive path
- `internal/transform/asset.go:24-29` — `case xdr.OperationTypeManageSellOffer:` extracts only `opSellOf.Selling`, `opSellOf.Buying` is never read
- `internal/transform/asset.go:40-52` — `transformSingleAsset(asset)` processes only the single selling asset
- `cmd/export_assets.go:44-69` — loop calls `TransformAsset` once per admitted operation, producing one row; `seenIDs` dedup is correct but irrelevant to the omission
- `internal/transform/asset_test.go:74-97` — tests only cover Payment operations, no ManageSellOffer test coverage

### Findings

The asymmetry is confirmed by direct code reading. The ManageSellOffer operation carries two assets (`Selling` and `Buying`), but only `Selling` is extracted. The buying asset is silently dropped. This means an asset appearing exclusively as the buying/quote side of ManageSellOffer operations within a given ledger range will produce no row in the export.

Severity is downgraded from High to Medium because:
1. The `export_assets` command is documented as intentionally narrow ("Exports the assets that are created from payment operations" — README:244, cmd/export_assets.go:17)
2. The function is named `GetPaymentOperations`, framing the scope around "payment-like" asset extraction
3. The selling asset is the "outgoing/payment" analog for offers, so there is a plausible (though undocumented) design rationale
4. The practical impact requires a specific scenario: an asset appearing ONLY as the buying side of ManageSellOffer in the entire exported range

Despite these mitigating factors, the inconsistency is real: the code admits the operation but only processes half its data. This is a conditional row omission, not a design choice that's been explicitly justified anywhere.

### PoC Guidance

- **Test file**: `internal/transform/asset_test.go`
- **Setup**: Construct a `ManageSellOfferOp` with a known `Selling` asset (e.g., USDT) and a known `Buying` asset (e.g., EUR credit). Wrap it in an `xdr.Operation` with `OperationTypeManageSellOffer`.
- **Steps**: Call `TransformAsset()` with the constructed operation. Verify it returns only the selling asset. Then demonstrate that no call path in the export pipeline ever processes the buying asset — there is no second call to `TransformAsset` or `transformSingleAsset` for the buying side.
- **Assertion**: Assert that `TransformAsset` returns only USDT (the selling asset) and that EUR (the buying asset) is not exported anywhere. This demonstrates the asymmetric extraction.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestManageSellOfferOmitsBuyingAsset"
**Test Language**: Go

### Demonstration

The test constructs a ManageSellOffer operation with USDT as the selling asset and ETH as the buying asset, then calls `TransformAsset()`. The function returns only a single `AssetOutput` containing USDT — the buying asset ETH is completely absent from the return value. Since the export pipeline calls `TransformAsset` exactly once per admitted operation and the return type is a single `(AssetOutput, error)`, there is no mechanism to ever export the buying side. This confirms that an asset appearing exclusively as the buying/quote side of ManageSellOffer operations will be silently omitted from the export.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestManageSellOfferOmitsBuyingAsset demonstrates that TransformAsset only
// extracts the Selling asset from a ManageSellOffer operation and silently
// drops the Buying asset. An asset appearing exclusively as the buying/quote
// side of ManageSellOffer operations will never be exported by export_assets.
func TestManageSellOfferOmitsBuyingAsset(t *testing.T) {
	// Construct a ManageSellOffer with known Selling (USDT) and Buying (ETH) assets
	sellOp := xdr.ManageSellOfferOp{
		Selling: usdtAsset,
		Buying:  ethAsset,
		Amount:  100000000,
		Price: xdr.Price{
			N: 1,
			D: 2,
		},
		OfferId: 0,
	}

	op := xdr.Operation{
		SourceAccount: nil,
		Body: xdr.OperationBody{
			Type:             xdr.OperationTypeManageSellOffer,
			ManageSellOfferOp: &sellOp,
		},
	}

	// Call TransformAsset — this is the only call the export pipeline makes per operation
	output, err := TransformAsset(op, 0, 0, 0, genericLedgerCloseMeta)
	if err != nil {
		t.Fatalf("TransformAsset returned unexpected error: %v", err)
	}

	// The function returns ONLY the selling asset (USDT)
	if output.AssetCode != "USDT" {
		t.Errorf("expected selling asset USDT, got %q", output.AssetCode)
	}

	// TransformAsset returns a single AssetOutput — there is no mechanism to
	// return both assets. The buying asset (ETH) is completely lost.
	// Demonstrate: call TransformAsset again with the same op — it still returns USDT only.
	output2, err := TransformAsset(op, 0, 0, 0, genericLedgerCloseMeta)
	if err != nil {
		t.Fatalf("second TransformAsset call returned unexpected error: %v", err)
	}

	// Both calls produce USDT, never ETH — the buying asset is irrecoverably omitted
	if output2.AssetCode != "USDT" {
		t.Errorf("expected selling asset USDT on second call, got %q", output2.AssetCode)
	}

	// The core assertion: TransformAsset's return type is (AssetOutput, error),
	// a single asset. There is no way to get ETH out of this function for a
	// ManageSellOffer. The buying asset code "ETH" never appears in the output.
	if output.AssetCode == "ETH" {
		t.Error("TransformAsset unexpectedly returned the buying asset — hypothesis disproven")
	}

	// Verify the buying asset (ETH) details to confirm it exists in the operation
	// but is not in the output
	var buyingAssetType, buyingAssetCode, buyingAssetIssuer string
	err = sellOp.Buying.Extract(&buyingAssetType, &buyingAssetCode, &buyingAssetIssuer)
	if err != nil {
		t.Fatalf("failed to extract buying asset: %v", err)
	}
	if buyingAssetCode != "ETH" {
		t.Fatalf("test setup error: expected buying asset ETH, got %q", buyingAssetCode)
	}

	t.Logf("ManageSellOffer contains Selling=USDT and Buying=ETH")
	t.Logf("TransformAsset returned: code=%q issuer=%q type=%q", output.AssetCode, output.AssetIssuer, output.AssetType)
	t.Logf("Buying asset (code=%q issuer=%q type=%q) is silently dropped — never exported",
		buyingAssetCode, buyingAssetIssuer, buyingAssetType)
	t.Logf("BUG CONFIRMED: export_assets admits ManageSellOffer but only extracts the Selling side")
}
```

### Test Output

```
=== RUN   TestManageSellOfferOmitsBuyingAsset
    data_integrity_poc_test.go:76: ManageSellOffer contains Selling=USDT and Buying=ETH
    data_integrity_poc_test.go:77: TransformAsset returned: code="USDT" issuer="GBVVRXLMNCJQW3IDDXC3X6XCH35B5Q7QXNMMFPENSOGUPQO7WO7HGZPA" type="credit_alphanum4"
    data_integrity_poc_test.go:78: Buying asset (code="ETH" issuer="GBT4YAEGJQ5YSFUMNKX6BPBUOCPNAIOFAVZOF6MIME2CECBMEIUXFZZN" type="credit_alphanum4") is silently dropped — never exported
    data_integrity_poc_test.go:80: BUG CONFIRMED: export_assets admits ManageSellOffer but only extracts the Selling side
--- PASS: TestManageSellOfferOmitsBuyingAsset (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.935s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-12
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Exercises claimed issue?** No. The PoC only shows that `TransformAsset()` returns the selling asset for a `ManageSellOffer`. It does not show that `export_assets` is contractually supposed to emit both assets from that operation.
2. **Realistic preconditions?** Yes. A real `manage_sell_offer` always references both a selling and buying asset.
3. **Bug or by design?** By design. The command, reader names, reader comments, transformer comment, and README all frame `export_assets` as a narrow asset-reference export derived from selected operations, not as a complete catalog of every asset mentioned by an operation. The implementation is consistently one-output-per-operation.
4. **Impact/severity match?** Not applicable. Because the behavior is contractual, there is no confirmed corruption to score.
5. **In scope?** The code path is in scope, but the reported "bug" is not: it is a feature-expansion argument about what `export_assets` ought to include.
6. **Is the test correct?** No. It passes by construction because `TransformAsset` has signature `(AssetOutput, error)`. Re-calling the same function and observing the same selling asset does not prove silent corruption; it only restates the current API shape.
7. **Alternative explanation?** Yes. `export_assets` intentionally records one representative asset per qualifying operation, and for `ManageSellOffer` that representative is the selling side.
8. **Novelty?** Likely novel, but invalid.

### Rejection Reason

The PoC proves an implementation detail, not a contract violation. `export_assets` is intentionally narrow and currently models `ManageSellOffer` as contributing its selling asset only; there is no evidence in source, tests, or documentation that the buying side must also produce a `history_assets` row.

### Failed Checks

- 1
- 3
- 6
