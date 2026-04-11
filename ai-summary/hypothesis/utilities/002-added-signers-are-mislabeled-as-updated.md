# H002: Newly added signer rows are mislabeled as `UPDATED`

**Date**: 2026-04-10
**Subsystem**: utilities / data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an existing account adds a new signer, the newly introduced `account_signers` row should be exported with `ledger_entry_change = CREATED` and `deleted = false`. The exporter should distinguish a first appearance of a signer from a modification of a signer that already existed.

## Mechanism

`TransformSigners()` does not diff for post-only signers at all. Every signer present in `Post` is emitted in the first pass with the parent account's `LedgerEntryUpdated` change type, and the diff branch only adds synthetic removals for pre-only signers. That means newly added signers are serialized as ordinary updates even though there is no pre-state signer row to update.

## Trigger

Process an `AccountEntry` `LedgerEntryUpdated` change where `Pre` lacks signer `S` and `Post` includes signer `S` (for example a `SetOptions` operation that adds a new signer to an existing account). The correct signer row should be a creation, but the current output marks it as an update.

## Target Code

- `internal/transform/account_signer.go:35-52` — emits all post-state signers with `LedgerEntryChange: uint32(changeType)`
- `internal/transform/account_signer.go:57-86` — update-specific diff only handles `Pre`-only signers and never upgrades `Post`-only signers to `CREATED`

## Evidence

For update changes, `changeType` is always `LedgerEntryUpdated`, so any signer found only in `Post` still leaves the first loop with `ledger_entry_change = 1`. There is no complementary branch that checks `if _, existed := preSigners[signer]; !existed` and rewrites the row to `CREATED`.

## Anti-Evidence

Some downstream loaders may treat `CREATED` and `UPDATED` identically as upserts, which would reduce operational impact. But the export still advertises the wrong row-level change kind, and that corruption matters for consumers that analyze signer lifecycle events or count newly added signers per ledger.
