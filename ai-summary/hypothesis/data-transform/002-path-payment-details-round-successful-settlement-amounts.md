# H002: Path-payment details round successful settlement amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Successful path-payment rows should preserve the exact settlement amounts implied by the XDR result. For a large successful `path_payment_strict_send`, `details.amount` should equal the exact delivered amount, and for a large successful `path_payment_strict_receive`, `details.source_amount` should equal the exact send amount; 1-stroop differences should survive export.

## Mechanism

These branches already carry exact decimal strings for some sibling fields (`destination_min`, default `amount`, and wrapper-side `amount.String(...)` output), but `extractOperationDetails()` overwrites the actual successful settlement values with `utils.ConvertStroopValueToReal()`. The detail map therefore mixes exact-string and rounded-float encodings inside the same operation family, and large valid settlement values lose low-order digits even though an exact string representation is already available in-package.

## Trigger

Process a successful path payment whose executed send or destination amount is large enough to exceed float64's exact decimal range after 7-digit scaling, for example:

1. `PathPaymentStrictSendResult.DestAmount = 90071992547409931`, or
2. `PathPaymentStrictReceiveResult.SendAmount = 90071992547409931`

Compare the exported `details.amount` / `details.source_amount` against `amount.String(...)` or against an adjacent 1-stroop value; the exported numbers will round together.

## Target Code

- `internal/transform/operation.go:619-656` — strict-receive branch converts `amount`, `source_max`, and successful `source_amount` through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:660-699` — strict-send branch converts `source_amount` and successful `amount` through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1379-1416` — sibling wrapper uses exact `amount.String(...)` values for both strict-receive and strict-send details
- `internal/transform/schema.go:136-150` — operation details are not constrained to a float-only schema

## Evidence

Within these same branches, some bounds are already exported as exact strings (`destination_min`, the initial zero-valued placeholders, and the wrapper's exact output), proving the payload can represent decimal strings today. The only reason successful settlement amounts become lossy is the final `ConvertStroopValueToReal()` choice applied after the XDR result has already provided an exact stroop integer.

## Anti-Evidence

There is already a rejected investigation about failed-path-payment zero defaults; this hypothesis is different because it targets successful-result precision loss, not failure semantics. Existing fixtures use mid-sized values like `894.6764349` and `433.4043858`, which do not cross the float-collision threshold, so tests will not reveal the issue.
