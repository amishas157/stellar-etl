# H003: `export_ledger_entry_changes --export-balances --write-parquet` silently omits claimable-balance parquet output

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When claimable balances are exported with `--write-parquet`, the Parquet side should contain the same claimable-balance rows that were written to the JSON batch file for that resource.

## Mechanism

`exportTransformedData()` recognizes `transform.ClaimableBalanceOutput` only by setting `skip = true`; it never appends the row to a Parquet slice and never assigns a schema. The command therefore writes correct JSON claimable-balance batches, then silently skips Parquet creation and upload for that resource even though `write-parquet` was requested.

## Trigger

Run `stellar-etl export_ledger_entry_changes --export-balances true --write-parquet ...` on any ledger batch containing claimable-balance changes, such as the existing claimable-balance test range in `cmd/export_ledger_entry_changes_test.go`.

## Target Code

- `cmd/export_ledger_entry_changes.go:90-107` — pre-creates the `claimable_balances` resource bucket when `--export-balances` is enabled
- `cmd/export_ledger_entry_changes.go:331-335` — explicitly marks `ClaimableBalanceOutput` as `skip = true`
- `cmd/export_ledger_entry_changes.go:368-372` — writes/upload Parquet only when `skip` is false
- `README.md:305-316` — advertises `export-balances` as a normal ledger-entry-changes export type

## Evidence

The production switch contains an in-code comment saying claimable-balance parquet is being skipped because it is "not needed in the current scope of work." There is no compensating warning, alternate artifact, or flag validation preventing users from requesting Parquet for this resource.

## Anti-Evidence

The JSON claimable-balance export still works, so the bug is Parquet-only. If the project intentionally treats claimable-balance Parquet as unsupported, the missing piece may be documentation or flag rejection rather than schema generation, but the current runtime behavior is still silent omission.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full export path for claimable balances through `export_ledger_entry_changes.go`. Confirmed that `exportTransformedData()` at line 331-335 intentionally sets `skip = true` for `ClaimableBalanceOutput` with an explicit code comment: "Skipping ClaimableBalanceOutputParquet because it is not needed in the current scope of work. Note that ClaimableBalanceOutputParquet uses nested structs that will need to be handled for parquet conversion." Additionally, `schema_parquet.go:131-135` has the `ClaimableBalanceOutputParquet` struct definition **commented out** with the same explanation. The JSON struct `ClaimableBalanceOutput` contains `Claimants []Claimant` where `Claimant` embeds `xdr.ClaimPredicate` — a nested/recursive XDR union type that the project's flat Parquet schema approach cannot represent without additional flattening work.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:331-335` — `case transform.ClaimableBalanceOutput`: intentionally sets `skip = true` with comment explaining the design decision
- `cmd/export_ledger_entry_changes.go:370-373` — Parquet write gated on `!skip && writeParquet`, so claimable balances are never written
- `internal/transform/schema_parquet.go:131-135` — `ClaimableBalanceOutputParquet` is commented out, never implemented
- `internal/transform/schema.go:153-169` — `ClaimableBalanceOutput` contains `Claimants []Claimant` with nested `xdr.ClaimPredicate`
- `internal/transform/parquet_converter.go:15-17` — `SchemaParquet` interface requires `ToParquet()` which `ClaimableBalanceOutput` does not implement

### Why It Failed

This is **working-as-designed behavior**, not a bug. The developers explicitly and intentionally chose not to implement Parquet support for claimable balances due to the complexity of flattening the nested `Claimants` field (which contains `xdr.ClaimPredicate`, a recursive XDR union). The evidence is unambiguous:

1. The `ClaimableBalanceOutputParquet` struct was **never implemented** — it's commented out in `schema_parquet.go` with an explanatory note
2. The `skip = true` in the switch statement includes an explicit code comment documenting the intentional omission
3. `ClaimableBalanceOutput` does not implement the `SchemaParquet` interface (`ToParquet()` method) — it literally cannot be used for Parquet output
4. No code path was "broken" — the Parquet code path for this type was never written

A missing-but-never-implemented feature is not a data correctness bug. The code does exactly what the developers designed it to do.

### Lesson Learned

Distinguish between "silent data omission from a broken code path" (a bug) and "silent data omission from a deliberately unimplemented feature" (a feature gap). When the code contains explicit comments explaining the skip, and the corresponding type/struct was never created, the omission is by design. A genuine data-omission bug would show a code path that *should* produce output but fails due to incorrect logic, not a feature that was consciously deferred.
