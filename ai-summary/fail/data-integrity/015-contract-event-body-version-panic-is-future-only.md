# H015: Contract-event helpers panic on non-v0 event bodies

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Contract-event export should either decode every legitimate current event body or return a normal error when it encounters an unsupported version, rather than panicking inside helper code.

## Mechanism

`getEventTopics()` and `getEventData()` switch on `eventBody.V` and call `panic(...)` in the default branch. If a new `ContractEventBody` arm ever appeared before stellar-etl was updated, the exporter would crash while parsing otherwise valid contract events instead of surfacing a typed error.

## Trigger

Process a contract event whose `ContractEventBody.V != 0`, then call `TransformContractEvent()` so `parseDiagnosticEvent()` reaches `getEventTopics()` / `getEventData()`.

## Target Code

- `internal/transform/contract_events.go:70-93` — helper switches panic on unsupported body versions
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:16980-17021` — current XDR union exposes only the `case 0` arm for `ContractEventBody`

## Evidence

The helper code undeniably panics for any non-zero discriminant. The generated XDR currently documents `ContractEventBody` with only a single `v0` arm, so the panic path exists only for a future protocol/body version.

## Anti-Evidence

Because the current Stellar XDR defines only `ContractEventBody.V == 0`, legitimate present-day ledger data cannot reach the panic branch. This is a forward-compatibility concern, not a current silent-corruption bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not identify any valid current contract event that uses a non-zero body version. The only reachable behavior today is the supported `v0` path, so this does not produce wrong output for legitimate current chain data.

### Lesson Learned

Future-version panics are worth recording, but they are out of scope for this objective unless the current protocol already emits the newer union arm on legitimate data.
