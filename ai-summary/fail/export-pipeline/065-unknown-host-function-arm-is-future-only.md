# H065: Unknown host-function arm panic is future-only

**Date**: 2026-04-14
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For every currently valid `INVOKE_HOST_FUNCTION` operation, the exporter should populate operation details for the concrete host-function arm (`invoke_contract`, `create_contract`, `upload_wasm`, or `create_contract_v2`) and complete the row without panicking.

## Mechanism

At first glance, both invoke-host-function detail formatters look brittle because they `panic` on an unknown `HostFunctionType` instead of returning an error. If a legitimate ledger could currently carry an unhandled host-function arm, the export would abort mid-run instead of emitting correct operation details.

## Trigger

Process an `INVOKE_HOST_FUNCTION` operation whose `HostFunction.Type` is not one of the four currently generated XDR enum values.

## Target Code

- `internal/transform/operation.go:1063-1143` — `extractOperationDetails()` panics on unknown `HostFunctionType`
- `internal/transform/operation.go:1685-1765` — `transactionOperationWrapper.Details()` repeats the same panic fallback
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:28387-28410` — current `HostFunctionType` enum only defines four values

## Evidence

Both transform paths enumerate four host-function arms and end in `panic(fmt.Errorf("unknown host function type: %s", op.HostFunction.Type))`, which would be a live export-killer if current Stellar XDR already exposed an additional arm.

## Anti-Evidence

The generated SDK enum currently defines exactly four values — `InvokeContract`, `CreateContract`, `UploadContractWasm`, and `CreateContractV2` — and both local switches handle all four. No present-day legitimate ledger can reach the panic branch without a future protocol/XDR expansion.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Current Stellar XDR exposes no fifth `HostFunctionType`, so the scary default branch is future-only rather than a present export-pipeline correctness bug.

### Lesson Learned

Default-case panics in enum switches are only viable here when the current generated XDR already contains an unhandled arm. Future-protocol brittleness should be recorded, but it is out of scope for legitimate current-chain data-integrity findings.
