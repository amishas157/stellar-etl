# H003: Asset export treats impossible ranges as empty asset sets

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_assets` should reject bounded requests like `--start-ledger 100 --end-ledger 50` or `--end-ledger 0`, because those ranges cannot correspond to a valid Stellar ledger interval and an empty asset file is materially different from a failed export.

## Mechanism

On the default path, `export_assets` calls `input.GetPaymentOperations()`, which prepares a bounded datastore range but never validates the start/end pair. When `seq := start; seq <= end` is false at entry, the helper returns an empty operation slice, and `export_assets` then emits a zero-row asset file as if the requested interval legitimately contained no relevant operations.

## Trigger

1. Run `stellar-etl export_assets --start-ledger 100 --end-ledger 50 -o assets.json`.
2. Because no validation fires before the loop, the command may succeed with an empty asset export instead of rejecting the impossible range.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:21-60` — missing `ValidateLedgerRange()` before iterating the bounded range
- `cmd/export_assets.go:Run:17-82` — accepts an empty `paymentOps` slice and produces a zero-row asset export
- `internal/utils/main.go:735-758` — contains the validation helper that this path currently skips

## Evidence

The asset reader follows the same datastore-backed shape as `GetLedgers()` but lacks the corresponding history-archive validation guard. `export_assets` then deduplicates and writes rows only if `paymentOps` is non-empty, so an impossible range becomes indistinguishable from “no new assets in this interval”.

## Anti-Evidence

When `--captive-core` is enabled, `export_assets` switches to `GetPaymentOperationsHistoryArchive()`, which likely inherits the validating `CreateBackend()` path instead of the silent datastore path. That suggests the bug may be configuration-specific rather than universal across all asset exports.
