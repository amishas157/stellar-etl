# H001: Shared `--captive-core` flag selects the wrong backend in archive-style exports

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a user sets the shared `--captive-core` flag, commands should route through the captive-core / `CreateLedgerBackend` path that `internal/utils` advertises. The exported rows should therefore be built from full `LedgerCloseMeta` input, preserving metadata such as real ledger close times, ledger sequence numbers, and Soroban-only ledger fields.

## Mechanism

`utils.AddCommonFlags()` documents `--captive-core` as "run captive core to retrieve data", and most readers honor that by calling `utils.CreateLedgerBackend(..., useCaptiveCore, ...)`. But `export_assets` and `export_ledgers` invert the meaning of the shared flag: when `UseCaptiveCore` is true they bypass `CreateLedgerBackend` and call history-archive helpers instead, which do not provide the same `LedgerCloseMeta` payload. That means the flag intended to request richer ledger metadata instead selects the poorer input path, producing partial or failed exports.

## Trigger

Run either of these commands with `--captive-core` over a range that contains Soroban ledger metadata or asset-issuing payment / manage-offer operations:

1. `stellar-etl export_ledgers --captive-core -s <start> -e <end>`
2. `stellar-etl export_assets --captive-core -s <start> -e <end>`

Compare the behavior to the default path without `--captive-core`.

## Target Code

- `internal/utils/main.go:AddCommonFlags:231-246` - defines `--captive-core` as the shared "run captive core" selector
- `internal/utils/main.go:MustCommonFlags:460-537` - returns the shared `UseCaptiveCore` value unchanged
- `cmd/export_ledgers.go:28-31` - routes `UseCaptiveCore=true` to `input.GetLedgersHistoryArchive(...)`
- `cmd/export_assets.go:30-33` - routes `UseCaptiveCore=true` to `input.GetPaymentOperationsHistoryArchive(...)`
- `internal/input/ledgers.go:16-25` - the normal path uses `utils.CreateLedgerBackend(...)` and receives real `LedgerCloseMeta`
- `internal/input/assets.go:23-33` - the normal path uses `utils.CreateLedgerBackend(...)` and receives real `LedgerCloseMeta`

## Evidence

The shared flag contract is established in `internal/utils`, but these two commands are the only call sites that branch the opposite way. Their `UseCaptiveCore=true` branch does not invoke `CreateCaptiveCoreBackend` or even `CreateLedgerBackend`; instead it selects history-archive readers that either omit `LCM` entirely or return only history-archive structures.

## Anti-Evidence

If users leave `--captive-core` unset, or use commands that consistently call `CreateLedgerBackend`, the shared flag contract is not violated. The corruption is therefore limited to the two archive-style exports that special-case the flag today.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the flag from `AddCommonFlags` (registers `--captive-core`) through `MustCommonFlags` (reads into `UseCaptiveCore`) into the two anomalous commands. In `export_ledgers.go:28-31`, `UseCaptiveCore=true` routes to `GetLedgersHistoryArchive()` which calls `utils.CreateBackend()` (history archive reader, NOT `CreateLedgerBackend`), returning `HistoryArchiveLedgerAndLCM` with a zero-valued `LCM` field. In `export_assets.go:30-33`, the same flag routes to `GetPaymentOperationsHistoryArchive()` which explicitly sets `LedgerCloseMeta: xdr.LedgerCloseMeta{}`. All 8 other export commands pass `UseCaptiveCore` through to `CreateLedgerBackend()` which correctly dispatches to either captive core or DataStore. These two commands invert the flag, routing to a poorer data source that silently zeroes Soroban-era fields.

### Code Paths Examined

