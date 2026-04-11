# H041: `extra_signers` parquet list tag is invalid

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Suspected parquet schema corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `TransactionOutputParquet.ExtraSigners` is a supported repeated string column, its parquet tag should match the syntax expected by the `parquet-go` library so valid transaction rows can be written without schema corruption.

## Mechanism

At first glance, `ExtraSigners` looks malformed because the tag uses `type=MAP, convertedtype=LIST` instead of a more obvious `LIST` element form. If that tag were unsupported, parquet exports with extra signers would either fail schema initialization or serialize the field under the wrong logical shape.

## Trigger

Run a parquet transaction export over a ledger range that contains `extra_signers`.

## Target Code

- `internal/transform/schema_parquet.go:31-74` - `TransactionOutputParquet.ExtraSigners`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/example/type.go:50-52` - upstream library example uses the same tag pattern for `[]string`

## Evidence

`schema_parquet.go` defines `ExtraSigners` as `[]string` with `parquet:"name=extra_signers, type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8"` (`internal/transform/schema_parquet.go:59`). That looked suspicious because the physical type string says `MAP` while the converted type says `LIST`.

## Anti-Evidence

The suspicion collapses against the library's own example: `parquet-go` documents the exact same tag form for a `[]string` field (`example/type.go:50-52`). That makes the tag a library convention rather than a local schema bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The allegedly invalid `type=MAP, convertedtype=LIST` tag matches the upstream `parquet-go` example for repeated UTF-8 string lists exactly, so the exporter is following the library's supported schema syntax rather than inventing a broken one.

### Lesson Learned

Parquet-go's struct-tag grammar is idiosyncratic. Before treating a tag shape as malformed, compare it with the library's own examples rather than assuming the logical type must look like the physical type name.
