# H001: Claimable-balance parquet omission in ledger-entry changes

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If claimable-balance rows are not supported in parquet mode, `export_ledger_entry_changes` should skip parquet generation for that resource instead of attempting to serialize it with the wrong schema. JSON output should remain correct.

## Mechanism

An initial suspicion was that `exportTransformedData` might mishandle `transform.ClaimableBalanceOutput` and emit malformed or empty parquet. That would matter because the command accepts `--write-parquet` globally and claimable balances are one of its exportable resources.

## Trigger

Run `export_ledger_entry_changes` with `--export-balances` and `--write-parquet` on a ledger range containing claimable-balance changes.

## Target Code

- `cmd/export_ledger_entry_changes.go:331-335` — explicit `ClaimableBalanceOutput` branch in the parquet switch

## Evidence

There is no `ClaimableBalanceOutputParquet` type in `internal/transform/schema_parquet.go`, so this looked like a likely missing-case bug at first glance.

## Anti-Evidence

The switch handles this exact case explicitly and sets `skip = true`, with comments stating claimable-balance parquet is out of scope because of nested structs. That means the code avoids writing incorrect parquet for this resource; it is an intentional feature gap rather than silent data corruption.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The implementation contains an explicit guard for `transform.ClaimableBalanceOutput`, so the command does not accidentally serialize claimable balances with the wrong parquet schema.

### Lesson Learned

In this subsystem, a missing parquet schema is only a viable bug if the command still falls through to `WriteParquet`. When the type switch sets a deliberate `skip` flag with an explanatory comment, treat it as an intentional scope boundary instead of a corruption finding.
