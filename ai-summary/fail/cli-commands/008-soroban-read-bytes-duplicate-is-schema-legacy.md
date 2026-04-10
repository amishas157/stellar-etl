# H008: `soroban_resources_read_bytes` duplicate looks suspicious but has no distinct current XDR source

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If Stellar XDR exposes both total Soroban read bytes and disk-backed read bytes, `export_transactions` should export those as distinct values. A transaction with memory reads beyond disk-backed reads should not report the same number in both columns.

## Mechanism

`TransformTransaction` assigns both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` from the same `outputSorobanResourcesDiskReadBytes` variable. That initially looked like a copy-paste bug that would under-report total read bytes for Soroban transactions.

## Trigger

Run `export_transactions` on Soroban transactions and compare the two read-byte columns for rows with non-zero resource usage.

## Target Code

- `internal/transform/transaction.go:137-140` — only a disk-read variable is declared
- `internal/transform/transaction.go:161-165` — Soroban resources populate only `DiskReadBytes`
- `internal/transform/transaction.go:257-260` — both exported columns receive the disk-read value
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:SorobanResources:33926-33930` — current XDR struct exposes `Instructions`, `DiskReadBytes`, and `WriteBytes`, but no separate `ReadBytes`

## Evidence

The transform code clearly duplicates the disk-read value into both exported fields. This is exactly the sort of copy-paste pattern that often produces wrong output in this codebase.

## Anti-Evidence

The current upstream XDR type does not define a distinct `ReadBytes` field to map from. `xdr.SorobanResources` only carries `DiskReadBytes` and `WriteBytes`, so there is no on-chain source value available today that would let stellar-etl populate a different `soroban_resources_read_bytes`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Although the duplicated assignment in `transaction.go` looks like a classic mapping bug, current Stellar XDR does not expose a distinct total-read-bytes field. Without a separate source field, this is a legacy/output-schema mismatch rather than a concrete case of stellar-etl corrupting an available on-chain value.

### Lesson Learned

For type- or field-mapping hypotheses, verify that the upstream XDR still carries a distinct source field before treating duplicate output columns as corruption. Legacy columns can survive in ETL schemas long after the upstream protocol collapses them to a single value.
