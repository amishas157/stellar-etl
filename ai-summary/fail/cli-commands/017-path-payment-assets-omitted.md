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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the complete `export_assets` pipeline from CLI entry (`cmd/export_assets.go`) through both input readers (`internal/input/assets.go:42`, `internal/input/assets_history_archive.go:32`) to the transformer (`internal/transform/asset.go:18`). All three layers consistently filter for exactly `OperationTypePayment` and `OperationTypeManageSellOffer`. The README explicitly documents this command as: "Exports the assets that are created from payment operations over a specified ledger range." The three-layer filter is deliberate and consistent with the documented scope.

### Code Paths Examined

- `cmd/export_assets.go:17-83` — CLI entry point; calls `GetPaymentOperations` or `GetPaymentOperationsHistoryArchive`, then `TransformAsset` on each result
- `internal/input/assets.go:42` — Filter: `op.Body.Type == xdr.OperationTypePayment || op.Body.Type == xdr.OperationTypeManageSellOffer`
- `internal/input/assets_history_archive.go:32` — Identical filter to the datastore-backed reader
- `internal/transform/asset.go:18` — Guard: `if opType != xdr.OperationTypePayment && opType != xdr.OperationTypeManageSellOffer { return error }`
- `README.md:236-245` — Documents: "Exports the assets that are created from payment operations"

### Why It Failed

This is **working-as-designed behavior**, not a data corruption bug. The hypothesis treats a documented feature limitation as a defect, but the code does exactly what its documentation describes:

1. **Intentional scope**: The command, its helper function (`GetPaymentOperations`), and the transformer all consistently scope to `Payment` and `ManageSellOffer` — this three-layer consistency is a strong signal of deliberate design, not an accidental omission.
2. **Documented behavior**: Both the README and the Cobra `Long` description explicitly state the command operates on "payment operations." The output matches the spec.
3. **Broader pattern**: The filter also excludes `ManageBuyOffer`, `CreatePassiveSellOffer`, `ChangeTrust`, and many other operation types that reference assets. If the exclusion of path payments were a bug, these other exclusions would be bugs too — but the command was never designed for comprehensive asset discovery across all operation types.

### Lesson Learned

When `export_assets` consistently documents and implements a two-operation-type scope across all layers (CLI docs, input readers, transformer), the scope is intentional. A feature request to broaden coverage is not a data correctness bug. Look for cases where documented behavior diverges from code behavior, not where both agree on a limited scope.
