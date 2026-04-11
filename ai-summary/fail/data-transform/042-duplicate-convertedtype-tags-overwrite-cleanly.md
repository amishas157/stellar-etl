# H042: Duplicate `convertedtype=` parquet tags corrupt integer columns

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Suspected parquet schema corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If duplicate `convertedtype=` keys appear in a parquet struct tag, the parser should either reject the tag or deterministically resolve it without producing silently corrupted numeric columns.

## Mechanism

`AccountSignerOutputParquet` and `OperationOutputParquet` contain tags like `convertedtype=INT64, convertedtype=UINT_64`, which looked like they might confuse the tag parser and produce mismatched logical types for unsigned ledger identifiers.

## Trigger

Inspect or export parquet rows from `account_signers` or `history_operations`.

## Target Code

- `internal/transform/schema_parquet.go:103-129` - duplicate `convertedtype=` keys on signer and operation fields
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/common/common.go:76-120` - `StringToTag()` parser

## Evidence

The schema file does include repeated tag keys such as `convertedtype=INT64, convertedtype=UINT_64` (`internal/transform/schema_parquet.go:109-113`, `128`). That is unusual and initially suggested an ambiguous parse.

## Anti-Evidence

`parquet-go`'s parser simply processes the comma-separated entries in order and overwrites the stored field each time it sees `convertedtype` (`common.go:96-105`). The later `UINT_64` value therefore wins deterministically instead of creating a mixed or partially parsed schema.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The duplicate keys are sloppy, but not silently corrupting: `StringToTag()` overwrites `mp.ConvertedType` on each occurrence, so the final logical type is stable and predictable.

### Lesson Learned

When a suspected schema bug depends on parser ambiguity, inspect the parser. A messy tag string is not automatically a live integrity defect if the downstream parser resolves it deterministically.
