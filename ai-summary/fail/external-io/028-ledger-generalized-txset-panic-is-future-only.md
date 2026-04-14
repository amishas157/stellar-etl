# H028: Ledger generalized-txset panic branches are live on current protocol data

**Date**: 2026-04-14
**Subsystem**: external-io
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Current legitimate ledger exports should successfully flatten both classic and generalized transaction sets without panicking. As long as the current XDR still defines only the presently supported union arms, no valid ledger should hit the `"Unsupported ..."` panic branches.

## Mechanism

I suspected `GetTransactionSet()` / `getTransactionPhase()` might already be missing a currently valid generalized-txset union arm and therefore abort `export_ledgers` on legitimate network data. That would turn a normal protocol variant into a hard export stop instead of a structured error or a normal ledger row.

## Trigger

Construct or read a ledger whose tx-set uses one of the current XDR-defined generalized arms for `TransactionHistoryEntryExt` or `TransactionPhase`, then run it through `TransformLedger()`.

## Target Code

- `internal/transform/ledger.go:GetTransactionSet:168-176` — panics on unsupported `TransactionHistoryEntry.Ext`
- `internal/transform/ledger.go:getTransactionPhase:181-207` — panics on unsupported phase/component variants
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:15064-15120` — current `TransactionHistoryEntryExt` defines only arms `0` and `1`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:14364-14520` — current `TransactionPhase` defines only arms `0` and `1`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:13737-13745` — current `TxSetComponentType` defines only component type `0`

## Evidence

The exporter does use raw `panic(...)` calls instead of returning errors, so if a legitimate current arm were unhandled the failure mode would be severe and immediate.

## Anti-Evidence

The current XDR module only defines the exact arms the exporter already handles: `TransactionHistoryEntryExt` `V=0/1`, `TransactionPhase` `V=0/1`, and a single `TxSetComponentType=0`. That makes the panic branches future-only rather than reachable on present-day legitimate ledger data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code is brittle, but not against the current protocol. Every currently defined generalized-txset union arm is already covered, so the panic paths require a future XDR extension rather than a present-day ledger.

### Lesson Learned

Future-compatibility panic branches are not enough by themselves for this objective. Before filing a protocol-union hypothesis, verify that the current pinned XDR already defines an unhandled arm; otherwise the issue is only speculative forward-compatibility risk.
