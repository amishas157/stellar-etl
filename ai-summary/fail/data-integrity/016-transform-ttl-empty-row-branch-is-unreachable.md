# H016: `TransformTtl()` can silently emit an empty row for non-TTL changes

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a non-TTL ledger change is ever routed into `TransformTtl()`, the function should reject it with an error rather than returning a zero-valued `TtlOutput` that could be exported as a plausible-but-empty row.

## Mechanism

`TransformTtl()` contains a guard that returns `TtlOutput{}, nil` when `ledgerEntry.Data.Type != xdr.LedgerEntryTypeTtl`. At first glance that looks like an empty-shell bug: a misrouted change could bypass error handling and append a metadata-only TTL row.

## Trigger

Call `TransformTtl()` with a ledger change that is not actually `LedgerEntryTypeTtl`, then let the caller append the returned row without additional validation.

## Target Code

- `internal/transform/ttl.go:18-26` — `GetTtl()` plus the suspicious `return TtlOutput{}, nil` branch
- `cmd/export_ledger_entry_changes.go:258-270` — live caller appends whatever `TransformTtl()` returns when the batch is already grouped under `LedgerEntryTypeTtl`

## Evidence

The explicit `return TtlOutput{}, nil` branch is present in the transformer and would be dangerous if it were reachable from misclassified input.

## Anti-Evidence

The branch is effectively dead in the current pipeline. `TransformTtl()` first calls `ledgerEntry.Data.GetTtl()`, which already returns an error for non-TTL entries, and `export_ledger_entry_changes` only invokes the function for changes drawn from the `LedgerEntryTypeTtl` bucket.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not find a concrete live path that reaches the nil-error empty-row branch. Non-TTL inputs fail earlier at `GetTtl()`, and the only shipped caller already filters the batch by `LedgerEntryTypeTtl`.

### Lesson Learned

Suspicious zero-value returns are only meaningful when they are actually reachable. For transform helpers, always trace both the function's internal guards and the caller's input filtering before classifying an empty-shell branch as a live integrity bug.
