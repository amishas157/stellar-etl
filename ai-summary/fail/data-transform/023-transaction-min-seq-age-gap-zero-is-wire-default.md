# H023: Transaction `min_seq_age` / `min_seq_ledger_gap` should stay nullable

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Transactions that omit `min_seq_age` or `min_seq_ledger_gap` should export those fields as absent/null so downstream consumers can distinguish "no age/gap constraint" from an explicitly configured zero value.

## Mechanism

At first glance, `TransformTransaction()` looks like it is wrongly fabricating zeros because it converts `MinSeqAge()` and `MinSeqLedgerGap()` into `null.IntFrom(...)` whenever the helpers return non-nil pointers. That would be a real bug if the XDR wire format could represent "absent" independently from `0`.

## Trigger

Export a V2-precondition transaction whose `PreconditionsV2` omits meaningful sequence-age and sequence-ledger-gap constraints.

## Target Code

- `internal/transform/transaction.go:119-129` — wraps `MinSeqAge()` and `MinSeqLedgerGap()` in `null.IntFrom(...)`
- `internal/transform/schema.go:66-67` — schema models both fields as `null.Int`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/transaction.go:44-68` — transaction helpers always return pointers for these V2 fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:33335-33340` — `PreconditionsV2` stores `MinSeqAge` and `MinSeqLedgerGap` inline, not as optional pointers
- `internal/transform/transaction_test.go:165-168` — checked-in expectations already treat zero as the exported value for these fields

## Evidence

The helper methods are pointer-returning and the output schema is nullable, which initially suggests a missing/null distinction is being lost in the transform layer. The repository tests also show V2-precondition rows exporting explicit `0` values for these fields.

## Anti-Evidence

The XDR definition itself makes both fields mandatory inline scalars inside `PreconditionsV2`, unlike `MinSeqNum`, which is genuinely optional. That means decoded XDR cannot distinguish "omitted" from `0` for these two fields; the apparent nullability gap is upstream in the protocol shape, not a transform-layer corruption.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`MinSeqAge` and `MinSeqLedgerGap` are not optional fields in `PreconditionsV2`; they are always present on the wire with `0` as the default. Because the XDR payload itself cannot encode a distinct null state, the transform cannot preserve one.

### Lesson Learned

For transaction preconditions, only fields backed by optional XDR members (like `MinSeqNum`) can support true null-vs-zero hypotheses. Before treating a `null.Int` output field as a lost-nullability bug, verify that the upstream XDR member is actually optional rather than an inline scalar with a zero default.
