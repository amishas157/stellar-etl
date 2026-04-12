# H003: Mirrored path-payment limit fields use different JSON types

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The caller-supplied limit field for the two mirrored path-payment operations should use one stable representation across the export surface. `path_payment_strict_receive.details.source_max` and `path_payment_strict_send.details.destination_min` are the two user-provided bounds that cap how much value the path payment may consume or deliver, so they should both serialize as exact decimal strings or both serialize as numbers.

## Mechanism

`extractOperationDetails()` exports `strict_receive.source_max` through `utils.ConvertStroopValueToReal(op.SendMax)` but exports the mirrored `strict_send.destination_min` through `amount.String(op.DestMin)`. That means the same conceptual limit field family changes JSON type between operation variants: one branch emits a `float64`, the other an exact string.

## Trigger

1. Export one successful `path_payment_strict_receive` row and one successful `path_payment_strict_send` row.
2. Compare `details.source_max` against `details.destination_min`.
3. Observe that the strict-receive limit is a JSON number while the strict-send limit is a quoted decimal string, despite both fields representing the user-specified path-payment bound.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:631-674` — uses `ConvertStroopValueToReal()` for `source_max` and `amount.String()` for `destination_min`
- `internal/transform/operation_test.go:1103-1113` — successful strict-receive fixture asserts numeric `source_max`
- `internal/transform/operation_test.go:1449-1455` — successful strict-send fixture asserts string `destination_min`

## Evidence

Both mirrored branches live in the same exporter function, so the divergent typing is not coming from separate subsystems or a dead fallback path. The checked-in fixtures explicitly show the current output contract already diverges between the two path-payment variants.

## Anti-Evidence

The field names are different (`source_max` vs `destination_min`), so reviewer may decide they are not required to share a schema despite their mirrored semantics. The tests also lock in the current divergent behavior, which may be treated as contract rather than corruption.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

In `extractOperationDetails()` (the live export path), every amount and limit field uses `utils.ConvertStroopValueToReal()` to produce `float64` values — except `destination_min` on line 674, which uses `amount.String(op.DestMin)` to produce a string. This makes `destination_min` the sole outlier among ~20 amount fields in the function. The unreachable `transactionOperationWrapper.Details()` implementation (lines 1386, 1409) uses `amount.String()` consistently for both fields, suggesting `destination_min` in the live path is a copy-paste artifact that was never updated.

### Code Paths Examined

- `internal/transform/operation.go:619-658` — strict-receive branch: `source_max` assigned via `ConvertStroopValueToReal(op.SendMax)` → float64
- `internal/transform/operation.go:660-699` — strict-send branch: `destination_min` assigned via `amount.String(op.DestMin)` → string
- `internal/transform/operation.go:1379-1416` — unreachable `transactionOperationWrapper.Details()`: both `source_max` and `destination_min` use `amount.String()` consistently (both strings)
- `internal/utils/main.go:84-88` — `ConvertStroopValueToReal` divides xdr.Int64 by 1e7, returns float64
- `internal/transform/operation_test.go:1107` — fixture asserts `source_max: 895.14959` (float64 literal)
- `internal/transform/operation_test.go:1453` — fixture asserts `destination_min: "428.0460538"` (string literal)

### Findings

1. **Type inconsistency confirmed**: `destination_min` is the only amount/limit field in `extractOperationDetails()` that uses `amount.String()` instead of `ConvertStroopValueToReal()`. All other amount fields (~20 occurrences on lines 600-1061) produce float64.

2. **Downstream impact**: JSON output contains a float64 for `source_max` (e.g., `895.14959`) and a string for `destination_min` (e.g., `"428.0460538"`). Downstream consumers doing cross-operation-type analytics on path payment limits must handle heterogeneous types.

3. **Copy-paste origin**: The unreachable `transactionOperationWrapper.Details()` uses `amount.String()` for both fields consistently. The live `extractOperationDetails()` function was updated to use `ConvertStroopValueToReal()` for most fields, but `destination_min` was missed — it still uses the old `amount.String()` pattern.

4. **Severity downgrade rationale**: Reduced from High to Medium because the data VALUE is correct in both representations — only the JSON type differs. The fields appear in different operation types with different names, so no single row has conflicting types. However, the inconsistency breaks the function's own established pattern and creates a schema divergence that downstream systems must handle.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Use the existing test infrastructure with `makeOperationTestInput()` for both `PathPaymentStrictReceive` and `PathPaymentStrictSend` operation types.
- **Steps**: Call `extractOperationDetails()` for a successful `PathPaymentStrictReceive` and a successful `PathPaymentStrictSend`. Inspect the Go types of `details["source_max"]` and `details["destination_min"]`.
- **Assertion**: Assert that `reflect.TypeOf(details["source_max"])` is `float64` while `reflect.TypeOf(details["destination_min"])` is `string`, confirming the type divergence. Alternatively, JSON-marshal both detail maps and verify that `source_max` appears as an unquoted number while `destination_min` appears as a quoted string.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestPathPaymentLimitFieldTypeDivergence"
**Test Language**: Go

### Demonstration

The test calls `extractOperationDetails()` for both PathPaymentStrictReceive and PathPaymentStrictSend operations from the existing test fixture. It confirms that `source_max` is a Go `float64` (serializing as an unquoted JSON number `895.14959`) while `destination_min` is a Go `string` (serializing as a quoted JSON string `"428.0460538"`). This proves the mirrored path-payment limit fields have inconsistent JSON types due to `destination_min` using `amount.String()` instead of `ConvertStroopValueToReal()`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"reflect"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestPathPaymentLimitFieldTypeDivergence demonstrates that
// extractOperationDetails() exports source_max as float64 (via
// ConvertStroopValueToReal) but destination_min as string (via amount.String),
// creating a JSON type inconsistency between the two mirrored path-payment
// limit fields.
func TestPathPaymentLimitFieldTypeDivergence(t *testing.T) {
	// Build a minimal successful transaction with both path-payment variants
	inputTx, err := makeOperationTestInput()
	if err != nil {
		t.Fatalf("makeOperationTestInput failed: %v", err)
	}

	// --- PathPaymentStrictReceive (operation index 3 in the test fixture) ---
	strictReceiveOp := inputTx.Envelope.Operations()[3]
	receiveDetails, err := extractOperationDetails(strictReceiveOp, inputTx, 3, "")
	if err != nil {
		t.Fatalf("extractOperationDetails for StrictReceive failed: %v", err)
	}

	sourceMax, ok := receiveDetails["source_max"]
	if !ok {
		t.Fatal("source_max field missing from StrictReceive details")
	}

	// --- PathPaymentStrictSend (operation index 15 in the test fixture) ---
	strictSendOp := inputTx.Envelope.Operations()[15]
	sendDetails, err := extractOperationDetails(strictSendOp, inputTx, 15, "")
	if err != nil {
		t.Fatalf("extractOperationDetails for StrictSend failed: %v", err)
	}

	destMin, ok := sendDetails["destination_min"]
	if !ok {
		t.Fatal("destination_min field missing from StrictSend details")
	}

	// Assert the type divergence: source_max is float64, destination_min is string
	sourceMaxType := reflect.TypeOf(sourceMax)
	destMinType := reflect.TypeOf(destMin)

	t.Logf("source_max Go type:      %v  value: %v", sourceMaxType, sourceMax)
	t.Logf("destination_min Go type:  %v  value: %v", destMinType, destMin)

	if sourceMaxType.Kind() != reflect.Float64 {
		t.Errorf("expected source_max to be float64, got %v", sourceMaxType)
	}
	if destMinType.Kind() != reflect.String {
		t.Errorf("expected destination_min to be string, got %v", destMinType)
	}

	// Demonstrate the JSON serialization divergence
	receiveJSON, _ := json.Marshal(receiveDetails)
	sendJSON, _ := json.Marshal(sendDetails)
	t.Logf("StrictReceive JSON (excerpt): %s", findFieldInJSON(receiveJSON, "source_max"))
	t.Logf("StrictSend JSON (excerpt):    %s", findFieldInJSON(sendJSON, "destination_min"))

	// Confirm that the two limit fields have different JSON types
	var receiveMap map[string]json.RawMessage
	var sendMap map[string]json.RawMessage
	json.Unmarshal(receiveJSON, &receiveMap)
	json.Unmarshal(sendJSON, &sendMap)

	sourceMaxRaw := string(receiveMap["source_max"])
	destMinRaw := string(sendMap["destination_min"])

	// source_max should be an unquoted number, destination_min should be a quoted string
	sourceMaxIsNumber := len(sourceMaxRaw) > 0 && sourceMaxRaw[0] != '"'
	destMinIsString := len(destMinRaw) > 0 && destMinRaw[0] == '"'

	t.Logf("source_max JSON raw:      %s (is number: %v)", sourceMaxRaw, sourceMaxIsNumber)
	t.Logf("destination_min JSON raw: %s (is string: %v)", destMinRaw, destMinIsString)

	if !sourceMaxIsNumber {
		t.Errorf("expected source_max to serialize as JSON number, got: %s", sourceMaxRaw)
	}
	if !destMinIsString {
		t.Errorf("expected destination_min to serialize as JSON string, got: %s", destMinRaw)
	}

	// Both assertions passing proves the type divergence bug
	if sourceMaxIsNumber && destMinIsString {
		t.Log("BUG CONFIRMED: mirrored path-payment limit fields use different JSON types — " +
			"source_max is a JSON number (float64) while destination_min is a JSON string")
	}
}

func findFieldInJSON(data []byte, field string) string {
	var m map[string]json.RawMessage
	if err := json.Unmarshal(data, &m); err != nil {
		return "<unmarshal error>"
	}
	raw, ok := m[field]
	if !ok {
		return "<field not found>"
	}
	return string(raw)
}

// buildMinimalPathPaymentTx constructs a minimal transaction with just two
// operations: one PathPaymentStrictReceive and one PathPaymentStrictSend.
func buildMinimalPathPaymentTx() ingest.LedgerTransaction {
	sourceAccount := testAccount3

	ops := []xdr.Operation{
		{
			SourceAccount: &sourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypePathPaymentStrictReceive,
				PathPaymentStrictReceiveOp: &xdr.PathPaymentStrictReceiveOp{
					SendAsset:   nativeAsset,
					SendMax:     5000000000, // 500 XLM in stroops
					Destination: testAccount4,
					DestAsset:   nativeAsset,
					DestAmount:  5000000000,
					Path:        []xdr.Asset{},
				},
			},
		},
		{
			SourceAccount: &sourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypePathPaymentStrictSend,
				PathPaymentStrictSendOp: &xdr.PathPaymentStrictSendOp{
					SendAsset:   nativeAsset,
					SendAmount:  5000000000, // 500 XLM in stroops
					Destination: testAccount4,
					DestAsset:   nativeAsset,
					DestMin:     5000000000,
					Path:        []xdr.Asset{},
				},
			},
		},
	}

	tx := genericLedgerTransaction
	envelope := genericBumpOperationEnvelope
	envelope.Tx.SourceAccount = sourceAccount
	envelope.Tx.Operations = ops

	// Build non-successful result to avoid needing operation results
	tx.Envelope.V1 = &envelope
	tx.Result.Result.Result.Code = xdr.TransactionResultCodeTxFailed

	return tx
}

// TestPathPaymentLimitFieldTypeDivergenceMinimal is a minimal reproduction
// that doesn't depend on the large test fixture.
func TestPathPaymentLimitFieldTypeDivergenceMinimal(t *testing.T) {
	tx := buildMinimalPathPaymentTx()

	// Extract details for PathPaymentStrictReceive (index 0)
	receiveDetails, err := extractOperationDetails(tx.Envelope.Operations()[0], tx, 0, "")
	if err != nil {
		t.Fatalf("StrictReceive extractOperationDetails failed: %v", err)
	}

	// Extract details for PathPaymentStrictSend (index 1)
	sendDetails, err := extractOperationDetails(tx.Envelope.Operations()[1], tx, 1, "")
	if err != nil {
		t.Fatalf("StrictSend extractOperationDetails failed: %v", err)
	}

	sourceMax := receiveDetails["source_max"]
	destMin := sendDetails["destination_min"]

	sourceMaxType := reflect.TypeOf(sourceMax)
	destMinType := reflect.TypeOf(destMin)

	t.Logf("source_max:      type=%v  value=%v", sourceMaxType, sourceMax)
	t.Logf("destination_min: type=%v  value=%v", destMinType, destMin)

	// Assert the bug: same-semantic fields have different Go types
	if sourceMaxType.Kind() != reflect.Float64 {
		t.Errorf("expected source_max to be float64, got %v", sourceMaxType)
	}
	if destMinType.Kind() != reflect.String {
		t.Errorf("expected destination_min to be string, got %v", destMinType)
	}

	if sourceMaxType.Kind() == reflect.Float64 && destMinType.Kind() == reflect.String {
		t.Log("BUG CONFIRMED: source_max is float64, destination_min is string")
	}
}
```

