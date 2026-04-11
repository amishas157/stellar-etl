# H050: Contract-code cost-input counters truncate large Wasm metadata values

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: suspected numeric truncation
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If Soroban contract-code cost inputs can exceed `uint32`, the exporter should preserve the full counter values in `n_instructions`, `n_functions`, `n_globals`, `n_table_entries`, `n_types`, `n_data_segments`, `n_elem_segments`, `n_imports`, `n_exports`, and `n_data_segment_bytes` instead of narrowing them during transformation.

## Mechanism

`TransformContractCode()` explicitly casts every `extV1.CostInputs` field through `uint32(...)`, which initially looked like a likely truncation path. Because the JSON schema uses `uint32` for all ten counters, a wider XDR source type would have produced silent wraparound in both JSON and Parquet exports.

## Trigger

Inspect `contract_code` exports for very large Wasm modules whose parsed cost-input counters might exceed `4294967295`.

## Target Code

- `internal/transform/contract_code.go:54-77` — casts every `CostInputs` member to `uint32`
- `internal/transform/schema.go:540-560` — JSON schema stores the counters as `uint32`
- `internal/transform/schema_parquet.go:293-302` — Parquet widens only after the JSON-layer `uint32` values are fixed

## Evidence

The transform code does contain repeated narrowing-looking casts at every cost-input assignment site, and the schema repeats that `uint32` choice across the exported contract-code row.

## Anti-Evidence

Upstream XDR does not provide a wider source type here. `go doc github.com/stellar/go-stellar-sdk/xdr.ContractCodeCostInputs` shows all ten fields are already `Uint32`, so the casts in `TransformContractCode()` are type-preserving rather than narrowing.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected overflow path is not live because the XDR source struct itself defines every contract-code cost-input counter as `uint32`. Since the source and destination ranges already match, the transform layer cannot truncate any additional high bits here.

### Lesson Learned

Repeated `uint32(...)` casts are only actionable when the upstream XDR member is wider than `Uint32`. For type-audit work in this repo, confirm the generated XDR field type before treating a cast-heavy transform as a live narrowing bug.
