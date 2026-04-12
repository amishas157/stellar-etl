# H001: Trustline rows export `trust_line_limit` in raw stroops

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`trust_lines.trust_line_limit` should use the same asset-unit scale as the other trustline amount columns on the same row. For a trustline whose XDR `Limit` is `50000000` stroops, the exported row should carry `trust_line_limit = 5`, just as `balance = 10000123` stroops becomes `balance = 1.0000123`.

## Mechanism

`TransformTrustline()` converts `Balance`, `BuyingLiabilities`, and `SellingLiabilities` with `utils.ConvertStroopValueToReal(...)`, but it writes `TrustlineLimit` as the raw `int64(trustEntry.Limit)`. That makes the same trustline row mix scaled asset amounts with an unscaled limit, so downstream consumers comparing `balance` to `trust_line_limit` see values that differ by `1e7`.

## Trigger

Export any non-native trustline row whose limit is not zero, for example a trustline with `Balance = 10000123` stroops and `Limit = 50000000` stroops. The current JSON/Parquet row will emit `balance = 1.0000123` and `trust_line_limit = 50000000` instead of `5`.

## Target Code

- `internal/transform/trustline.go:68-88` — converts trustline balances/liabilities but copies `Limit` raw
- `internal/transform/schema.go:243-259` — exposes `trust_line_limit` as part of the exported trustline row
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:5103-5108` — `TrustLineEntry.Balance` and `TrustLineEntry.Limit` are both `Int64`

## Evidence

`TransformTrustline()` uses the shared stroop helper for every other trustline amount field but leaves `TrustlineLimit` unconverted. The checked-in trustline tests and goldens already show the mismatch propagating into output: `balance` is rendered as `0.6203` while `trust_line_limit` remains `9000000000000000000`, and another fixture exports `balance = 10000123` beside `trust_line_limit = 50000000`.

## Anti-Evidence

Current tests and goldens already anchor the raw `trust_line_limit` representation, so this may reflect a legacy BigQuery contract rather than an accidental regression. But there is no parallel exact/raw limit field or comment explaining why only this one monetary trustline field should stay in stroops while its sibling columns are scaled.
