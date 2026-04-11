# H015: Reverse-scan sponsorship pairing matches currently valid sandwich shapes

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`EndSponsoringFutureReserves` metadata should identify the begin-sponsor that is actually being closed for that end operation. If Stellar allowed overlapping sponsorship contexts that reused the same sponsored account, a simple reverse scan could attach the wrong sponsor.

## Mechanism

Both `findInitatingBeginSponsoringOp()` helpers walk backward through prior operations and return the first `BeginSponsoringFutureReserves` whose `SponsoredId` matches the current operation's source account. That looked risky because the scan does not track an explicit sponsorship stack or skip already-closed begin/end pairs.

## Trigger

Execute a successful transaction containing multiple open sponsorship contexts for the same sponsored account, then inspect the `begin_sponsor` fields exported for the closing `EndSponsoringFutureReserves` operations.

## Target Code

- `internal/transform/operation.go:533-552` — standalone reverse scan for begin sponsorship pairing
- `internal/transform/operation.go:1563-1567` — operation-details path uses the reverse-scan result
- `internal/transform/operation.go:1721-1763` — duplicate helper path for wrapper-based operation logic

## Evidence

The code never records nesting depth or previously closed begin/end pairs; it simply returns the nearest prior begin for the same sponsoree. If overlapping same-sponsoree sponsorship contexts were valid, the second closing operation could reuse the wrong earlier begin.

## Anti-Evidence

Current Stellar sponsored-reserve rules require each sponsorship context to be closed before another begins; overlapping open sponsorship contexts are not valid successful transaction shapes. Under that constraint, the nearest prior begin with the same sponsored account is exactly the one being closed, so the reverse scan matches the protocol's reachable sandwich structure.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The reverse-scan helper would be too weak for overlapping sponsorship stacks, but those transaction shapes are not currently valid successful Stellar transactions. For the reachable sandwich structure enforced by the protocol, the code's backward match is sufficient.

### Lesson Learned

When a helper looks like it should model a stack, confirm the protocol actually admits the stacked state you are worried about. A simpler search can still be correct if transaction validation rules rule out the ambiguous cases.
