# H026: Negative transaction or operation orders silently mask into wrong TOIDs

**Date**: 2026-04-12
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Any live TOID producer should either reject negative transaction/operation indexes or never supply them, so exported IDs always preserve the real `(ledger, transaction, operation)` position. A negative transaction or operation order should not be silently remapped into the maximum packed bitfield value.

## Mechanism

`ToInt64()` panics on negative ledger sequences and on values above the transaction/operation masks, but it does not reject negative `TransactionOrder` or `OperationOrder` before bit-masking them into the packed ID. That means `-1` would be encoded as `1048575` for the transaction field or `4095` for the operation field, yielding a plausible but false TOID if any production caller ever passed a negative value.

## Trigger

Identify a production caller that can pass `tx = -1` or `op = -1` into `toid.New(...).ToInt64()` under normal export operation, then compare the emitted ID against the correct tuple.

## Target Code

- `internal/toid/main.go:139-155` — bit-packs transaction and operation fields without `< 0` checks
- `internal/transform/ledger.go:20-23` — ledger IDs always use literal `0, 0`
- `internal/transform/transaction.go:23-27` — transaction IDs derive from unsigned ledger/transaction indexes
- `internal/transform/operation.go:31-32` — operation IDs derive from local slice indexes with `+1`
- `internal/input/trades.go:64-70` — trade staging TOIDs use Go range indexes
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251210093830-27970a00c2d7/processors/token_transfer/token_transfer_event.go:89-103` — token-transfer metadata is built from non-negative ledger/transaction indexes and normalized operation indexes

## Evidence

The encoder omits explicit negative checks for transaction and operation components even though it does check overflow and negative ledgers. Bit-masking signed negatives would indeed create silent wraparound instead of an obvious failure if a caller supplied them.

## Anti-Evidence

Every live exporter in this repository sources transaction and operation components from non-negative origins: constants, Go slice indexes, `operationIndex+1`, or upstream metadata that is explicitly documented as one-based. I did not find any production path that can feed a negative transaction or operation order into `toid.New(...)`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The encoder has a theoretical sharp edge, but the current ETL never supplies negative transaction or operation indexes, so the masking behavior cannot corrupt exported data today.

### Lesson Learned

For low-level packing helpers, an unsafe branch is only in scope when a real production caller can reach it with legitimate input. Validate the value origin at each call site before filing a helper-level correctness bug.
