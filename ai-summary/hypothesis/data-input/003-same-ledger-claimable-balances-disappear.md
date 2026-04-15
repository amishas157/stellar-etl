# H003: Same-ledger create-and-claim claimable balances vanish from change export

**Date**: 2026-04-15
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a claimable balance is created and later claimed within the same ledger, `export_ledger_entry_changes --export-balances` should still emit a `claimable_balances` row describing the removed balance object. The exported schema already has `deleted` and `ledger_entry_change` fields, so downstream consumers should be able to see that the balance existed and was removed in that ledger instead of observing no row at all.

## Mechanism

`extractBatch()` feeds claimable-balance changes through `ChangeCompactor` before export. In the compactor's `CREATED -> REMOVED` transition, the cache entry is deleted as a no-op; that erases balances that were created and then claimed later in the same ledger, so `export_ledger_entry_changes` never calls `TransformClaimableBalance()` for them even though the transformer can represent removed balances correctly.

## Trigger

1. Find a ledger where one transaction creates a claimable balance and a later transaction in the same ledger claims it.
2. Run `stellar-etl export_ledger_entry_changes --export-balances --start-ledger <L> --end-ledger <L>`.
3. The correct export should include a removed `claimable_balances` row for that balance ID.
4. The current path can emit no row for the balance at all because the create/remove pair is compacted away before transformation.

## Target Code

- `internal/input/changes.go:102-149` — claimable-balance changes are compacted before export
- `cmd/export_ledger_entry_changes.go:160-172` — exports only the compacted claimable-balance slice
- `internal/transform/claimable_balance.go:24-76` — transformer already supports removed balances via `Deleted` and `LedgerEntryChange`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor.go:233-236` — `CREATED -> REMOVED` is treated as a no-op and deleted from the cache
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor.go:10-22` — compactor is explicitly optimized for final-state DB updates, not event-preserving exports

## Evidence

The export command reads only `changes.Changes` after compaction, and the compactor explicitly drops entries created and removed in the same ledger. `TransformClaimableBalance()` is not the limiting factor: it can encode removed balances using the pre-change ledger entry plus `deleted=true`, which shows the omission happens earlier in the input layer.

## Anti-Evidence

If `claimable_balances` under `export_ledger_entry_changes` is intended to model only end-of-ledger state deltas, suppressing transient same-ledger balances could be defended as intentional. The command README says it exports ledger-entry changes, not just surviving ledger-close state, so silently erasing a real create-and-claim sequence still looks like a meaningful contract mismatch.
