# H001: Account export fabricates empty `inflation_destination` strings

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an `AccountEntry` has no inflation destination (`InflationDest == nil`), the exported account row should preserve that field as absent or null. Downstream consumers should be able to distinguish "this account never set an inflation destination" from "this row contains a concrete destination account."

## Mechanism

`TransformAccount()` reads the optional XDR pointer `AccountEntry.InflationDest`, but it stores the result in a plain Go `string` and leaves that string at its zero value when the pointer is nil. `AccountOutput` also models `inflation_destination` as a required string, so the JSON export emits `""` instead of preserving missingness, fabricating a value that never existed on-chain.

## Trigger

1. Export any account ledger entry whose XDR `InflationDest` pointer is unset.
2. Inspect the JSON account row.
3. Observe `inflation_destination: ""` even though the source ledger entry had no destination account.

## Target Code

- `internal/transform/account.go:59-66` — optional `InflationDest` is copied into a plain `string`.
- `internal/transform/account.go:86-110` — `AccountOutput` is populated with that zero-valued string.
- `internal/transform/schema.go:97-111` — `InflationDestination` is declared as non-nullable `string`.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:3835-3862` — `AccountEntry.InflationDest` is optional (`AccountID*` / `*AccountId`).

## Evidence

The XDR schema explicitly marks `inflationDest` as an optional pointer, so "missing" is a first-class source-state value. In the transform layer, the only guard is `if inflationDestAccountID != nil { ... }`; otherwise `outputInflationDest` stays `""`, and the final `AccountOutput` has no nullable wrapper or `omitempty` tag to preserve the absence.

## Anti-Evidence

The current schema may be intentionally using empty strings as a sentinel for some legacy consumers, because `inflation_destination` is not typed as `null.String` or `omitempty`. But the same package already preserves other optional values with nullable wrappers or omitted fields, and `""` is not a valid Stellar account ID, so this still looks like fabricated data rather than a meaningful encoded state.

---

## Review

**Verdict**: VIABLE
**Severity**: Informational
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from XDR `AccountEntry.InflationDest` (an optional `*AccountId` pointer) through `TransformAccount()` which reads it into a plain Go `string` variable (zero value `""`), to the `AccountOutput` struct where `InflationDestination` is declared as `string` without `null.String` or `omitempty`. Confirmed that JSON marshaling produces `"inflation_destination": ""` for accounts with no inflation destination. Verified that the analogous optional field `Sponsor` correctly uses `null.String` via `ledgerEntrySponsorToNullString()`, establishing an internal inconsistency.

### Code Paths Examined

- `internal/transform/account.go:59-66` — `inflationDestAccountID` is checked for nil; if nil, `outputInflationDest` stays at Go zero value `""`
- `internal/transform/account.go:86-110` — `AccountOutput` is populated with the zero-valued string at line 95
- `internal/transform/schema.go:106` — `InflationDestination string \`json:"inflation_destination"\`` — plain string, no nullable wrapper
- `internal/transform/schema.go:113` — `Sponsor null.String \`json:"sponsor"\`` — contrast: optional pointer field correctly uses nullable type
- `internal/transform/trustline.go:109-118` — `ledgerEntrySponsorToNullString()` shows the correct pattern for optional XDR pointer → nullable output

### Findings

The bug is real: the code fabricates `""` in JSON output for an XDR field whose on-chain state is "absent." This is inconsistent with how the same codebase handles the `Sponsor` field (also an optional XDR pointer), which correctly uses `null.String` to preserve nullability.

**Severity downgraded from High to Informational** because:
1. `""` is not a valid Stellar account address — no downstream consumer would mistake it for a real inflation destination
2. The empty string effectively serves as a sentinel for "no value," even if semantically imprecise
3. In BigQuery, queries like `WHERE inflation_destination IS NOT NULL` would incorrectly include these rows, but `WHERE inflation_destination != ''` works around it
4. The impact is a schema-level nullability inconsistency, not data corruption that would produce wrong analytical results

This is consistent with similar Parquet-level findings (success items 010, 012, 013) that documented nullable field collapsing, though those were about Parquet conversion losing existing JSON nullability, while this is about the JSON schema never having nullability in the first place.

### PoC Guidance

