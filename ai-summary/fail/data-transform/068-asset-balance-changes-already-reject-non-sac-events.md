# H068: `asset_balance_changes` already rejects non-SAC transfer-shaped events

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`invoke_contract.asset_balance_changes` should only include genuine Stellar
Asset Contract events. A custom contract that emits `transfer`/`mint`-shaped
events should not be able to populate those balance-change rows unless the event
really corresponds to the claimed asset contract.

## Mechanism

At first glance, the helper looked like it might trust any structurally
transfer-shaped contract event because it loops over diagnostic events and
blindly appends parsed SAC events into `asset_balance_changes`. If the parser
only matched topic names and i128 payloads, arbitrary custom contracts could
masquerade as SAC balance changes.

## Trigger

Export `history_operations` for a custom contract invocation that emits
`transfer`, `mint`, `clawback`, or `burn`-shaped contract events whose topic
layout resembles a Stellar Asset Contract event, then inspect
`asset_balance_changes`.

## Target Code

- `internal/transform/operation.go:1952-1970` — helper appends balance changes for every successfully parsed SAC event
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/event.go:61-145` — parser validates the event's contract ID against the canonical asset-derived contract ID before returning a SAC event
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/contractevents/transfer.go:18-24` — transfer parser assumes the asset↔contract relationship was already validated upstream

## Evidence

The helper itself performs no local filtering beyond "parser returned nil error,"
so whether this is viable depends entirely on the strength of the upstream
`contractevents` integrity check.

## Anti-Evidence

`contractevents.NewStellarAssetContractEvent()` performs a definitive integrity
check: it parses the canonical asset topic, derives the expected asset contract
ID for the current network passphrase, and rejects any event whose `ContractId`
does not match. That means non-SAC contracts cannot reach the append path merely
by copying SAC-like topic names.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The apparent trust gap is closed inside the upstream parser. Only events whose
embedded canonical asset topic hashes back to the emitting contract ID are
accepted as Stellar Asset Contract events, so transfer-shaped custom contract
events are filtered out before the ETL appends anything.

### Lesson Learned

When a transform helper delegates classification to an upstream parser, inspect
the parser's integrity checks before assuming "shape-only" matching. Here the
asset↔contract ID validation in `NewStellarAssetContractEvent()` eliminates the
forgery path entirely.
