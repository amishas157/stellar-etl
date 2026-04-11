# H001: `export_assets` omits assets referenced only by path-payment operations

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_assets` is documented and named around payment-operation asset discovery, so a ledger range containing `PathPaymentStrictReceive` or `PathPaymentStrictSend` operations should still export the assets referenced by those payment operations. If an asset first appears in a path payment's `send_asset`, `dest_asset`, or intermediate `path`, the command should emit a `history_assets` row for that asset instead of silently dropping it.

## Mechanism

Both asset readers hard-code a two-type filter of `Payment` and `ManageSellOffer`, and `TransformAsset()` rejects every other operation type. That means path-payment operations never even enter the asset transform path, despite carrying explicit asset references in XDR. Downstream `history_assets` consumers can therefore get an incomplete asset dimension whenever an asset appears only in path payments within the scanned range.

## Trigger

Run `export_assets` over a ledger range where an asset appears only inside `PathPaymentStrictReceive` or `PathPaymentStrictSend` operations and does not also appear in a plain `Payment` or `ManageSellOffer` in the same range. The correct output should contain that asset; the current command will omit it entirely.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:40-49` — datastore-backed asset reader only admits `Payment` and `ManageSellOffer`
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:30-39` — history-archive reader repeats the same two-type filter
- `internal/transform/asset.go:13-38` — transformer rejects every op type except `Payment` and `ManageSellOffer`
- `README.md:236-245` — command is presented as exporting assets from payment operations
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:25898-25905` — `PathPaymentStrictReceiveOp` carries `SendAsset`, `DestAsset`, and `Path`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:26040-26046` — `PathPaymentStrictSendOp` carries `SendAsset`, `DestAsset`, and `Path`

## Evidence

The asset-reader helper is literally named `GetPaymentOperations`, but its filter excludes both path-payment op types. The transformer mirrors that restriction, so there is no later stage that could recover assets referenced only by those payment operations.

## Anti-Evidence

If the same asset also appears in a plain `Payment` or `ManageSellOffer` elsewhere in the requested range, the omission is masked because the command can still emit one row for that asset from the other operation type.
