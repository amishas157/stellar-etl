# H056: `create_contract_v2` should export the executable kind explicitly

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details` for `create_contract_v2` should ideally expose the
operation's executable kind (`Wasm` vs `StellarAsset`) so downstream consumers
can distinguish builtin asset deployments from Wasm deployments whose
`contract_code_hash` happens to be empty.

## Mechanism

The create-contract-v2 branches serialize constructor args and preimage details
but never read `args.Executable.Type`, even though the XDR union has multiple
arms. That makes `StellarAsset` deployments structurally ambiguous in the
exported row because there is no explicit field telling consumers which
executable arm was used.

## Trigger

1. Export an operation whose host function is `create_contract_v2` and whose
   `Executable.Type` is `ContractExecutableTypeContractExecutableStellarAsset`.
2. Inspect the resulting `history_operations.details`.
3. The row has no field that directly records the executable type.

## Target Code

- `internal/transform/operation.go:1119-1139` — free-function
  `create_contract_v2` detail builder
- `internal/transform/operation.go:1741-1761` — duplicated method-receiver path
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:28913-28916`
  — `CreateContractArgsV2` contains `Executable ContractExecutable`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:56389-56425`
  — `ContractExecutable` has distinct `Wasm` and `StellarAsset` arms

## Evidence

The XDR union clearly carries the executable kind, and the exporter clearly does
not. Asset-based `create_contract_v2` expectations in `operation_test.go`
already show rows with an empty `contract_code_hash` and no explicit executable
indicator, so the ambiguity is visible in current repository fixtures.

## Anti-Evidence

The codebase has no explicit schema field, test assertion, or documentation
stating that `history_operations.details` must preserve the executable kind
itself rather than only derived identifiers such as `contract_code_hash`. That
makes this look more like a desirable schema extension than a defensible
incorrect-output bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not establish an in-repo contract that requires the operation-details
schema to expose the executable arm explicitly. The current evidence supports
"useful missing field" more strongly than "wrong existing output value".

### Lesson Learned

For schema-omission hypotheses, a live upstream field is not enough by itself:
there also needs to be a clear local contract that the exporter intends to
surface that field rather than a narrower derived summary.
