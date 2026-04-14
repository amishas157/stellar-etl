# H001: `export_ledgers --captive-core` zeroes Soroban ledger metrics

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: High
**Impact**: wrong ledger-level Soroban pricing/state metrics
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledgers` is run with `--captive-core`, the exported ledger rows should preserve the same Soroban-only ledger metadata that the normal ledger-backend path can read from `LedgerCloseMeta`, including non-zero `soroban_fee_write_1kb` and `total_byte_size_of_live_soroban_state` values on post-Soroban ledgers.

## Mechanism

The command's `--captive-core` branch does **not** call the captive-core `LedgerBackend`; it switches to `GetLedgersHistoryArchive()`, which returns `HistoryArchiveLedgerAndLCM` values with an empty `LCM`. `TransformLedger()` only fills Soroban-only metrics from `lcm.GetV1()` / `lcm.GetV2()`, so this branch silently serializes those columns as zero even when the same ledger has non-zero values in real `LedgerCloseMeta`.

## Trigger

Run `export_ledgers --captive-core` on any protocol-23+ ledger whose close meta carries non-zero Soroban ledger-extension values, such as `SorobanFeeWrite1Kb` or `TotalByteSizeOfLiveSorobanState`. Compare the JSON output to the same range exported without `--captive-core`: the `--captive-core` run will emit `0` for those fields.

## Target Code

- `cmd/export_ledgers.go:28-32` — `--captive-core` selects `GetLedgersHistoryArchive()` instead of the normal ledger-backend path
- `internal/input/ledgers_history_archive.go:GetLedgersHistoryArchive:10-34` — returns `HistoryArchiveLedgerAndLCM` with no populated `LCM`
- `internal/transform/ledger.go:61-90` — reads Soroban-only metrics exclusively from `LedgerCloseMeta` V1/V2 extensions
- `internal/transform/ledger.go:104-128` — writes those default-zero values into the exported `LedgerOutput`

## Evidence

`GetLedgersHistoryArchive()` constructs `HistoryArchiveLedgerAndLCM{Ledger: ledger}` and never sets `LCM`, so `ledger.LCM` reaches `TransformLedger()` as the zero value. In `TransformLedger()`, `outputSorobanFeeWrite1Kb` and `outputTotalByteSizeOfLiveSorobanState` are initialized to zero and only updated inside the `lcm.GetV1()` / `lcm.GetV2()` branches, which are unreachable for an empty `LCM`.

## Anti-Evidence

For pre-Soroban ledgers these fields are legitimately zero, so the discrepancy only appears on ledgers whose close meta actually carries Soroban extension data. The JSON row still looks structurally valid, which makes this easy for downstream analytics to miss.

---

## Review

**Verdict**: NOT_VIABLE — duplicate of `success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published` and `success/utilities/003-captive-core-flag-routes-to-history-archive.md.gh-published`
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis describes the exact same bug already confirmed and published in the `data-input` and `utilities` success directories. The `--captive-core` flag in `export_ledgers` routes to `GetLedgersHistoryArchive()` which omits `LCM`, causing `TransformLedger()` to zero Soroban fields. This was fully traced, PoC-verified, and published as `success/data-input/001` and also covered in the broader `success/utilities/003`.

### Code Paths Examined

- `cmd/export_ledgers.go:28-32` — confirmed `UseCaptiveCore` routes to `GetLedgersHistoryArchive()`
- `internal/input/ledgers_history_archive.go:24-26` — confirmed `LCM` field is never set
- `internal/transform/ledger.go:61-91` — confirmed Soroban fields only populated from `lcm.GetV1()`/`lcm.GetV2()`

### Why It Failed

This is an exact duplicate of two already-confirmed findings: `success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published` (same bug, same code path, same severity) and `success/utilities/003-captive-core-flag-routes-to-history-archive.md.gh-published` (broader scope covering both ledgers and assets).

### Lesson Learned

The `--captive-core` zero-LCM bug in `export_ledgers` is one of the earliest confirmed findings in this codebase. Check `success/data-input/` and `success/utilities/` before re-proposing captive-core routing hypotheses.
