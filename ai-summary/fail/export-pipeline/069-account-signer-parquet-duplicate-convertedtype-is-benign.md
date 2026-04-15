# H069: Account-signer Parquet duplicate `convertedtype` tags are benign

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: parquet schema / numeric encoding
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes --export-account-signers --write-parquet` should encode `last_modified_ledger`, `ledger_entry_change`, and `ledger_sequence` with the intended unsigned INT64 Parquet annotation. Duplicate tag text should not make these columns fail writer initialization or serialize with the wrong signedness.

## Mechanism

`AccountSignerOutputParquet` repeats `convertedtype` in three struct tags (`convertedtype=INT64, convertedtype=UINT_64`), which initially looks like a schema bug that could either fail parsing or preserve the wrong annotation. If parquet-go honored the first value or treated duplicates as invalid, large unsigned ledger counters could be mislabeled or the whole export could abort.

## Trigger

Run `export_ledger_entry_changes --export-account-signers --write-parquet` on any ledger range containing account-signer rows, especially rows with large `last_modified_ledger` / `ledger_sequence` values.

## Target Code

- `internal/transform/schema_parquet.go:103-114` — duplicate `convertedtype` tags on `AccountSignerOutputParquet`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/common/common.go:76-105` — tag parser assigns `mp.ConvertedType = val` on each `convertedtype`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/example/type.go:23-30` — parquet-go examples use the same duplicate-tag pattern for integer logical types

## Evidence

The local schema literally repeats the same key twice on the three INT64 fields. parquet-go's tag parser is simple key/value iteration, so duplicate keys are not rejected at parse time.

## Anti-Evidence

The parser overwrites `ConvertedType` on each `convertedtype` token, so the trailing `UINT_64` wins deterministically. parquet-go also ships example tags with the same duplicate-key style, indicating this odd formatting is tolerated by the library rather than treated as an error.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated in `export-pipeline`

### Why It Failed

parquet-go does not error on the duplicate key and simply keeps the final `convertedtype` value, which is already the intended `UINT_64`. The export therefore preserves the same effective annotation it would have had with a cleaned-up tag.

### Lesson Learned

For parquet-tag investigations, duplicate text alone is not enough — the parser's duplicate-key behavior determines whether the bug is real. If the library is "last write wins" and the trailing value matches the intended annotation, the schema quirk is cosmetic rather than corrupting.
