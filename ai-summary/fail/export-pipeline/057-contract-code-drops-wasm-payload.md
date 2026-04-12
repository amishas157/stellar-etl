# H001: Contract-code export drops the deployed Wasm payload

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-contract-code` exports a
`LedgerEntryTypeContractCode` row, the output should preserve the contract code
entry's defining payload: `ContractCodeEntry.Code`. A downstream consumer
should be able to recover the exact deployed Wasm bytes from the exported row,
not just a hash and derived cost counters.

## Mechanism

`TransformContractCode()` reads `contractCode.Hash` and `contractCode.Ext`, but
never reads `contractCode.Code` at all. The JSON schema has a commented-out
`contract_code` field, and the Parquet schema / converter have no destination
for the payload either, so every exported contract-code row silently loses the
bytecode that the ledger entry actually stores.

## Trigger

1. Run `export_ledger_entry_changes --export-contract-code` on any ledger batch
   containing a `LedgerEntryTypeContractCode` change.
2. Compare the emitted row against the source XDR `ContractCodeEntry`.
3. The source contains non-empty `Code []byte`, but the exported JSON and
   Parquet rows contain only `contract_code_hash`, cost inputs, and ledger-key
   metadata with no field carrying the Wasm payload.

## Target Code

- `internal/transform/contract_code.go:12-99` — transforms contract-code rows
  but never reads `contractCode.Code`
- `internal/transform/schema.go:540-560` — `ContractCodeOutput` has no live
  `contract_code` field; the only candidate field is commented out
- `internal/transform/schema_parquet.go:284-303` — Parquet schema exposes no
  Wasm-payload column
- `internal/transform/parquet_converter.go:316-338` — Parquet conversion copies
  the existing metadata fields only
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:8178-8182`
  — `ContractCodeEntry` includes `Code []byte`

## Evidence

The source XDR row is unambiguous: `ContractCodeEntry` has three members
(`Ext`, `Hash`, and `Code`), and the exporter only uses the first two. The
schema comment says this row "aligns with the Bigquery table
`soroban_contract_code`", which makes the absence of the contract code payload
stand out as a structural omission rather than a formatting choice.

## Anti-Evidence

The repository may have intended `soroban_contract_code` to behave as a
metadata-only table keyed by `contract_code_hash`, and the commented-out field
suggests the omission could be a deliberate size tradeoff. But there is no
README, schema comment, or code comment that documents such an exclusion, and
the commented-out field is equally consistent with an unfinished export path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Success entry 014 covers the contract-code Parquet missing `ledger_key_hash_base64` (a different field omission). Fail entry 052 covers `upload_wasm` footprint ordering (a different code path). Neither addresses the Wasm payload omission from `ContractCodeOutput`.
**Failed At**: reviewer

### Trace Summary

I traced the full `TransformContractCode()` function (contract_code.go:12-101), the `ContractCodeOutput` struct (schema.go:540-561), `ContractCodeOutputParquet` (schema_parquet.go:284-303), and the `ToParquet()` converter (parquet_converter.go:316-337). The function reads `contractCode.Hash` and `contractCode.Ext` (including cost inputs) but never accesses `contractCode.Code`. The schema has a commented-out field `//ContractCodeCode string \`json:"contract_code"\`` at line 549, confirming the field was explicitly considered and excluded. The XDR struct `ContractCodeEntry` (xdr_generated.go:8178-8182) does include `Code []byte`.

### Code Paths Examined

- `internal/transform/contract_code.go:12-101` — `TransformContractCode()` accesses `contractCode.Hash` (line 45), `contractCode.Ext` (lines 43, 65-77), but never `contractCode.Code`
- `internal/transform/schema.go:539-561` — `ContractCodeOutput` struct has 18 fields; line 549 is `//ContractCodeCode string \`json:"contract_code"\`` — explicitly commented out
- `internal/transform/schema_parquet.go:284-303` — `ContractCodeOutputParquet` has no Wasm payload column
- `internal/transform/parquet_converter.go:316-337` — `ToParquet()` maps only existing fields, no Code field
- XDR `ContractCodeEntry` (xdr_generated.go:8178-8182) — includes `Code []byte` as third member

### Why It Failed

This describes **working-as-designed behavior**, not a bug. The commented-out field `//ContractCodeCode` at schema.go:549 is strong evidence that the Wasm payload was explicitly considered and deliberately excluded from the export schema. This is a design tradeoff, not an oversight:

1. **Deliberate exclusion**: The commented-out field proves a developer wrote the field, considered it, and chose not to include it. This is not a case of "forgetting" an XDR member.
2. **Reasonable design rationale**: WASM contract code can be hundreds of KB per entry. Including raw bytecode in every JSON/Parquet row would dramatically increase export sizes with limited analytical value — the `contract_code_hash` uniquely identifies the code and the 10 cost-input fields provide the analytics metadata that BigQuery consumers need.
3. **The struct is clearly a metadata/analytics table**: With 10 cost-input fields (NInstructions, NFunctions, NGlobals, etc.) and a hash identifier, the schema is designed for contract complexity analytics, not bytecode storage.
4. **The hypothesis's "expected behavior" is an assumption**: There is no documentation, issue, or TODO establishing that the export SHOULD include raw Wasm bytes. The claimed expected behavior ("A downstream consumer should be able to recover the exact deployed Wasm bytes") is the hypothesis author's expectation, not the code's design intent.

### Lesson Learned

A commented-out field in a schema struct is evidence of a deliberate design decision, not a bug. When a hypothesis claims an XDR field is "dropped," check whether the schema explicitly considered and excluded it. Not every XDR member needs to appear in every export table — size tradeoffs for large binary payloads (like WASM bytecode) are a valid design choice, especially when a hash identifier preserves referential integrity.