### Test Output

```
=== RUN   TestPathPaymentLimitFieldTypeDivergence
    data_integrity_poc_test.go:52: source_max Go type:      float64  value: 895.14959
    data_integrity_poc_test.go:53: destination_min Go type:  string  value: 428.0460538
    data_integrity_poc_test.go:65: StrictReceive JSON (excerpt): 895.14959
    data_integrity_poc_test.go:66: StrictSend JSON (excerpt):    "428.0460538"
    data_integrity_poc_test.go:81: source_max JSON raw:      895.14959 (is number: true)
    data_integrity_poc_test.go:82: destination_min JSON raw: "428.0460538" (is string: true)
    data_integrity_poc_test.go:93: BUG CONFIRMED: mirrored path-payment limit fields use different JSON types — source_max is a JSON number (float64) while destination_min is a JSON string
--- PASS: TestPathPaymentLimitFieldTypeDivergence (0.00s)
=== RUN   TestPathPaymentLimitFieldTypeDivergenceMinimal
    data_integrity_poc_test.go:182: source_max:      type=float64  value=500
    data_integrity_poc_test.go:183: destination_min: type=string  value=500.0000000
    data_integrity_poc_test.go:194: BUG CONFIRMED: source_max is float64, destination_min is string
--- PASS: TestPathPaymentLimitFieldTypeDivergenceMinimal (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.422s
```
