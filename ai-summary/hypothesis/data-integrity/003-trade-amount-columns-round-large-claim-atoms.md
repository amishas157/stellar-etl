# H003: `history_trades` rounds exact claim-atom amounts in `selling_amount` and `buying_amount`

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each trade row should preserve the exact sold and bought amounts represented by the underlying claim atom. Two valid trades whose `AmountSold` or `AmountBought` differ by 1 stroop should export different `selling_amount` / `buying_amount` values.

## Mechanism

`TransformTrade()` reads exact `Int64` amounts from `ClaimAtom.AmountSold()` and `ClaimAtom.AmountBought()`, then converts both to `float64` with `utils.ConvertStroopValueToReal()` when building `TradeOutput`. Once the amounts are large enough, distinct claim atoms collapse to the same JSON number even though these are the authoritative monetary columns of `history_trades`.

## Trigger

Run `TransformTrade()` on two otherwise identical successful trade-producing operations whose claim atom differs by 1 stroop in `AmountSold` or `AmountBought`, e.g. `90071992547409930` versus `90071992547409931`. Inspect `selling_amount` or `buying_amount`: the current output should serialize the same rounded number for both trades.

## Target Code

- `internal/transform/trade.go:64-67` — reads exact bought amount from the claim atom
- `internal/transform/trade.go:139-145` — writes `selling_amount` and `buying_amount` via `ConvertStroopValueToReal`
- `internal/transform/schema.go:285-300` — `TradeOutput` exposes these rounded values as the row's monetary columns
- `internal/utils/main.go:84-87` — conversion path returns `float64`
- `.../xdr/claim_atom.go:51-87` — `ClaimAtom.AmountBought()` / `AmountSold()` return exact `Int64`
- `.../xdr/xdr_generated.go:36327-36333` — order-book claim atoms store `AmountSold` and `AmountBought` as `Int64`

## Evidence

The trade transformer keeps the exact XDR amounts intact until the final row-construction step, so the precision loss is introduced locally rather than inherited from upstream parsing. Unlike derived analytics fields, `selling_amount` and `buying_amount` are the primary monetary outputs of the trade table, so rounding them changes the exported facts rather than merely presentation.

## Anti-Evidence

This only manifests for very large trades; smaller amounts still survive the float conversion unchanged. There is no exact raw-stroop or decimal-string companion field in `TradeOutput`, so downstream systems cannot reconstruct the lost stroop once the row is written.
