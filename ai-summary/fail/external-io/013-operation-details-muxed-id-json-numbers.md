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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of H003 (fail/external-io/012-trade-effect-seller-muxed-id-json-number-is-schema-defined.md)
**Failed At**: reviewer

### Trace Summary

Traced `addAccountAndMuxedAccountDetails` at operation.go:423-440. Confirmed `GetId()` returns `uint64` and line 437 stores it raw into `map[string]interface{}`. This is the identical root cause function and mechanism as H003 (seller_muxed_id in trade effects), which targeted the same function at the same line. H003 went through full review (VIABLE), PoC (PASS), and was ultimately REJECTED at final review because the ETL emits exact decimal digits and precision loss is a downstream consumer issue, not ETL-side corruption.

### Code Paths Examined

- `internal/transform/operation.go:addAccountAndMuxedAccountDetails:423-440` — identical root cause as H003; `GetId()` returns `uint64`, stored raw at line 437
- `internal/transform/operation.go:extractOperationDetails:596-669` — multiple call sites for payment/path-payment operations, all through the same helper
- `cmd/command_utils.go:ExportEntry:55-86` — `UseNumber()` decoder preserves digits in Go, but final JSON output is still bare number
- `internal/transform/schema.go:676` — `TokenTransferOutput.ToMuxedID` is `null.String`, showing the codebase is inconsistent, but this was already evaluated in H003

### Why It Failed

This hypothesis targets the same root cause function (`addAccountAndMuxedAccountDetails:437`), the same mechanism (raw `uint64` in `map[string]interface{}` → bare JSON number), and the same precision concern as H003 (fail/external-io/012). H003 was fully investigated through PoC and rejected at final review because: (1) the ETL emits exact decimal digits — the JSON text `9007199254740993` is correct, (2) precision loss occurs only in downstream consumers that choose float64-based JSON decoders, (3) this is a JSON interoperability concern, not ETL-side data corruption, and (4) comparing to Horizon's `json:",string"` tags is comparing two separate projects' contracts. These rejection reasons apply identically to the operation details path — it's a different call site of the same function with the same output behavior.

### Lesson Learned

When a root-cause function has already been investigated and rejected, hypotheses targeting different call sites of the same function with the same mechanism are duplicates. The operation details path through `addAccountAndMuxedAccountDetails` is not meaningfully distinct from the trade effects path — both store `uint64` in a `map[string]interface{}` and produce identical JSON output behavior.
