# H003: Seller-side trade effects export `seller_muxed_id` as a JSON number

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: identifier precision / JSON contract drift
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_effects` emits seller-side trade / offer effects for a muxed counterparty, the `details.seller_muxed_id` field should preserve the exact 64-bit muxed ID as a quoted decimal string, e.g. `"seller_muxed_id":"9007199254740993"`. That matches Horizon's trade-effect schema and keeps the dedicated ID field lossless without forcing downstream systems to decode the `seller_muxed` address.

## Mechanism

`tradeDetails()` builds the seller-side detail map and then calls `addAccountAndMuxedAccountDetails(sd, buyer, "seller")`. The helper writes the muxed ID as raw `uint64`, so JSON export emits a bare numeric literal even though Horizon's `Trade` effect resource defines `seller_muxed_id` with `json:",string"`. For any legitimate muxed buyer whose ID exceeds `2^53`, downstream JSON pipelines can silently round the identifier or infer the wrong type.

## Trigger

1. Create any trade-producing operation whose counterparty buyer is a muxed account with ID above `9007199254740992`.
2. Run `export_effects` across the ledger so the seller-side trade / offer effect rows are emitted.
3. Observe that `details.seller_muxed_id` is exported as a bare JSON number instead of a quoted decimal string.

## Target Code

- `internal/transform/effects.go:tradeDetails:1229-1245` — seller-side trade effect details call the muxed-account helper
- `internal/transform/operation.go:addAccountAndMuxedAccountDetails:423-438` — inserts the raw `uint64` into the generic map

## Evidence

The seller-side branch of `tradeDetails()` populates `sd` and then delegates `seller_*` fields to `addAccountAndMuxedAccountDetails()`, whose line 437 stores `muxedAccountId` directly. The upstream Horizon `Trade` effect schema declares `SellerMuxedID uint64 \`json:"seller_muxed_id,omitempty,string"\``, which is direct evidence that the canonical JSON representation is string-valued.

## Anti-Evidence

The companion `seller_muxed` address is present, so a consumer willing to decode muxed addresses can reconstruct the ID. Buyer-side trade effect details also omit `seller_muxed_id`, so only the seller-side branch is affected. But when the field is present, the exported JSON type is still inconsistent with Horizon and unsafe for generic number-based JSON processing.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `tradeDetails()` through `addAccountAndMuxedAccountDetails()` to `ExportEntry()`. The `xdr.MuxedAccount.GetId()` returns `uint64`. At `operation.go:437`, the raw `uint64` is stored into `map[string]interface{}` at key `seller_muxed_id`. This flows into `EffectOutput.Details` (type `map[string]interface{}`), is marshaled by `ExportEntry()` at `command_utils.go:57`, round-tripped through `UseNumber()` decode (line 63), and re-marshaled (line 73). The final JSON output contains a bare numeric literal. Empirically verified: `uint64(9007199254740993)` round-trips through `ExportEntry` as the bare JSON number `9007199254740993`, which a standard float64-based JSON parser decodes as `9007199254740992` — a silent 1-unit precision loss.

### Code Paths Examined

- `internal/transform/effects.go:tradeDetails:1229-1249` — creates seller-side details map `sd`, calls `addAccountAndMuxedAccountDetails(sd, buyer, "seller")` at line 1244
- `internal/transform/operation.go:addAccountAndMuxedAccountDetails:423-440` — calls `a.GetId()` (returns `uint64`) at line 433, stores raw `uint64` at line 437 via `result[prefix+"muxed_id"] = muxedAccountId`
- `internal/transform/effects.go:addClaimTradeEffects:985-1013` — calls `tradeDetails` at line 987, passes `sd` to `addUnmuxed` at line 1008, which flows into `EffectOutput.Details`
- `internal/transform/effects.go:add:176-184` — assigns the details map directly to `EffectOutput.Details`
- `cmd/command_utils.go:ExportEntry:55-80` — marshals struct to JSON, round-trips through `UseNumber()` decoder (preserves precision within Go), then re-marshals. Output is a bare JSON number.
- `stellar/go xdr/muxed_account.go:GetId:156` — confirmed return type is `(uint64, error)`

### Findings

The bug is confirmed and the mechanism is exactly as described. Key observations:

1. **Bare number in output**: `uint64` values in `map[string]interface{}` are marshaled as bare JSON numbers by Go's `encoding/json`. The `UseNumber()` call in `ExportEntry` only prevents intermediate float64 rounding within the Go process — the final output is still a bare number.

2. **Precision loss is demonstrable**: For `uint64(9007199254740993)`, the output JSON `9007199254740993` is silently rounded to `9007199254740992` by any IEEE 754 float64-based JSON parser (JavaScript, Python `json.loads`, many BigQuery JSON import paths).

3. **Muxed IDs can exceed 2^53**: Unlike sequentially-allocated offer IDs (H010, rejected), muxed account IDs are arbitrary user-chosen uint64 values. There is no protocol constraint preventing IDs above 2^53. Exchange deposit identifiers and other application-specific uses commonly use large values.

4. **Broader scope**: The `addAccountAndMuxedAccountDetails` function is called ~30 times across `operation.go` and `effects.go` for prefixes including `funder`, `from`, `to`, `seller`, `trustor`, `trustee`, `account`, `into`, `claimant`, `begin_sponsor`. ALL of these paths have the same bare-number encoding for `{prefix}_muxed_id`. The hypothesis correctly identifies the trade-effect path but the root cause in `addAccountAndMuxedAccountDetails:437` affects all muxed ID fields in operation and effect details.

5. **Inconsistent with Horizon**: Horizon uses `json:",string"` tags on muxed ID fields, producing quoted strings. This tool's output diverges from that convention.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go` (or a new `effects_muxed_test.go`)
- **Setup**: Construct an `xdr.MuxedAccount` of type `CryptoKeyTypeKeyTypeMuxedEd25519` with ID = `9007199254740993` (2^53 + 1). Build a mock trade claim with this muxed account as the buyer.
- **Steps**: Call `tradeDetails(muxedBuyer, seller, claim)` to get `bd, sd`. Then JSON-marshal `sd` (or pass through `ExportEntry` round-trip).
- **Assertion**: Assert that `sd["seller_muxed_id"]` is a `uint64` (not a string). Then assert that after `json.Marshal(sd)`, the output contains `"seller_muxed_id":9007199254740993` (bare number, no quotes). Finally, unmarshal that JSON with a standard decoder (no `UseNumber`) and assert that the float64 value does NOT equal the original `9007199254740993` — demonstrating precision loss. The fix would be to store `strconv.FormatUint(muxedAccountId, 10)` instead of the raw uint64 at `operation.go:437`.
