# H013: Config-setting `eviction_scan_size` int64 conversion does not actually overflow

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_entry_changes --export-config-settings --write-parquet` converts `eviction_scan_size` into a signed parquet field, the conversion should still preserve every valid on-chain value without wraparound or sign changes. A large state-archival setting should not become negative in parquet output.

## Mechanism

At first glance, `ConfigSettingOutput.EvictionScanSize` looks suspicious because the JSON schema uses `uint64` while `ConfigSettingOutput.ToParquet()` casts it to `int64`. If the upstream XDR source were also 64-bit unsigned, values above `math.MaxInt64` would wrap into negative parquet values and corrupt the exported config-setting row.

## Trigger

Run `export_ledger_entry_changes --export-config-settings --write-parquet` on a ledger containing large state-archival settings and inspect the `eviction_scan_size` parquet column for negative values.

## Target Code

- `internal/transform/schema.go:617-619` — JSON config-setting schema uses `uint64` for `EvictionScanSize`
- `internal/transform/parquet_converter.go:399-405` — parquet conversion casts `EvictionScanSize` to `int64`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:60784-60788` — upstream `StateArchivalSettings` defines `EvictionScanSize` as `Uint32`

## Evidence

The local transform schema widens `EvictionScanSize` to `uint64`, and the parquet converter later narrows it back to `int64`, which is exactly the shape of a potential overflow bug. That made this a worthwhile type-audit target.

## Anti-Evidence

Tracing the XDR source shows the protocol field is only `Uint32`, not `Uint64`. Every valid on-chain `EvictionScanSize` therefore fits comfortably in an `int64`, so the cast cannot overflow for real Stellar data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected overflow depends on an unsigned 64-bit source that does not exist in the current Stellar XDR. Because `StateArchivalSettings.EvictionScanSize` is `Uint32`, the `uint64 -> int64` conversion in the parquet converter is behaviorally safe for all real inputs.

### Lesson Learned

For type-audit findings, the decisive question is the upstream XDR range, not the local Go field width alone. A widened JSON struct field can still be safe if the underlying on-chain type is much smaller than the parquet target.