- `cmd/export_ledgers.go:28-31` — `if commonArgs.UseCaptiveCore { GetLedgersHistoryArchive(...) } else { GetLedgers(...) }` — INVERTED: captive-core flag routes to history archive
- `cmd/export_assets.go:30-33` — `if commonArgs.UseCaptiveCore { GetPaymentOperationsHistoryArchive(...) } else { GetPaymentOperations(...) }` — INVERTED: same pattern
- `internal/input/ledgers_history_archive.go:11-35` — calls `utils.CreateBackend(start, end, env.ArchiveURLs)` (history archive); returns `HistoryArchiveLedgerAndLCM{Ledger: ledger}` with LCM zero-valued (line 24-26)
- `internal/input/assets_history_archive.go:13-51` — calls `utils.CreateBackend()`; explicitly sets `LedgerCloseMeta: xdr.LedgerCloseMeta{}` with comment "Using historyArchive will not support getting LCM" (line 38)
- `internal/input/ledgers.go:14-91` — calls `utils.CreateLedgerBackend(ctx, useCaptiveCore, env)` which correctly dispatches; returns both `Ledger` AND `LCM` populated (line 79-82)
- `internal/transform/ledger.go:65-91` — `lcm.GetV1()` and `lcm.GetV2()` on zero LCM return `ok=false`, zeroing `SorobanFeeWrite1Kb`, `TotalByteSizeOfLiveSorobanState`, `EvictedKeysHash`, `EvictedKeysType`
- `internal/transform/asset.go:45-50` — `utils.GetCloseTime(lcm)` and `utils.GetLedgerSequence(lcm)` on zero LCM return epoch time (1970-01-01) and sequence 0
- `internal/utils/main.go:968-976` — `GetCloseTime` and `GetLedgerSequence` call `lcm.LedgerHeaderHistoryEntry()` which for V=0 returns zero-valued header
- `cmd/export_token_transfers.go:28` — contrast: calls `GetLedgers(..., commonArgs.UseCaptiveCore)` unconditionally (correct pattern)
- `cmd/export_transactions.go:25` — contrast: calls `GetTransactions(..., commonArgs.UseCaptiveCore)` (correct pattern)

### Findings

**The flag routing is inverted in exactly 2 of 10 export commands.** When `--captive-core` is set:

1. **`export_ledgers`**: Routes to `GetLedgersHistoryArchive()` → `utils.CreateBackend()` (history archive). The returned `HistoryArchiveLedgerAndLCM` has a zero-valued `LCM`. `TransformLedger` receives this zero LCM, causing all Soroban fields to be silently zeroed: `SorobanFeeWrite1Kb=0`, `TotalByteSizeOfLiveSorobanState=0`, `EvictedKeysHash=nil`, `EvictedKeysType=nil`. For Protocol 20+ ledgers, these should contain real values.

2. **`export_assets`**: Routes to `GetPaymentOperationsHistoryArchive()` → `utils.CreateBackend()`. The code explicitly assigns `LedgerCloseMeta: xdr.LedgerCloseMeta{}` (with a comment acknowledging the limitation). `TransformAsset` calls `GetCloseTime(zero LCM)` → returns epoch time (1970-01-01T00:00:00Z), and `GetLedgerSequence(zero LCM)` → returns 0. Every exported asset gets `ClosedAt=epoch` and `LedgerSequence=0`.

**All 8 other export commands** (`export_transactions`, `export_operations`, `export_effects`, `export_trades`, `export_contract_events`, `export_token_transfers`, `export_ledger_transaction`, `export_ledger_entry_changes`) pass `UseCaptiveCore` through to `CreateLedgerBackend()` which correctly dispatches to captive core or DataStore. The inversion is specific to `export_ledgers` and `export_assets`.

**Note**: The `--captive-core` flag is deprecated ("Will be removed in the Protocol 23 update") and the default DataStore path works correctly. However, when the deprecated flag IS used, the corruption is silent — no error, no warning, valid-looking output with wrong values.

### PoC Guidance

- **Test file**: `internal/input/ledgers_history_archive_test.go` (create new, or append to existing test in `cmd/` if more appropriate)
- **Setup**: Construct a mock scenario where `UseCaptiveCore=true` and compare the output of `GetLedgersHistoryArchive` vs `GetLedgers` for the same ledger range. Alternatively, verify the flag routing directly.
- **Steps**:
  1. Verify `export_ledgers.go:28` branches to `GetLedgersHistoryArchive` when `UseCaptiveCore=true`
  2. Verify `GetLedgersHistoryArchive` returns `HistoryArchiveLedgerAndLCM` with zero-valued `LCM`
  3. Call `TransformLedger` with that zero LCM and a Soroban-era ledger header
  4. Assert Soroban fields are zeroed (demonstrating the corruption)
  5. Similarly for `export_assets.go:30`: verify `GetPaymentOperationsHistoryArchive` returns empty `LedgerCloseMeta`
  6. Call `TransformAsset` with the empty LCM and verify `ClosedAt` and `LedgerSequence` are wrong
- **Assertion**: `TransformLedger` output with zero LCM should have `SorobanFeeWrite1Kb=0` even for a ledger that should have non-zero values; `TransformAsset` output should have `ClosedAt=time.Unix(0,0)` and `LedgerSequence=0`
