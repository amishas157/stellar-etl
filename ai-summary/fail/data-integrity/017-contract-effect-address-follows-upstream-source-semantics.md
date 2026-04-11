# H017: Soroban contract effects should put the contract itself in top-level `address`

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `contract_debited` and `contract_credited` effects, the top-level `address`
column should identify the contract that sent or received the asset. A
downstream consumer should not need to inspect `details.contract` to discover
which contract the effect is about.

## Mechanism

`addInvokeHostFunctionEffects()` stores the contract address under
`details["contract"]` but calls `addMuxed(source, ...)`, which writes the
operation source account into the top-level `address` column. That initially
looked like silent misattribution because the effect row's primary address and
its nested contract detail can disagree.

## Trigger

1. Export effects for a Soroban SAC transfer, mint, burn, or clawback where one
   side of the event is a contract.
2. Inspect the resulting `contract_debited` / `contract_credited` rows.
3. Observe that the top-level `address` is the transaction source/admin account
   while the contract address appears in `details.contract`.

## Target Code

- `internal/transform/effects.go:1354-1375` - transfer events call
  `addMuxed(source, EffectContractDebited/Credited, ...)`
- `internal/transform/effects.go:1384-1393` - mint-to-contract uses the same
  pattern
- `internal/transform/effects.go:1402-1427` - clawback/burn-from-contract use
  the same pattern
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/effects/effects.go:1488-1554`
  - upstream Stellar processor emits the same shape
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/services/horizon/internal/ingest/processors/effects_processor_test.go:3512-3535`
  - upstream tests expect `address=admin` with `details.contract=<contract>`

## Evidence

The local transform path clearly separates the two identities: it places the
contract in `details["contract"]` and the source/admin account in the top-level
effect `address`. The upstream `stellar/go` processor uses the exact same
`addMuxed(source, ...)` pattern, and the upstream Horizon ingest tests
explicitly assert that `contract_debited` / `contract_credited` rows keep the
source/admin account in `address` while storing the actual contract under the
details map.

## Anti-Evidence

If the intended semantics of the effect schema are "address identifies the
logical subject of the effect," then this shape would indeed look wrong.
However, neither this repository nor the upstream Horizon-compatible processor
implements or tests that interpretation.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is not a local data-integrity bug; it is the established upstream effect
schema. The ETL intentionally mirrors `stellar/go` / Horizon behavior, and the
upstream tests lock in `address=<source/admin>` plus `details.contract=<actual contract>`
for contract effects.

### Lesson Learned

For effect rows, the top-level `address` field does not always mean "the asset
holder being debited or credited." When Soroban contract effects are involved,
verify the upstream Horizon processor and tests before treating a top-level vs
nested identity split as corruption.
