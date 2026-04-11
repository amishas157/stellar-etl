# H002: TOID should support the full uint32 ledger range because it reserves 32 ledger bits

**Date**: 2026-04-10
**Subsystem**: toid
**Severity**: Medium
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

At first glance, a 32-bit ledger field suggests TOIDs should encode every valid Stellar `uint32` ledger sequence up to `4294967295`, preserving ledger/transaction/operation IDs across the full XDR range.

## Mechanism

The suspected bug was that exporters cast `uint32` ledger sequences to `int32` before calling `toid.New(...)`, so ledgers above `2147483647` become negative and `ToInt64()` panics. That looked like an avoidable narrowing conversion because `LedgerMask` itself spans 32 bits.

## Trigger

Process any ledger with sequence `2147483648` or above and try to derive a TOID-based ledger, transaction, or operation identifier.

## Target Code

- `internal/toid/main.go:16-18` — TOIDs are stored in a signed `int64` for SQL compatibility.
- `internal/toid/main.go:129-156` — `New()` takes `int32` ledger values and `ToInt64()` rejects negative ledger sequences.
- `internal/toid/main_test.go:47-63` — tests explicitly treat `math.MaxInt32` as the largest valid ledger component and expect negative ledgers to panic.

## Evidence

Export paths routinely cast `uint32` ledger sequences down to `int32` before TOID encoding, and the package still masks the ledger field with 32 bits. That combination initially suggests a latent truncation bug.

## Anti-Evidence

The package design is intentionally constrained to a **signed** `int64`, which leaves only 63 usable bits; allowing ledger bit 31 to be set would flip the sign bit after the 32-bit left shift. The code comments, the MaxInt32 test case, and the upstream SDK's old FIXME all point to this as a known representation limit rather than an accidental implementation bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Supporting the full `uint32` ledger range would require abandoning the current "positive signed int64" TOID representation, so the `int32` ceiling is a design constraint, not a local corruption bug.

### Lesson Learned

For TOID audits, treat the signed-`int64` storage choice as the first hard boundary. A wider bit mask alone is not enough evidence of a bug if the sign bit would be consumed by a nominally valid value.
