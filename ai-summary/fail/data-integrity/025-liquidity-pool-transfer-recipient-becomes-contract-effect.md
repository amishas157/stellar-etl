# H001: Liquidity-pool SAC transfer recipients become contract effects

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a Stellar Asset Contract transfer credits a liquidity-pool holder, the exported effect row for that credit should identify the actual `L...` liquidity-pool participant. The row should preserve the credited pool address in the effect entity slot instead of rebinding the credit to the invoking operation's source account, and it should not relabel the pool as a contract.

## Mechanism

`addInvokeHostFunctionEffects()` decides whether the transfer recipient is "account-like" by calling `strkey.IsValidEd25519PublicKey(transferEvent.To)`. Liquidity-pool strkeys begin with `L`, so they always miss that check and fall into the contract fallback, which writes `details["contract"] = transferEvent.To` and then calls `addMuxed(source, EffectContractCredited, ...)`. The exported row therefore shows the operation source account in `address` and `type_string=contract_credited`, even though the credited holder was the liquidity pool.

## Trigger

Export effects for any valid SAC transfer whose recipient is a liquidity pool, such as the upstream liquidity-pool deposit / strict-path fixtures that emit `transferEvent(..., lpIdToStrkey(...), ...)`. The corresponding effect row will attribute the credit to the operation source account while burying the real `L...` pool address under `details.contract`.

## Target Code

- `internal/transform/effects.go:addInvokeHostFunctionEffects:1366-1375` — transfer recipient classification uses `IsValidEd25519PublicKey` and falls back to `addMuxed(source, EffectContractCredited, ...)`
- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents/createSACBalanceChangeEntry:1921-1934,1984-1997` — sibling operation export preserves raw SAC participant strings in `from` / `to`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/scval.go:34-39` — `ScAddress.String()` encodes liquidity-pool participants as `L...`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/processors/token_transfer/token_transfer_processor_test.go:1392-1408` — realistic token-transfer fixtures emit SAC transfers whose recipient is a liquidity pool

## Evidence

The SDK already treats liquidity pools as first-class `ScAddress` values and stringifies them into `L...` strkeys. Upstream token-transfer fixtures exercise real SAC transfer events whose `to` endpoint is a liquidity pool, and the local operation exporter preserves those same participant strings verbatim in `asset_balance_changes`. Only the effects exporter reclassifies the `L...` recipient as a contract and replaces the affected entity with the operation source account.

## Anti-Evidence

The current effect enum set only has `account_*` and `contract_*` debit/credit variants, and upstream Horizon currently mirrors the same `G...`-only split. That could be cited as intentional taxonomy reuse. But even within that taxonomy, overwriting the affected entity with the source account is still structurally wrong because `EffectOutput.Address` is a generic string field and sibling exports already preserve the actual liquidity-pool participant.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of fail/data-integrity/017 (Soroban contract_debited/contract_credited effects misattribute top-level address to source account instead of the contract)
**Failed At**: reviewer

### Trace Summary

Traced the complete code path from `addInvokeHostFunctionEffects` in `effects.go:1342-1429` through the `IsValidEd25519PublicKey` check at line 1366. Confirmed that any non-G... address (including L... liquidity pools) falls into the `else` branch at line 1373, setting `details["contract"] = transferEvent.To` and calling `addMuxed(source, EffectContractCredited, toDetails)`. Then compared against the upstream Horizon effects processor at `stellar/go` `services/horizon/internal/ingest/processors/effects_processor.go:1505-1516` and confirmed the code is an exact mirror — same `IsValidEd25519PublicKey` check, same `EffectContractCredited` fallback with `addMuxed(source, ...)`.

### Code Paths Examined

- `internal/transform/effects.go:1354-1376` — local SAC transfer effect classification: `IsValidEd25519PublicKey` splits G... (account path) from everything else (contract path)
- `stellar/go effects_processor.go:1491-1516` — upstream Horizon uses identical pattern: same `IsValidEd25519PublicKey` guard, same `EffectContractCredited`/`EffectContractDebited` fallback with `addMuxed(source, ...)`
- `go-stellar-sdk xdr/scval.go:34-39` — `ScAddress.String()` encodes `ScAddressTypeLiquidityPool` as L... via `strkey.VersionByteLiquidityPool`
- `go-stellar-sdk support/contractevents/utils.go:15-53` — `parseBalanceChangeEvent` extracts `ScAddress.String()` for both parties, so L... addresses can theoretically appear in `TransferEvent.To`

### Why It Failed

This hypothesis is a specific instance of the pattern already rejected in fail 017. The core mechanism is identical: any non-G... `ScAddress` (whether C... contract or L... liquidity pool) fails the `IsValidEd25519PublicKey` check and falls into the `EffectContractCredited`/`EffectContractDebited` branch, which sets `address=source` and `details.contract=actual_entity`. The upstream Horizon effects processor (`stellar/go` effects_processor.go:1491-1516) uses the **exact same code pattern** — same guard, same fallback, same semantics. Meta-pattern 5 from prior investigations establishes: "Upstream Effect Schema Is the Source of Truth." Additionally, the LP-specific trigger scenario (SAC transfer to an LP address via InvokeHostFunction) is theoretical — the referenced upstream test fixtures (`TestLiquidityPoolEvents`) exercise classic LP deposit/withdraw operations that generate synthetic token transfer events from ledger entry changes, NOT SAC contract events that would flow through `addInvokeHostFunctionEffects`.

### Lesson Learned

The `address=source, details.contract=entity` pattern for non-account SAC participants applies uniformly to all non-G... address types (C... contracts, L... liquidity pools, M... muxed accounts). All are the same design decision mirroring upstream Horizon. Do not resubmit variants of fail 017 for different address types.
