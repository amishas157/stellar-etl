# H003: `create_contract` and `create_contract_v2` ignore the executable's Wasm hash

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: contract metadata corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `create_contract` or `create_contract_v2` uses a Wasm executable, the exported `contract_code_hash` should equal `args.Executable.WasmHash` for that operation. In a transaction that creates two contracts from different Wasm hashes, each operation row should preserve its own executable hash.

## Mechanism

`CreateContractArgs` and `CreateContractArgsV2` already carry a `ContractExecutable`, and the Wasm arm stores the exact `WasmHash` the operation is creating from. The transform never reads `args.Executable`; it instead scans the transaction-wide footprint for the first `ContractCode` ledger key, so single-operation creates can export `""` and multi-operation creates can silently inherit a sibling operation's hash.

## Trigger

Submit a successful `create_contract` or `create_contract_v2` transaction with `Executable.Type == CONTRACT_EXECUTABLE_WASM`. A strong trigger is a transaction containing two create operations with different Wasm hashes, then exporting operations and checking whether both rows report the same `contract_code_hash`.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:1099-1127` — both create-operation paths populate `contract_code_hash` from the transaction footprint
- `internal/transform/operation.go:(*transactionOperationWrapper).Details:1721-1749` — duplicate operation-detail path with the same behavior
- `internal/transform/operation.go:contractCodeHashFromTxEnvelope:1841-1883` — helper returns the first footprint `ContractCode` hash, not the operation executable hash
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:28831-28916` — both create-operation argument structs include `Executable`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:56401-56460` — `ContractExecutable` stores a `WasmHash` on the Wasm arm

## Evidence

The create-operation branches read `args.ContractIdPreimage` and constructor parameters, but never inspect `args.Executable` before writing `contract_code_hash`. Upstream XDR makes the authoritative per-operation Wasm hash available directly on the host-function args, so using the first footprint `ContractCode` entry is an unnecessary and lossy indirection.

## Anti-Evidence

For stellar-asset executables, there is no Wasm hash to export, so this bug is limited to Wasm-backed contract creation. A single-operation transaction whose footprint contains exactly one matching `ContractCode` key may also export the right value by accident.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (H029 covers `ledger_key_hash` footprint scoping, H031 covers `upload_wasm` empty hash; neither addresses `create_contract` with multiple ContractCode entries)

### Trace Summary

Traced both `extractOperationDetails` (line 1099) and `Details` (line 1721) for `create_contract` and `create_contract_v2`. Both obtain `args` from `op.HostFunction.MustCreateContract()` / `.MustCreateContractV2()` — which carries `args.Executable.WasmHash` — but never read it. Instead both call `contractCodeHashFromTxEnvelope(transactionEnvelope)`, which iterates ReadOnly then ReadWrite footprint keys and returns the **first** `ContractCode` hash found. When only one ContractCode exists in the footprint, this is coincidentally correct. When multiple exist (contract-as-deployer auth, v2 constructor invoking other contracts), the first by footprint sort order is returned, which may not match the operation's `args.Executable.WasmHash`.

### Code Paths Examined

- `internal/transform/operation.go:1099-1113` — `create_contract` case: obtains `args` with `MustCreateContract()`, reads `args.ContractIdPreimage` but ignores `args.Executable`; calls `contractCodeHashFromTxEnvelope` for `contract_code_hash`
- `internal/transform/operation.go:1119-1127` — `create_contract_v2` case: obtains `args` with `MustCreateContractV2()`, ignores `args.Executable`; identical footprint-scanning approach
- `internal/transform/operation.go:1721-1735` — duplicate `create_contract` path in `Details()` method; same pattern
- `internal/transform/operation.go:1741-1749` — duplicate `create_contract_v2` path in `Details()` method; same pattern
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope`: iterates ReadOnly footprint keys first, then ReadWrite; returns the FIRST `ContractCode` hash via `contractCodeFromContractData`
- `internal/transform/operation.go:1876-1884` — `contractCodeFromContractData`: calls `GetContractCode()` on a LedgerKey and returns `Hash.HexString()`
- XDR `CreateContractArgs` (line 28836): struct has `Executable ContractExecutable` field carrying `WasmHash *Hash` on the Wasm arm — the authoritative per-operation source
- `internal/transform/test_variables_test.go:29-52` — test envelope has empty footprint (`ReadOnly: []`, `ReadWrite: []`) and `Executable: xdr.ContractExecutable{}`, so tests always get `contract_code_hash: ""` — never exercising the Wasm case

### Findings

**Core bug confirmed**: The code uses `contractCodeHashFromTxEnvelope()` (a lossy footprint scan) instead of reading `args.Executable.WasmHash` directly. The authoritative per-operation value is available but unused.

**Multi-operation trigger is impossible**: The Stellar protocol enforces exactly one operation per Soroban transaction. The hypothesis's "two create operations with different Wasm hashes" trigger cannot occur on-chain. This corrects the trigger description but does not invalidate the finding.

**Actual trigger — multiple ContractCode entries in a single operation's footprint**: A `create_contract` where the deployer is itself a contract (via `ContractIdPreimage::FromAddress` with a contract address) requires the deployer's `ContractCode` in ReadOnly for auth verification (`__check_auth`), plus the target Wasm's `ContractCode` also in ReadOnly. The helper returns whichever ContractCode appears first by footprint sort order, which is lexicographic on the hash bytes — effectively random from the developer's perspective. If the deployer's Wasm hash sorts before the target's, the wrong hash is exported.

**create_contract_v2 amplifies the risk**: Constructors can invoke other contracts, adding more ContractCode entries to ReadOnly. Each additional entry increases the chance the first-found ContractCode is not the target.

**Test coverage gap**: All test fixtures use empty footprints and empty `Executable` structs, so the tests never exercise the Wasm executable path and always expect `contract_code_hash: ""`.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct a `create_contract` operation with `Executable.Type = CONTRACT_EXECUTABLE_WASM` and a known `WasmHash` (e.g., `0xBBBB...`). Build a `TransactionV1Envelope` whose ReadOnly footprint contains TWO `ContractCode` entries: one for a deployer contract (hash `0xAAAA...` which sorts before `0xBBBB...`) and one for the target Wasm (hash `0xBBBB...`). Use `ContractIdPreimage::FromAddress` with a contract ScAddress.
- **Steps**: Call `extractOperationDetails` (or the `Details` method) on the constructed operation and extract `details["contract_code_hash"]`.
- **Assertion**: Assert that `details["contract_code_hash"]` equals the deployer's hash (`0xAAAA...HexString`) NOT the target Wasm hash (`0xBBBB...HexString`), demonstrating the bug. Then show that `args.Executable.MustWasmHash().HexString()` returns the correct target hash. The fix would replace the footprint scan with a direct read of `args.Executable` when `Executable.Type == ContractExecutableTypeContractExecutableWasm`.
