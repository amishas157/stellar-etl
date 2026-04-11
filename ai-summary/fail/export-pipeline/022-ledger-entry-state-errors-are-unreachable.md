# H022: `export_ledger_entry_changes` silently drops tracked `STATE` rows through ignored compactor errors

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the ledger-entry-changes export encounters tracked ledger-entry `STATE` changes,
it should either export them correctly or fail loudly. A valid close-meta stream
should not lose tracked account / trustline / config-setting rows just because the
input passed through the compactor.

## Mechanism

`extractBatch()` calls `cache.AddChange(change)` and ignores the returned error.
Because upstream `ChangeCompactor.AddChange()` rejects unknown change states, I
initially suspected that a tracked `LedgerEntryState` row could be dropped
silently during `export_ledger_entry_changes`.

## Trigger

Run `export_ledger_entry_changes` on a ledger whose raw close meta contains
tracked `LedgerEntryState` entries and inspect whether those rows disappear from
the exported batch.

## Target Code

- `internal/input/changes.go:117-145` — `extractBatch()` ignores `cache.AddChange(change)` errors
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor.go:97-109` — `AddChange()` rejects unsupported change states
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change.go:194-195` — `GetChangesFromLedgerEntryChanges()` skips `LedgerEntryState` entries before they become `Change` values
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_change_reader.go:172-200` — `LedgerChangeReader` only appends the normalized `Get*Changes()` outputs into the ETL stream

## Evidence

The call site in `extractBatch()` really does discard the `AddChange()` error,
and the upstream compactor really does return `"Unknown entry change state"` for
unsupported `ChangeType` values. On a first pass, that looks like a silent data
loss path for tracked ledger-entry rows.

## Anti-Evidence

The live export path never feeds raw `LedgerEntryState` items into the compactor.
`LedgerChangeReader` populates its stream from `tx.GetFeeChanges()`,
`tx.GetChanges()`, `tx.GetPostApplyFeeChanges()`, and upgrade-change helpers, and
all of those normalize through `GetChangesFromLedgerEntryChanges()`, which
explicitly `continue`s past `LedgerEntryState` entries before any `Change` is
emitted.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspicious ignored-error site is real, but the current export pipeline does
not deliver `LedgerEntryState` rows to it. Upstream normalization strips those
entries before `extractBatch()` ever sees a `Change`, so the feared silent-drop
path is unreachable on the live `export_ledger_entry_changes` flow.

### Lesson Learned

When a compactor rejects some low-level change kind, first verify whether the
reader feeding it emits that kind at all. In this case, the ETL consumes the
SDK's already-normalized `Change` stream rather than raw `LedgerEntryChanges`,
which makes the ignored `AddChange()` error much less dangerous than it first
appears.
