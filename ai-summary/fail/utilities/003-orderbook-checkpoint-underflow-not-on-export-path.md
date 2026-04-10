# H003: Early-ledger checkpoint underflow is not on a live export path

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the ETL computes a “most recent checkpoint” for ledgers `1` through `62`, it should clamp or reject the value instead of underflowing to `4294967295`. An exported orderbook stream should never start from an impossible checkpoint.

## Mechanism

`GetMostRecentCheckpoint` returns `seq - remainder` when `(seq+1)%64 != 0`, which underflows for every `seq < 63`. That looked like it could poison orderbook initialization and therefore emit missing or broken orderbook data for early ledgers.

## Trigger

Call `StreamOrderbooks` with `start` in the range `1..62`, so it computes `checkpointSeq := utils.GetMostRecentCheckpoint(start)` before updating the initial orderbook.

## Target Code

- `internal/utils/main.go:GetMostRecentCheckpoint:867-874` — underflows for `seq < 63`
- `internal/input/orderbooks.go:StreamOrderbooks:212-216` — consumes the helper result directly

## Evidence

The arithmetic is straightforward: for `seq=1`, `remainder=(1+1)%64=2`, so `seq-remainder` wraps to `4294967295` in `uint32`. I reproduced the same wraparound for every `seq` from `1` through `62`.

## Anti-Evidence

There is no command or other package reference to `StreamOrderbooks` anywhere else in this repo, so I could not identify an active JSON/Parquet export flow that reaches this code today. The path appears dormant rather than part of the current ETL surface.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The helper math is wrong, but I could not tie it to a currently reachable export command in this repository, so it does not yet qualify as a concrete data-integrity issue for the ETL output surface.

### Lesson Learned

Arithmetic bugs in shared helpers still need a live path into exported datasets before they become viable findings. Reachability matters as much as local correctness.
