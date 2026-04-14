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

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Success entries 002 (token-transfer) and 024 (change-trust-limit) confirm the same class of bug in sibling entities but not in trade amounts.

### Trace Summary

`TransformTrade()` in `trade.go` reads exact `xdr.Int64` amounts from `ClaimAtom.AmountSold()` (line 52) and `ClaimAtom.AmountBought()` (line 64), then passes both through `utils.ConvertStroopValueToReal()` at lines 139 and 145 when constructing `TradeOutput`. `ConvertStroopValueToReal()` in `utils/main.go:85-87` uses `big.NewRat(int64(input), 10000000).Float64()`, which rounds to the nearest IEEE-754 float64. The resulting `float64` is stored in `TradeOutput.SellingAmount` and `TradeOutput.BuyingAmount` (both `float64` in `schema.go:294,300`), and also in `TradeOutputParquet.SellingAmount` and `TradeOutputParquet.BuyingAmount` (both `float64` in `schema_parquet.go:227,233`). No guard, alternate representation, or exact companion field exists.

### Code Paths Examined

- `internal/transform/trade.go:52` — `outputSellingAmount := claimOffer.AmountSold()` returns exact `xdr.Int64`
- `internal/transform/trade.go:64` — `outputBuyingAmount := int64(claimOffer.AmountBought())` casts exact `xdr.Int64` to `int64`
- `internal/transform/trade.go:139` — `SellingAmount: utils.ConvertStroopValueToReal(outputSellingAmount)` — lossy conversion to `float64`
- `internal/transform/trade.go:145` — `BuyingAmount: utils.ConvertStroopValueToReal(xdr.Int64(outputBuyingAmount))` — same lossy conversion
- `internal/utils/main.go:85-87` — `ConvertStroopValueToReal` does `big.NewRat(int64(input), 10000000).Float64()` — rounding occurs here
- `internal/transform/schema.go:294,300` — `SellingAmount float64` and `BuyingAmount float64` in `TradeOutput`
- `internal/transform/schema_parquet.go:227,233` — `SellingAmount float64` and `BuyingAmount float64` in `TradeOutputParquet`

### Findings

Empirically confirmed: `ConvertStroopValueToReal(90071992547409930)` and `ConvertStroopValueToReal(90071992547409931)` both produce `9007199254.740993499755859375` — identical float64 values for distinct stroop inputs. The ULP at ~9e9 is approximately 1.9e-6, which is larger than the 1e-7 difference between adjacent stroops, so the rounding is mathematically inevitable once amounts reach this range.

Key aggravating factors compared to sibling findings (success/002, success/024):
1. **No exact companion field**: `TokenTransferOutput` has `AmountRaw` string; `TradeOutput` has no raw/exact alternative. Precision is permanently lost.
2. **Both monetary columns affected**: Both `SellingAmount` and `BuyingAmount` go through the same lossy path.
3. **Parquet equally affected**: `TradeOutputParquet` stores the same rounded `float64` values (`type=DOUBLE`).

The code path is live and exercised by `export_trades` via `TransformTrade()` for every successful trade-producing operation (ManageBuyOffer, ManageSellOffer, CreatePassiveSellOffer, PathPaymentStrictSend, PathPaymentStrictReceive).

### PoC Guidance

- **Test file**: `internal/transform/trade_test.go`
- **Setup**: Construct two `ingest.LedgerTransaction` objects with successful ManageSellOffer results. Each should contain a single `ClaimAtom` with `AmountSold = 90071992547409930` for the first and `AmountSold = 90071992547409931` for the second. Use existing test helper patterns (`utils.CreateSampleTx`, or construct XDR directly as done in existing trade tests).
- **Steps**: Call `TransformTrade()` for each transaction. Extract `SellingAmount` from the first element of each returned `[]TradeOutput`.
- **Assertion**: Assert that the two `SellingAmount` values are equal (`==`), demonstrating that distinct on-chain amounts collapse to the same exported value. Optionally, marshal both `TradeOutput` structs to JSON and assert byte-equality of the `selling_amount` field.
