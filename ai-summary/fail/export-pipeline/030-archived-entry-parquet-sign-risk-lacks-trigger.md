# H030: `soroban_resources_archived_entries` Parquet sign-loss risk lacks a concrete trigger

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If transaction Parquet export stores archived Soroban entry indices, every
exported element should remain a non-negative footprint index and preserve the
same value as the JSON `soroban_resources_archived_entries` array.

## Mechanism

At first glance, `TransactionOutputParquet` looks suspicious because it models
`SorobanResourcesArchivedEntries` as `[]uint32` with a plain `INT32` repeated
tag instead of an unsigned logical type. If legitimate archived-entry indices
could exceed `math.MaxInt32`, a Parquet reader might interpret them as negative
signed values or otherwise mis-handle the column.

## Trigger

Export a Soroban transaction whose `archivedSorobanEntries` array contains an
index above `2147483647` and inspect the resulting Parquet column.

## Target Code

- `internal/transform/transaction.go:171-174` — copies XDR archived-entry indices into the JSON/parquet model as `[]uint32`
- `internal/transform/schema_parquet.go:66` — declares the field as `type=INT32, repetitiontype=REPEATED`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:34014-34022` — XDR defines these values as footprint indices in `archivedSorobanEntries []Uint32`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/README.md:50-61,105-117` — parquet-go documents unsigned INT32 values via `convertedtype=UINT_32`, while plain `INT32` is the signed path

## Evidence

The field is one of the few repeated unsigned integer arrays in the repository
that does not widen to `int64` or declare an unsigned logical type. The
parquet-go README explicitly distinguishes `UINT_32` from plain `INT32`.

## Anti-Evidence

The XDR comment shows these are not arbitrary 32-bit counters; they are indices
into a single transaction footprint vector. I did not find a concrete,
repository-grounded path that could produce billions of archived entries in one
transaction, so I could not substantiate a real on-chain trigger for values near
the signed `INT32` boundary.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The type mismatch is real at the schema level, but the trigger remains
theoretical. Without a concrete way to produce archived-entry indices anywhere
near `math.MaxInt32`, I cannot show current exported data being corrupted rather
than merely carrying a far-future encoding risk.

### Lesson Learned

Unsigned/signed Parquet mismatches only become viable data-integrity findings
when the source domain can realistically reach the signed boundary. For bounded
index vectors, prove the maximum practical index before treating the missing
logical type as an active bug.
