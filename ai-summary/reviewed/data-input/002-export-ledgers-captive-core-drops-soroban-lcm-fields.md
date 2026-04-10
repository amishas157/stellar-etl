# H002: export_ledgers Captive-Core Path Drops Soroban LCM Fields

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_ledgers` should populate Soroban-derived ledger fields such as `soroban_fee_write_1kb`, `total_byte_size_of_live_soroban_state`, `evicted_ledger_keys_type`, and `evicted_ledger_keys_hash` from the ledger's close meta whenever those values exist on-chain, regardless of backend selection.

## Mechanism

`cmd/export_ledgers.go` also inverts the `UseCaptiveCore` branch and calls `GetLedgersHistoryArchive()` when captive core is requested. That reader returns `HistoryArchiveLedgerAndLCM` values with only `Ledger` populated; `LCM` remains zero-valued, and `TransformLedger()` only fills the Soroban-specific columns by reading `lcm.GetV1()` / `lcm.GetV2()`, so those columns silently become `0` or empty slices even for ledgers whose real close meta contains non-zero values.

## Trigger

Run `stellar-etl export_ledgers --use-captive-core --start-ledger <L> --end-ledger <R>` over a Soroban-heavy range where the real `LedgerCloseMeta` includes `SorobanFeeWrite1Kb`, live-state byte size, or evicted keys. Compare the JSON/Parquet output with the same range exported without `--use-captive-core` or with the raw close meta.

## Target Code

- `cmd/export_ledgers.go:28-43` — selects `GetLedgersHistoryArchive()` for captive-core exports and passes the returned `LCM` into `TransformLedger`
- `internal/input/ledgers_history_archive.go:10-28` — constructs `HistoryArchiveLedgerAndLCM` without setting the `LCM` field
- `internal/utils/main.go:1122-1125` — defines the wrapper struct whose zero-value `LCM` flows downstream
- `internal/transform/ledger.go:61-91` — populates Soroban ledger fields only from `lcm.GetV1()` / `lcm.GetV2()`

## Evidence

The history-archive reader intentionally omits `LCM`, while the normal `GetLedgers()` path preserves it from `backend.GetLedger()`. `TransformLedger()` gets most basic ledger fields from `historyarchive.Ledger`, but every Soroban-specific field is gated on a non-empty `LedgerCloseMeta`, so the branch inversion creates a partial row that looks valid except the Soroban columns are zeroed out.

## Anti-Evidence

Core header-derived fields like sequence, close time, transaction counts, and hashes still come from `historyarchive.Ledger`, so the exported row is not obviously broken. The corruption is concentrated in the close-meta-derived Soroban columns.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. A sibling hypothesis (fail/004 — export_assets captive-core zeroes metadata) covers the same branching pattern in export_assets but was rejected because that path panics rather than silently corrupting. This ledger hypothesis is distinct because TransformLedger uses safe GetV1()/GetV2() accessors that silently return zero instead of panicking.

### Trace Summary

The branch in `cmd/export_ledgers.go:28-31` routes `UseCaptiveCore=true` to `GetLedgersHistoryArchive()`, which uses `utils.CreateBackend()` (a history archive HTTP client) rather than `utils.CreateLedgerBackend()` (captive core or GCS datastore). The history archive reader at `ledgers_history_archive.go:24-26` constructs `HistoryArchiveLedgerAndLCM{Ledger: ledger}` without setting the `LCM` field, leaving it as a zero-value `xdr.LedgerCloseMeta{V: 0, V0: nil, V1: nil, V2: nil}`. In `TransformLedger()` at `ledger.go:65-91`, `lcm.GetV1()` and `lcm.GetV2()` both check the discriminant via `ArmForSwitch(0)` which returns arm "V0", so neither "V1" nor "V2" matches — both return `ok=false` without panicking. All Soroban fields remain at Go zero values.

### Code Paths Examined

- `cmd/export_ledgers.go:28-31` — branch condition routes `UseCaptiveCore=true` to history archive reader (not captive core); other commands like `export_operations`, `export_transactions` do NOT branch and pass `UseCaptiveCore` directly to `CreateLedgerBackend`
- `internal/input/ledgers_history_archive.go:24-26` — `HistoryArchiveLedgerAndLCM{Ledger: ledger}` leaves `LCM` as zero value
- `internal/input/ledgers.go:79-82` — by contrast, `GetLedgers()` explicitly sets both `Ledger` and `LCM` fields from the ledger backend
- `internal/transform/ledger.go:65-77` — `lcm.GetV1()` returns `ok=false` on zero-value LCM (V=0, arm="V0" ≠ "V1"); Soroban fields `SorobanFeeWrite1Kb`, `TotalByteSizeOfLiveSorobanState`, `EvictedKeys` all remain zero
- `internal/transform/ledger.go:79-91` — `lcm.GetV2()` similarly returns `ok=false`; same fields remain zero
- `go-stellar-sdk/xdr/xdr_generated.go:20160-20168` — `GetV1()` uses `ArmForSwitch(int32(u.V))`, returns early without dereference when arm ≠ "V1" — confirms NO panic, just silent skip
- `internal/utils/main.go:238` — flag `captive-core` is documented as "(Deprecated; Will be removed in the Protocol 23 update)"
- `internal/utils/main.go:1011-1018` — `CreateLedgerBackend(ctx, true, env)` correctly creates a captive core backend that would provide full V1/V2 LCM, but this path is never reached by `export_ledgers` when `UseCaptiveCore=true`

### Findings

1. **Silent zeroing confirmed**: When `--captive-core` is set, `export_ledgers` produces rows where `soroban_fee_write_1kb=0`, `total_byte_size_of_live_soroban_state=0`, `evicted_ledger_keys_type=[]`, `evicted_ledger_keys_hash=[]` regardless of actual on-chain values. All other fields (sequence, close_time, tx counts, hashes, fees) are correct, making the output appear valid.

2. **Branch is functionally inverted**: `UseCaptiveCore=true` routes to the history archive (no LCM), while `UseCaptiveCore=false` routes to the datastore backend (has LCM). Contrast with 8 other export commands that pass `UseCaptiveCore` straight to `CreateLedgerBackend()`, which correctly dispatches to captive core or datastore.

3. **Deprecated but not dead**: The `--captive-core` flag is deprecated but still functional. Any user running with this flag on a Soroban-era ledger range gets silently corrupted Soroban columns in their ledger export.

4. **Distinct from export_assets (fail/004)**: The export_assets sibling hypothesis was rejected because `TransformAsset` calls `lcm.LedgerHeaderHistoryEntry()` → `MustV0()` which panics on nil V0. Here, `TransformLedger` uses `GetV1()`/`GetV2()` which safely return `ok=false` — producing silent corruption, not a crash.

### PoC Guidance

- **Test file**: `internal/transform/ledger_test.go` (append a new test case)
- **Setup**: Construct a `historyarchive.Ledger` with a valid header and a `xdr.LedgerCloseMeta` with V=1 containing non-zero `SorobanFeeWrite1Kb` and `TotalByteSizeOfLiveSorobanState`. Then construct a second call with the same ledger but a zero-value `xdr.LedgerCloseMeta{}`.
- **Steps**: Call `TransformLedger(ledger, fullLCM)` and `TransformLedger(ledger, zeroLCM)`. Compare the Soroban fields.
- **Assertion**: Assert that `TransformLedger(ledger, zeroLCM).SorobanFeeWrite1Kb == 0` while `TransformLedger(ledger, fullLCM).SorobanFeeWrite1Kb != 0`, demonstrating that the zero-value LCM silently drops Soroban data. Additionally, verify that `GetLedgersHistoryArchive()` returns structs with zero-value LCM fields to confirm the end-to-end path.