- **Test file**: `internal/transform/account_test.go`
- **Setup**: Create a mock `ingest.Change` with an `AccountEntry` whose `InflationDest` is nil
- **Steps**: Call `TransformAccount()` with the mock change and inspect the returned `AccountOutput`
- **Assertion**: Assert that `result.InflationDestination == ""` to demonstrate the fabrication. A fix would change the schema to `null.String` and assert `result.InflationDestination.Valid == false`

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestAccountInflationDestinationEmptyString"
**Test Language**: Go

### Demonstration

The test constructs an `AccountEntry` with `InflationDest = nil` (no inflation destination on-chain), passes it through `TransformAccount()`, and confirms the output fabricates `InflationDestination: ""`. It further demonstrates the inconsistency by JSON-marshaling the output, showing `"inflation_destination":""` (fabricated empty string) alongside `"sponsor":null` (correctly nullable), proving that the same codebase handles analogous optional XDR pointers differently.

### Test Body

```go
package transform

import (
	"encoding/json"
	"strings"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestAccountInflationDestinationEmptyString demonstrates that TransformAccount
// fabricates an empty string "" for InflationDestination when the on-chain
// AccountEntry has no inflation destination (InflationDest == nil). The field
// should be null/absent, but because the schema uses a plain string type
// instead of null.String, the JSON output contains "inflation_destination": ""
// — a value that never existed on-chain.
func TestAccountInflationDestinationEmptyString(t *testing.T) {
	// 1. Construct an AccountEntry with InflationDest = nil (no inflation destination set)
	accountEntry := xdr.AccountEntry{
		AccountId:     genericAccountID,
		Balance:       100000000, // 10 XLM
		SeqNum:        1,
		NumSubEntries: 0,
		InflationDest: nil, // explicitly no inflation destination
		Flags:         0,
		HomeDomain:    "",
		Thresholds:    xdr.Thresholds([4]byte{1, 0, 0, 0}),
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeAccount,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
			LastModifiedLedgerSeq: 100,
			Data: xdr.LedgerEntryData{
				Type:    xdr.LedgerEntryTypeAccount,
				Account: &accountEntry,
			},
		},
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	// 2. Run through the production code path
	output, err := TransformAccount(change, header)
	if err != nil {
		t.Fatalf("TransformAccount returned unexpected error: %v", err)
	}

	// 3. Assert the output fabricates an empty string for a nil inflation destination
	if output.InflationDestination != "" {
		t.Fatalf("expected InflationDestination to be empty string (the bug), got %q", output.InflationDestination)
	}

	// 4. Demonstrate the downstream impact: JSON serialization emits a concrete
	//    empty string instead of null, which is inconsistent with how the Sponsor
	//    field (also an optional XDR pointer) is handled.
	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("json.Marshal failed: %v", err)
	}
	jsonStr := string(jsonBytes)

	// The inflation_destination field is present as an empty string in JSON
	if !strings.Contains(jsonStr, `"inflation_destination":""`) {
		t.Errorf("expected JSON to contain \"inflation_destination\":\"\" but got: %s", jsonStr)
	}

	// In contrast, the Sponsor field (also nil/unset) correctly serializes as null
	if !strings.Contains(jsonStr, `"sponsor":null`) {
		t.Errorf("expected JSON to contain \"sponsor\":null but got: %s", jsonStr)
	}

	t.Logf("BUG CONFIRMED: InflationDestination=%q (empty string fabricated for nil XDR pointer)", output.InflationDestination)
	t.Logf("CONTRAST: Sponsor field correctly serializes as null for unset optional pointer")
	t.Logf("JSON output (truncated): ...inflation_destination\":\"\",... vs ...\"sponsor\":null,...")
}
```

### Test Output

```
=== RUN   TestAccountInflationDestinationEmptyString
    data_integrity_poc_test.go:83: BUG CONFIRMED: InflationDestination="" (empty string fabricated for nil XDR pointer)
    data_integrity_poc_test.go:84: CONTRAST: Sponsor field correctly serializes as null for unset optional pointer
    data_integrity_poc_test.go:85: JSON output (truncated): ...inflation_destination":"",... vs ..."sponsor":null,...
--- PASS: TestAccountInflationDestinationEmptyString (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.926s
```
