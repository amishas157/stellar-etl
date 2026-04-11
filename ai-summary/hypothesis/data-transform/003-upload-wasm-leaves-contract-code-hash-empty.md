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
