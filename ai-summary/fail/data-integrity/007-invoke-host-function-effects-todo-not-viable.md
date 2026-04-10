# H007: `InvokeHostFunction` effects do not currently lose transaction-level events

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformEffect()` should emit account credited/debited contract effects for
the current set of supported SAC event types (`transfer`, `mint`, `clawback`,
`burn`) without omitting any effect-bearing event class that the exporter
already knows how to interpret.

## Mechanism

At first glance, the `TODO` in the `InvokeHostFunction` branch looks like a
coverage gap: `effects()` calls `transaction.GetContractEvents()` instead of
the broader `TransformContractEvent()` path. If transaction-level CAP-67 events
carried the same SAC transfer/mint/clawback/burn payloads, the effect exporter
could undercount rows by only reading operation-level contract events.

## Trigger

1. Run `export_effects` on ledgers containing `InvokeHostFunction`
   transactions with CAP-67 unified metadata.
2. Compare the effect path's use of `GetContractEvents()` against the broader
   contract-event transform path.
3. Suspect that transaction-level events are being skipped.

## Target Code

- `internal/transform/effects.go:118-130` — `InvokeHostFunction` branch uses
  `transaction.GetContractEvents()` and includes a TODO about broader event
  coverage
- `internal/transform/effects.go:1322-1433` — effect generation only handles
  SAC `transfer`/`mint`/`clawback`/`burn` event types
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.1.0/xdr/Stellar-ledger.x:500-506`
  — current XDR defines transaction-level events as fee events/refunds

## Evidence

The `effects()` implementation explicitly notes `TODO: Replace GetContractEvents
with TransformContractEvent to get all the events`, which suggests the author
was aware of a narrower read path. The broader contract-event transform indeed
merges `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents`.

## Anti-Evidence

Current XDR documents `TransactionEvent` as "limited to the fee events (when
fee is charged or refunded)", while `addInvokeHostFunctionEffects()` only
translates SAC `transfer`/`mint`/`clawback`/`burn` events into effects. That
means the apparently missing transaction-level events are not part of the
effect types this code currently exports, so replacing `GetContractEvents()`
would not change today's effect rows.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspicious TODO does not correspond to a current data-loss bug because
transaction-level CAP-67 events are presently fee/refund events, not the SAC
transfer-style events that `addInvokeHostFunctionEffects()` turns into effect
rows.

### Lesson Learned

Do not treat every broader-event TODO as a live integrity bug. First verify the
current XDR meaning of the omitted event class; here, the omitted class is a
different semantic family (fees) than the effect types this exporter handles.
