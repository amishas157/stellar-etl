# H017: Ledger-key hash helpers do not silently drop live hashes on valid exports

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a ledger key cannot be derived or serialized, the relevant export path should fail rather than silently emitting an empty or truncated hash field. JSON and Parquet rows should not keep flowing with missing `ledger_key_hash` values for otherwise valid contract or Soroban-footprint records.

## Mechanism

`LedgerEntryToLedgerKeyHash()` and `LedgerKeyToLedgerKeyHash()` ignore `LedgerKey()` / `MarshalBinary()` errors, which initially looked like a silent-corruption risk. In particular, `ledgerKeyHashFromTxEnvelope()` filters out empty strings instead of surfacing errors, so a serialization failure would appear to remove one footprint key from the exported array without aborting the row.

## Trigger

Process a legitimate ledger entry or transaction footprint whose `xdr.LedgerKey` cannot be marshaled, while exporting contract-code, contract-data, or Soroban operation details.

## Target Code

- `internal/utils/main.go:LedgerEntryToLedgerKeyHash:978-984` — ignores `LedgerKey()` and `MarshalBinary()` errors
- `internal/utils/main.go:LedgerKeyToLedgerKeyHash:1043-1048` — ignores `MarshalBinary()` errors on a concrete `xdr.LedgerKey`
- `internal/transform/contract_code.go:28-40` — immediately recomputes and hard-fails on `ledgerEntry.LedgerKey()` / `xdr.MarshalBase64(ledgerKey)`
- `internal/transform/contract_data.go:65-76` — does the same for contract-data rows
- `internal/transform/operation.go:1859-1873` — silently skips empty footprint hashes, but starts from already-materialized `xdr.LedgerKey` values in a validated transaction envelope

## Evidence

The utility helpers really do discard serialization errors, and the operation-details path treats an empty return as "no hash" rather than as an error. In isolation that looks like a classic silent-drop bug for identifier fields.

## Anti-Evidence

For contract-data and contract-code rows, the neighboring transform code immediately re-derives the ledger key and returns an error if that step fails, so a broken live key cannot survive into output merely because the helper swallowed an earlier error. For operation footprints, the inputs are already concrete `xdr.LedgerKey` values decoded from validated transaction envelopes; I did not find a realistic legitimate-ledger shape where `MarshalBinary()` fails there without the whole XDR object already being malformed.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The swallowed errors are ugly, but I could not tie them to a live path that emits plausible wrong rows. The contract transforms already fail loudly on the same key-generation step, and the footprint path would require malformed in-memory XDR rather than a legitimate Stellar transaction.

### Lesson Learned

Ignored helper errors are only actionable when no later layer revalidates the same value and when a realistic production input can trigger the failure. Here, the surrounding call sites either hard-fail on the same operation or consume already-validated ledger-key unions.
