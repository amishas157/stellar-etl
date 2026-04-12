# H024: `invoke_host_function.details.function` should contain the invoked contract method name

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `details.type="invoke_contract"` rows, a field named `function` would
normally be expected to carry the Soroban contract method symbol being invoked
(for example `transfer` or `mint`), not the outer host-function enum.

## Mechanism

`extractOperationDetails()` sets `details["function"] = op.HostFunction.Type.String()`
before branching into the specific host-function subtype, so `invoke_contract`
rows export `HostFunctionTypeHostFunctionTypeInvokeContract` instead of the
actual contract method symbol. That initially looked like a mislabeled field
because the real function name is present in `invokeArgs.FunctionName`.

## Trigger

1. Export an `invoke_host_function` operation whose host function is
   `invoke_contract`.
2. Inspect `history_operations.details.function`.
3. The row exports the host-function enum string rather than the Soroban method
   symbol.

## Target Code

- `internal/transform/operation.go:1063-1088` — `extractOperationDetails()`
  assigns `details["function"]` from the host-function enum before processing
  `invoke_contract`
- `internal/transform/operation.go:1685-1710` — duplicate
  `transactionOperationWrapper.Details()` path does the same
- `internal/transform/operation_test.go:1854-1900` — checked-in expectations
  assert the enum-string value for `invoke_contract`

## Evidence

The live code has direct access to `invokeArgs.FunctionName`, but it is only
serialized inside the `parameters` / `parameters_json` arrays. The separate
`function` field therefore reads like a mislabeled duplicate of `type`.

## Anti-Evidence

The repository's operation tests explicitly assert the current enum-string value
for `details.function` on `invoke_contract`, and the transform intentionally
serializes the actual method symbol inside the parameter payload. That points to
an established local contract rather than an accidental regression.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The current repository contract already treats `details.function` as the
host-function enum label, not the invoked Soroban method name. The checked-in
operation tests lock that behavior in, and the actual method symbol is still
preserved elsewhere in the exported parameter arrays.

### Lesson Learned

Do not infer field semantics from the key name alone when the repo has explicit
goldens or unit tests for that surface. For host-function operation details,
the tests are the authoritative contract, even when the field naming is
somewhat misleading.
