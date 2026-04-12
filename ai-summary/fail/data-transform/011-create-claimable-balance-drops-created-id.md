# H003: Successful `create_claimable_balance` rows omit the created `balance_id`

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A successful `create_claimable_balance` operation row should export the created claimable-balance identifier (`balance_id` / `balance_id_strkey`) from the success result. That identifier is the durable object key for the newly created balance, and downstream consumers should be able to join the creation row to later claim or clawback operations using the same ID.

## Mechanism

`extractOperationDetails()` serializes only the request payload (`asset`, `amount`, `claimants`) for `create_claimable_balance`. The XDR success arm, however, returns `CreateClaimableBalanceResult.BalanceId`. Because the transform never inspects that result field, successful create rows omit the only authoritative identifier of the object they just created.

## Trigger

Export any successful `create_claimable_balance` operation whose `OperationResult` includes the normal success `BalanceId`. The current operation row will contain `asset`, `amount`, and `claimants`, but no `balance_id`, while later `claim_claimable_balance` and `clawback_claimable_balance` rows do export that identifier.

## Target Code

- `internal/transform/operation.go:881-885` — `create_claimable_balance` details only copy request fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:41839-41880` — success result carries `BalanceId`
- `internal/transform/operation_test.go:1478-1497` — current expected output for successful `create_claimable_balance` omits any balance ID

## Evidence

The transform branch for `create_claimable_balance` never touches the transaction result, even though the XDR success union has a dedicated `BalanceId` field. The repository's own operation tests currently assert that successful create rows contain only `asset`, `amount`, and `claimants`, which means the omission is not just theoretical — it is the present export contract.

## Anti-Evidence

The checked-in test fixture for `CreateClaimableBalanceResult` does not populate `BalanceId`, so the repository may be intentionally mirroring an older or Horizon-style operation schema that omits it. If final review treats this as a schema-extension request instead of a wrong-value bug, it may be rejected despite the result field being available in modern XDR.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the `create_claimable_balance` case in both `extractOperationDetails` functions (lines 881–885 and 1537–1549) in `internal/transform/operation.go`. Both extract only request-body fields (`asset`, `amount`, `claimants`) and do not read the operation result's `BalanceId`. Then traced the upstream Horizon `operations_processor.go` (lines 537–549 in `go@v0.0.0-20250729094549`), which implements the identical behavior — Horizon also only extracts `asset`, `amount`, and `claimants` for `create_claimable_balance` and does not include the result `BalanceId`.

### Code Paths Examined

- `internal/transform/operation.go:881-885` — `create_claimable_balance` branch extracts only `asset`, `amount`, `claimants` from operation body
- `internal/transform/operation.go:1537-1549` — second variant (Horizon-mirroring path) does exactly the same extraction
- `go@v0.0.0-20250729094549/services/horizon/internal/ingest/processors/operations_processor.go:537-549` — upstream Horizon code also omits `balance_id` from `create_claimable_balance` details

### Why It Failed

This is **working-as-designed behavior**, not a data correctness bug. The stellar-etl code faithfully mirrors the upstream Horizon operation detail schema, which intentionally extracts only the request payload for `create_claimable_balance` operations. The Horizon API has maintained this schema since the operation type was introduced. The omission of `balance_id` from the create operation is a deliberate schema design choice, not an oversight. Adding it would be a schema extension/feature request, not a bug fix.

### Lesson Learned

When evaluating missing fields in operation details, always check the upstream Horizon `operations_processor.go` to determine whether the omission is intentional schema design. The stellar-etl operation transform is designed to mirror Horizon's output schema, so "missing" fields that Horizon also omits are by-design, not bugs.
