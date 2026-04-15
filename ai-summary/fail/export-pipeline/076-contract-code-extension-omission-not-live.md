# H076: Contract-code export does not currently miss a higher ext arm

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `ContractCodeEntryExt` had already gained a live arm beyond `V1`, `TransformContractCode()` should decode and export the newer cost-input payload instead of silently leaving the derived instruction/function counters at zero.

## Mechanism

`TransformContractCode()` only calls `contractCode.Ext.GetV1()` before populating `n_instructions`, `n_functions`, and the other cost-input fields. That looked like a stale arm-specific parser that could underfill contract-code rows after an SDK/XDR update.

## Trigger

Export contract-code changes from a ledger whose `ContractCodeEntryExt` uses a hypothetical arm above `V1`.

## Target Code

- `internal/transform/contract_code.go:TransformContractCode:54-77` — reads only `Ext.GetV1()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:8009-8067` — current `ContractCodeEntryExt` union definition

## Evidence

The transform is explicitly version-gated to `Ext.GetV1()`, so it would be vulnerable to under-export if the underlying union had already advanced.

## Anti-Evidence

The current generated XDR still defines `ContractCodeEntryExt` with only arms `0` and `1`. There is no `V2` payload for the exporter to ignore.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The trigger depends on a higher extension arm that does not exist in the current dependency set. `TransformContractCode()` fully covers the present `ContractCodeEntryExt` surface.

### Lesson Learned

Version-gated parsing is only a live omission when the upstream union has already moved forward. Generated-XDR inspection is the quickest way to separate a real stale-parser bug from a merely future-sensitive implementation.
