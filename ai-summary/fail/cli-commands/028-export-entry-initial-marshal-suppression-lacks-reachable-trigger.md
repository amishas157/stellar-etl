# H028: `ExportEntry()` initial marshal suppression could emit metadata-only rows

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: metadata-only row emission on JSON marshal failure
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If an export row cannot be JSON-marshaled, `ExportEntry()` should fail that row explicitly and return a non-nil error. A command should never continue by writing `{}` or just the `--extra-fields` metadata for a row whose real payload could not be serialized.

## Mechanism

`ExportEntry()` logs errors from its first `json.Marshal(entry)` and from the subsequent `decoder.Decode(&i)`, but it does not return either error. That means a genuinely non-marshalable `entry` would leave `i` as an empty map, after which the function would still marshal and write that empty map plus any extra metadata fields. If reachable from a legitimate transformed row, this would silently replace real export payloads with metadata-only JSON objects.

## Trigger

Find a legitimate Stellar export row whose concrete Go value makes `json.Marshal(entry)` fail while still allowing the later `json.Marshal(i)` call to succeed. Then run any normal export command that routes the row through `ExportEntry()`.

## Target Code

- `cmd/command_utils.go:ExportEntry:55-75` — suppresses the first marshal/decode failures and continues with `i := map[string]interface{}{}` 
- `internal/transform/schema.go:153-175` — `ClaimableBalanceOutput` includes `xdr.ClaimPredicate`, a plausible complex JSON surface
- `internal/transform/schema.go:520-536` — `ContractDataOutput` exposes several `interface{}` payload fields
- `internal/transform/schema.go:640-656` — `ContractEventOutput` also exposes `interface{}` / `[]interface{}` payload fields
- `internal/transform/contract_data.go:121-156` — current contract-data serializer populates those interface fields with strings / decoded JSON payloads rather than arbitrary Go objects
- `internal/transform/contract_events.go:96-136` — current contract-event serializer likewise normalizes values to base64 strings and `json.RawMessage`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/json_test.go:15-80` — upstream XDR tests show `ClaimPredicate` JSON marshaling succeeds for representative and randomized cases

## Evidence

The error-suppression path is real: `ExportEntry()` logs and discards both the first `json.Marshal(entry)` failure and the following `decoder.Decode(&i)` failure. If `entry` were non-marshalable, the function would still proceed with an empty map and write whatever metadata had been appended through `extra`.

## Anti-Evidence

After tracing the current transform outputs, I did not find a concrete legitimate row shape that can hit this path. The most suspicious schema fields (`interface{}` / `[]interface{}` in contract-data and contract-event outputs, plus `xdr.ClaimPredicate`) are currently normalized to strings or `json.RawMessage`, and the upstream SDK explicitly tests `ClaimPredicate` JSON marshaling successfully.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suppression bug exists in abstract, but I could not identify any legitimate current export type that actually makes `json.Marshal(entry)` fail. Without a chain-backed trigger, this remains a theoretical hardening issue rather than a concrete data-integrity bug.

### Lesson Learned

An error-swallowing path in shared serialization code is only a viable data-integrity finding when at least one production output type can reach it with valid on-chain data. For `ExportEntry()`, the right next step is always to trace the actual concrete values placed into `interface{}` fields before escalating the suppression pattern itself.
