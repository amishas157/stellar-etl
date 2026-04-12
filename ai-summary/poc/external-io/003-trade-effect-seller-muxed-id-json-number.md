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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestSellerMuxedIdJsonNumber"
**Test Language**: Go

### Demonstration

The test constructs a muxed account with ID 9007199254740993 (2^53 + 1), calls `addAccountAndMuxedAccountDetails` — the same production code path used by `tradeDetails()` for seller-side trade effects — and confirms three things: (1) the muxed ID is stored as raw `uint64` in the details map, (2) `json.Marshal` emits it as a bare number `9007199254740993` without quotes, and (3) a standard `json.Unmarshal` (float64-based) round-trip silently truncates the value to `9007199254740992`, proving 1-unit precision loss.

### Test Body

```go
package transform

import (
	"encoding/json"
	"fmt"
	"math"
	"strings"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestSellerMuxedIdJsonNumber demonstrates that addAccountAndMuxedAccountDetails
// stores muxed IDs as raw uint64, causing bare JSON numbers that lose precision
// when parsed by standard float64-based JSON decoders.
func TestSellerMuxedIdJsonNumber(t *testing.T) {
	// 1. Construct a muxed account with ID > 2^53 (the float64 safe integer limit)
	const muxedId uint64 = 9007199254740993 // 2^53 + 1

	var ed25519Key xdr.Uint256
	// Use a deterministic key (all zeros is fine for this test)
	muxedAccount := xdr.MuxedAccount{
		Type: xdr.CryptoKeyTypeKeyTypeMuxedEd25519,
		Med25519: &xdr.MuxedAccountMed25519{
			Id:      xdr.Uint64(muxedId),
			Ed25519: ed25519Key,
		},
	}

	// 2. Call the production code path
	result := map[string]interface{}{}
	err := addAccountAndMuxedAccountDetails(result, muxedAccount, "seller")
	if err != nil {
		t.Fatalf("addAccountAndMuxedAccountDetails failed: %v", err)
	}

	// 3a. Assert the stored value is uint64 (not string)
	rawVal, ok := result["seller_muxed_id"]
	if !ok {
		t.Fatal("seller_muxed_id not found in result map")
	}
	storedUint, isUint64 := rawVal.(uint64)
	if !isUint64 {
		t.Fatalf("seller_muxed_id is %T, expected uint64 — the bug may have been fixed", rawVal)
	}
	if storedUint != muxedId {
		t.Fatalf("seller_muxed_id value wrong: got %d, want %d", storedUint, muxedId)
	}
	t.Logf("seller_muxed_id stored as uint64: %d (confirms bare number will be emitted)", storedUint)

	// 3b. JSON-marshal the map and verify the output is a bare number (no quotes)
	jsonBytes, err := json.Marshal(result)
	if err != nil {
		t.Fatalf("json.Marshal failed: %v", err)
	}
	jsonStr := string(jsonBytes)
	t.Logf("JSON output: %s", jsonStr)

	// The bare number pattern: "seller_muxed_id":9007199254740993 (no quotes around value)
	barePattern := fmt.Sprintf(`"seller_muxed_id":%d`, muxedId)
	quotedPattern := fmt.Sprintf(`"seller_muxed_id":"%d"`, muxedId)

	if strings.Contains(jsonStr, quotedPattern) {
		t.Fatal("seller_muxed_id is quoted as a string — the bug appears to be fixed")
	}
	if !strings.Contains(jsonStr, barePattern) {
		t.Fatalf("Expected bare number pattern %q in JSON, got: %s", barePattern, jsonStr)
	}
	t.Log("Confirmed: seller_muxed_id is a bare JSON number (not quoted)")

	// 3c. Demonstrate precision loss: unmarshal with standard float64 decoder
	var parsed map[string]interface{}
	err = json.Unmarshal(jsonBytes, &parsed)
	if err != nil {
		t.Fatalf("json.Unmarshal failed: %v", err)
	}

	// Standard json.Unmarshal decodes numbers as float64
	floatVal, ok := parsed["seller_muxed_id"].(float64)
	if !ok {
		t.Fatalf("After standard unmarshal, seller_muxed_id is %T, expected float64", parsed["seller_muxed_id"])
	}

	// Convert back to uint64 to check precision
	recoveredId := uint64(floatVal)
	precisionLost := recoveredId != muxedId

	t.Logf("Original muxed ID:  %d", muxedId)
	t.Logf("After float64 round-trip: %d", recoveredId)
	t.Logf("Precision lost: %v (diff = %d)", precisionLost, int64(muxedId)-int64(recoveredId))
	t.Logf("Max safe integer (2^53): %d", uint64(math.Pow(2, 53)))

	if !precisionLost {
		t.Fatal("Expected precision loss but float64 preserved the value — hypothesis not demonstrated")
	}

	t.Log("BUG CONFIRMED: seller_muxed_id exported as bare JSON number loses precision in standard JSON parsers")
}
```

### Test Output

```
=== RUN   TestSellerMuxedIdJsonNumber
    data_integrity_poc_test.go:49: seller_muxed_id stored as uint64: 9007199254740993 (confirms bare number will be emitted)
    data_integrity_poc_test.go:57: JSON output: {"seller":"GAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAWHF","seller_muxed":"MAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAGW72","seller_muxed_id":9007199254740993}
    data_integrity_poc_test.go:69: Confirmed: seller_muxed_id is a bare JSON number (not quoted)
    data_integrity_poc_test.go:88: Original muxed ID:  9007199254740993
    data_integrity_poc_test.go:89: After float64 round-trip: 9007199254740992
    data_integrity_poc_test.go:90: Precision lost: true (diff = 1)
    data_integrity_poc_test.go:91: Max safe integer (2^53): 9007199254740992
    data_integrity_poc_test.go:97: BUG CONFIRMED: seller_muxed_id exported as bare JSON number loses precision in standard JSON parsers
--- PASS: TestSellerMuxedIdJsonNumber (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.961s
```
