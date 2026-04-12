# H024: `export_ledger_entry_changes` select loop might drop the final batch when `closeChan` wins

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: missing final batch in ledger-entry-change exports
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The `export_ledger_entry_changes` consumer loop should process every batch produced by `input.StreamChanges()` before exiting. Even at shutdown, the final batch should either be exported or the command should fail loudly rather than silently dropping the tail of the requested range.

## Mechanism

At first glance, the `select` in `export_ledger_entry_changes` looks racy: it listens to both `closeChan` and `changeChan`, so it seems possible for `closeChan` to fire first and make the command return before the last `changeChan` batch is drained. That would produce a valid-looking set of output files missing the final batch.

## Trigger

Run `export_ledger_entry_changes` on a bounded range whose last batch arrives at about the same time as stream shutdown. If `closeChan` could win while a real batch was still pending on `changeChan`, the command would return early and omit the trailing batch's files.

## Target Code

- `cmd/export_ledger_entry_changes.go:76-86` — consumer loop selects between `closeChan` and `changeChan`
- `internal/input/changes.go:162-177` — producer sends batches, closes `changeChannel`, then signals `closeChan`

## Evidence

The consumer does `case <-closeChan: return` ahead of `case batch, ok := <-changeChan`, which initially looks like it could exit before draining the last batch. `StreamChanges()` also uses a separate completion channel instead of relying solely on `range changeChannel`, so there appears to be a second shutdown signal that might race with the final data delivery.

## Anti-Evidence

`changeChan` is created as an unbuffered channel, so every `changeChannel <- batch` send at `internal/input/changes.go:170` must synchronize with a receiver before execution can continue to `close(changeChannel)` and `closeChan <- 1`. That means `closeChan` cannot become ready while a real, unread batch is still pending. Once `changeChannel` is closed, the only remaining receive there is the zero-value `ok=false` case, so the loop may spin briefly but it is not dropping data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The feared race depends on a final batch coexisting with a ready `closeChan` signal, but the producer's unbuffered send order forbids that state. `StreamChanges()` cannot reach `closeChan <- 1` until the last batch send has already been received by the consumer.

### Lesson Learned

When a shutdown race seems plausible around a `select`, verify the buffering and send order before escalating it. An unbuffered data channel plus "send batch, then close, then signal done" sequencing is a strong anti-pattern against silent tail loss, even if the consumer loop looks suspicious at first glance.
