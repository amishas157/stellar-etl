# H001: Operation `details` emit muxed IDs as JSON numbers instead of exact strings

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: identifier precision / JSON contract drift
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_operations` writes operation `details` for muxed-account operations, the `*_muxed_id` fields should preserve the exact 64-bit Stellar ID in the exported JSON shape, e.g. `"from_muxed_id":"9007199254740993"`. That matches the canonical Horizon operation schema and keeps the ID lossless for downstream generic JSON consumers.

## Mechanism

`addAccountAndMuxedAccountDetails()` inserts the muxed ID into a `map[string]interface{}` as a raw `uint64`, so `ExportEntry()` later marshals it as a JSON number rather than a quoted string. Horizon models the same fields with `json:",string"` tags; any legitimate muxed account whose ID exceeds `2^53` therefore leaves stellar-etl with the wrong JSON type, and common downstream decoders that treat JSON numbers as IEEE-754 doubles will round the identifier even though the ledger input was exact.

## Trigger

1. Submit any operation that populates `details` through `addAccountAndMuxedAccountDetails()` with a muxed account ID above `9007199254740992`, for example a payment from or to `M...` with ID `9007199254740993`.
2. Run `export_operations` across that ledger.
3. Observe that the JSON row contains `from_muxed_id`, `to_muxed_id`, `trustor_muxed_id`, `trustee_muxed_id`, `account_muxed_id`, `into_muxed_id`, `claimant_muxed_id`, or `begin_sponsor_muxed_id` as bare JSON numbers instead of quoted decimal strings.

## Target Code

- `internal/transform/operation.go:addAccountAndMuxedAccountDetails:423-438` — inserts `muxedAccountId` into the generic map as raw `uint64`
- `internal/transform/operation.go:extractOperationDetails:596-611` — payment/create-account paths feed the helper into exported operation details
- `internal/transform/operation.go:extractOperationDetails:817-856` — trust/account-merge paths feed the same helper
- `internal/transform/operation.go:extractOperationDetails:895-929` — claimant / sponsorship / clawback paths feed the same helper

## Evidence

The helper writes `result[prefix+"muxed_id"] = muxedAccountId` directly into the exported `details` map. The upstream Horizon operation resource types tag these exact fields as strings (`source_account_muxed_id`, `funder_muxed_id`, `from_muxed_id`, `to_muxed_id`, `trustor_muxed_id`, `trustee_muxed_id`, `account_muxed_id`, `into_muxed_id`, `claimant_muxed_id`, `begin_sponsor_muxed_id`) in `protocols/horizon/operations/main.go`, which is strong evidence that the canonical JSON contract is string-valued precisely to avoid 64-bit JSON-number ambiguity.

## Anti-Evidence

The paired `*_muxed` address field is also exported, so a downstream system that reparses `M...` addresses can recover the ID indirectly. Some JSON stacks also preserve large integers without rounding. But this operation `details` path is still a live outlier against Horizon's documented JSON shape and against other stellar-etl surfaces such as token transfers that already stringify muxed IDs.
