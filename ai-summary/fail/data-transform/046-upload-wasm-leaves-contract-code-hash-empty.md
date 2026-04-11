# H003: `upload_wasm` details leave `contract_code_hash` empty despite carrying the Wasm blob

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For successful `invoke_host_function` rows whose `details.type` is `upload_wasm`, `details.contract_code_hash` should equal the hash of the uploaded Wasm bytes. That value is the canonical identifier later used by `contract_code` ledger entries, so the operation export should preserve it directly from the host function payload.

## Mechanism

The `upload_wasm` branch ignores `op.HostFunction.Wasm` and instead calls `contractCodeHashFromTxEnvelope(transactionEnvelope)`, which only scans footprint `ContractCode` ledger keys. An upload operation already carries the exact Wasm bytes inline in the XDR host function, but a fresh upload often has no preexisting `ContractCode` key in the footprint, so the helper returns `""` and the exporter silently drops the code hash.

## Trigger

Export any successful Soroban `upload_contract_wasm` transaction. The emitted `history_operations.details.contract_code_hash` will be empty even though the operation body contains the uploaded Wasm and the resulting `contract_code` ledger entry is keyed by that hash.

## Target Code

- `internal/transform/operation.go:1114-1118` - `upload_wasm` branch sets `contract_code_hash` from footprint scan
- `internal/transform/operation.go:1841-1857` - helper only returns hashes from footprint `ContractCode` keys
- `internal/transform/operation_test.go:2059-2073` - current expected output anchors `contract_code_hash: ""`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:29116-29126` - `HostFunction` includes inline `opaque wasm<>` for uploads
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:8162-8182` - `ContractCodeEntry` stores both `Hash` and `Code`

## Evidence

The repository's own operation fixtures expect blank `contract_code_hash` for `upload_wasm` rows (`internal/transform/operation_test.go:2059-2073`), showing the live output currently drops it. The XDR definitions make the missing value concrete: the upload operation has raw Wasm bytes in `HostFunction.Wasm`, and the persisted `ContractCodeEntry` schema defines a dedicated `Hash` field for that code object.

## Anti-Evidence

This hypothesis assumes the contract-code hash is the canonical digest of the uploaded Wasm bytes; while that matches the ledger-entry schema, the exact hashing step is not performed in this package today and would need reviewer confirmation against core semantics. It is also possible for some envelopes to include a matching `ContractCode` footprint key, in which case the current helper could coincidentally return a non-empty value.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (related but distinct from success/data-transform/015 which covers `create_contract`, not `upload_wasm`)
**Failed At**: reviewer

### Trace Summary

Traced `contractCodeHashFromTxEnvelope` (operation.go:1841-1857) through `contractCodeFromContractData` (operation.go:1876-1884). The helper scans `ReadOnly` then `ReadWrite` footprint entries for `LedgerKeyContractCode` keys and returns the first hash found. For `upload_wasm` operations in the Soroban protocol, the newly created `ContractCode` ledger entry — keyed by the SHA-256 hash of the uploaded Wasm — MUST be declared in the `ReadWrite` footprint, because the Soroban protocol requires exhaustive footprint declaration of all entries created or modified. Therefore, the helper WILL find the hash in the `ReadWrite` section and return a non-empty value in production.

### Code Paths Examined

- `internal/transform/operation.go:1114-1118` — `upload_wasm` branch calls `contractCodeHashFromTxEnvelope(transactionEnvelope)` and stores result in `details["contract_code_hash"]`
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope` iterates `ReadOnly` first (lines 1842-1847), then `ReadWrite` (lines 1849-1854), returning first `ContractCode` hash found
- `internal/transform/operation.go:1876-1884` — `contractCodeFromContractData` calls `ledgerKey.GetContractCode()`, extracts the `Hash` field, and returns its hex string
- `internal/transform/test_variables_test.go:81-92` — `genericLedgerTransaction` uses `genericBumpOperationEnvelope` which has NO `SorobanData` set, meaning the footprint is empty in tests
- `internal/transform/operation_test.go:2059-2073` — test expects `contract_code_hash: ""` because the fixture lacks Soroban footprint data, NOT because production transactions would be empty

### Why It Failed

The hypothesis's core claim — "a fresh upload often has no preexisting `ContractCode` key in the footprint" — is factually incorrect about the Soroban protocol. In Soroban, every transaction must declare its complete footprint via preflight simulation. An `upload_wasm` operation creates a new `ContractCode` ledger entry, and that entry MUST appear in the `ReadWrite` footprint section. Since `contractCodeHashFromTxEnvelope` scans both `ReadOnly` and `ReadWrite`, it will find the `ContractCode` key and return the correct hash in production.

Furthermore, unlike `create_contract` (where multiple `ContractCode` entries can exist in the footprint, causing the "wrong first" bug documented in success/data-transform/015), `upload_wasm` operations typically have exactly one `ContractCode` entry in the footprint — the one being uploaded. So the footprint-based approach returns the correct hash for `upload_wasm`.

The test evidence (`contract_code_hash: ""`) is misleading because the test fixture uses `genericBumpOperationEnvelope` which has no `SorobanData` set — meaning no footprint at all. This is a test-fixture limitation, not evidence of production behavior. This is the same pattern established in fail/data-transform/045 for `create_contract`.

### Lesson Learned

The Soroban protocol requires exhaustive footprint declaration — every ledger entry accessed, created, or modified must be in the footprint. For `upload_wasm`, the `ContractCode` entry keyed by the Wasm hash is always in `ReadWrite`. Test fixtures with empty Soroban footprints produce empty results but do not reflect production transaction structure. Before claiming a footprint-scanning helper returns empty for an operation type, verify whether the target entry is present in production footprints.
