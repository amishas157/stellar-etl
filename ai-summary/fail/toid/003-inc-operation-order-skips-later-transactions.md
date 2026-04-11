# H001: IncOperationOrder skips later transactions in the same ledger

**Date**: 2026-04-10
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a TOID cursor advances from the last representable operation slot of a transaction, it should move to the next lexicographic TOID in the same ledger while there are still later transaction-order values available. For example, advancing `(ledger=100, tx=1, op=4095)` should yield `(ledger=100, tx=2, op=0)`, not skip straight to ledger 101.

## Mechanism

`IncOperationOrder()` only increments `OperationOrder`; on overflow it resets the operation field to `0` and increments `LedgerSequence`, leaving `TransactionOrder` untouched. That means any caller using this helper as a paging primitive will skip every transaction that follows a max-op transaction in the same ledger, producing silently incomplete result sets.

## Trigger

Call `(*ID).IncOperationOrder()` on an ID whose `OperationOrder == OperationMask` and whose `TransactionOrder < TransactionMask`, then use the resulting TOID as the next-page lower bound.

## Target Code

- `internal/toid/main.go:66-83` — TOID bit-field layout reserves a transaction field between ledger and operation.
- `internal/toid/main.go:117-126` — overflow path resets `OperationOrder` and jumps directly to the next ledger.

## Evidence

The helper comment says it "allows queries to easily advance a cursor to the next operation," but the implementation never increments `TransactionOrder`. In the packed ordering, `(ledger, tx+1, 0)` is the immediate successor of `(ledger, tx, OperationMask)` whenever another transaction slot exists in the same ledger.

## Anti-Evidence

There is no in-repo caller today, so current CLI exports do not exercise this path. A reviewer should confirm whether any downstream pagination logic imports this helper before treating it as a live production issue.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `IncOperationOrder()` at `internal/toid/main.go:120-127`. The function increments `OperationOrder`, and on overflow (exceeding `OperationMask = 4095`) resets it to 0 and increments `LedgerSequence`, completely skipping `TransactionOrder`. The logic bug is genuine — the correct next TOID after `(L, T, 4095)` when `T < TransactionMask` is `(L, T+1, 0)`, not `(L+1, T, 0)`. However, a full codebase search confirms zero callers outside the test file.

### Code Paths Examined

- `internal/toid/main.go:120-127` — `IncOperationOrder()` confirmed to skip `TransactionOrder` on overflow, jumping directly to `LedgerSequence++`
- `internal/toid/main_test.go:127-135` — Test validates the incorrect behavior (asserts `LedgerSequence` increments to 1, does not check `TransactionOrder`)
- Grep across entire repo — `IncOperationOrder` appears only in `main.go` (definition), `main_test.go` (test), and the hypothesis file itself. No ETL export code, no transform code, no command code calls this function.

### Why It Failed

The bug in `IncOperationOrder()` is real at the function level — the overflow logic skips remaining transactions in a ledger. However, **no code path in the ETL actually calls this function**. The only references are the function definition and its unit test. Since no export, transform, or command code invokes `IncOperationOrder`, it cannot produce wrong output in any current ETL pipeline. This falls under the explicit out-of-scope criterion: "Theoretical issues without a concrete code path producing wrong output."

### Lesson Learned

When evaluating utility functions with real logic bugs, verify that the function is actually called by production code paths before classifying it as a data corruption risk. A dead-code bug, however genuine, cannot corrupt output that never exercises it.
