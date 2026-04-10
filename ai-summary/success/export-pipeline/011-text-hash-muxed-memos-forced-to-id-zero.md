# 011: Text/hash muxed memos are exported as fabricated ID 0

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_token_transfers` treats every non-nil `EventMeta.ToMuxedInfo` as though it contained the numeric-ID arm of the upstream oneof. When the upstream token-transfer parser supplies a text or hash memo instead, stellar-etl calls `GetId()`, receives protobuf's zero value, and exports a believable but fake muxed account plus `to_muxed_id="0"`.

This silently destroys the real memo shape for valid SEP-41 V4 events and replaces it with wrong exported data that downstream consumers cannot distinguish from a legitimate muxed account with ID 0.

## Root Cause

`transformEvents()` checks only `event.Meta.ToMuxedInfo != nil` and then unconditionally executes the numeric muxed-account path. Upstream `MuxedInfo` is a oneof with `text`, `id`, and `hash` arms; generated protobuf accessors return zero values for inactive arms, so `GetId()` returns `0` for text/hash memos and stellar-etl synthesizes an `M...` address from that fake ID.

## Reproduction

During normal token-transfer export, the upstream SDK sets `EventMeta.ToMuxedInfo` from either a real muxed destination account or from transaction / V4 memo data. For valid text or hash memos, the SDK populates the `text` or `hash` arm, but stellar-etl still converts that metadata into numeric `to_muxed` / `to_muxed_id` output.

## Affected Code

- `internal/transform/token_transfer.go:transformEvents:95-106` — any non-nil `ToMuxedInfo` is coerced through `GetId()` and encoded as a numeric muxed account
- `internal/transform/token_transfer.go:transformEvents:108-126` — fabricated `to_muxed` and `to_muxed_id` values are emitted into exported rows
- `internal/transform/schema.go:TokenTransferOutput:675-676` — the export schema only preserves the numeric interpretation, so non-ID arms are lost

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTextMuxedMemoForcedToIDZero`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create or replace `internal/transform/data_integrity_poc_test.go` with the test body below
  3. Run `go test ./internal/transform/... -run TestTextMuxedMemoForcedToIDZero -v`
  4. Observe that text/hash cases export `to_muxed_id="0"` plus a fabricated `to_muxed` address instead of leaving those fields unset

### Test Body

```go
package transform

import (
	"testing"
	"time"

	"github.com/guregu/null"
	"github.com/stellar/go-stellar-sdk/asset"
	"github.com/stellar/go-stellar-sdk/processors/token_transfer"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTextMuxedMemoForcedToIDZero demonstrates that when a token transfer event
// carries a text memo in ToMuxedInfo, transformEvents() unconditionally calls
// GetId() which returns 0, fabricating a muxed address with ID 0 and setting
// ToMuxedID="0" instead of leaving these fields unset.
func TestTextMuxedMemoForcedToIDZero(t *testing.T) {
	operationIndex := uint32(1)

	// Construct a transfer event with a TEXT memo in ToMuxedInfo
	textMemoEvent := &token_transfer.TokenTransferEvent{
		Meta: &token_transfer.EventMeta{
			LedgerSequence:   10,
			TxHash:           "txhash",
			TransactionIndex: 1,
			OperationIndex:   &operationIndex,
			ContractAddress:  "contractaddress",
			ToMuxedInfo:      token_transfer.NewMuxedInfoFromText("hello world"),
		},
		Event: &token_transfer.TokenTransferEvent_Transfer{
			Transfer: &token_transfer.Transfer{
				From: "from",
				To:   "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN7",
				Asset: &asset.Asset{
					AssetType: &asset.Asset_IssuedAsset{
						IssuedAsset: &asset.IssuedAsset{
							AssetCode: "abc",
							Issuer:    "def",
						},
					},
				},
				Amount: "100",
			},
		},
	}

	// Construct a control event with a NUMERIC ID memo (should produce valid muxed output)
	numericIDEvent := &token_transfer.TokenTransferEvent{
		Meta: &token_transfer.EventMeta{
			LedgerSequence:   10,
			TxHash:           "txhash",
			TransactionIndex: 1,
			OperationIndex:   &operationIndex,
			ContractAddress:  "contractaddress",
			ToMuxedInfo:      token_transfer.NewMuxedInfoFromId(42),
		},
		Event: &token_transfer.TokenTransferEvent_Transfer{
			Transfer: &token_transfer.Transfer{
				From: "from",
				To:   "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN7",
				Asset: &asset.Asset{
					AssetType: &asset.Asset_IssuedAsset{
						IssuedAsset: &asset.IssuedAsset{
							AssetCode: "abc",
							Issuer:    "def",
						},
					},
				},
				Amount: "100",
			},
		},
	}

	lcm := xdr.LedgerCloseMeta{
		V: 1,
		V1: &xdr.LedgerCloseMetaV1{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					ScpValue: xdr.StellarValue{
						CloseTime: 1000,
					},
					LedgerSeq: 10,
				},
			},
		},
	}

	// --- Test 1: Text memo should NOT produce a muxed address ---
	t.Run("text_memo_produces_fabricated_muxed_id", func(t *testing.T) {
		events := []*token_transfer.TokenTransferEvent{textMemoEvent}
		outputs, err := transformEvents(events, lcm)
		if err != nil {
			t.Fatalf("transformEvents failed: %v", err)
		}
		if len(outputs) != 1 {
			t.Fatalf("expected 1 output, got %d", len(outputs))
		}

		output := outputs[0]

		// BUG DEMONSTRATION: The text memo "hello world" causes ToMuxedID to be
		// set to "0" and ToMuxed to a fabricated M... address. A correct
		// implementation would leave both fields unset (null) since the memo is
		// text, not a numeric muxed account ID.

		// The bug: ToMuxedID is "0" instead of being unset
		if output.ToMuxedID.Valid && output.ToMuxedID.String == "0" {
			t.Errorf("BUG CONFIRMED: text memo 'hello world' was coerced to ToMuxedID=%q; "+
				"expected ToMuxedID to be unset (null) since the memo is text, not a numeric ID",
				output.ToMuxedID.String)
		}

		// The bug: ToMuxed is a fabricated M... address instead of being unset
		if output.ToMuxed.Valid && output.ToMuxed.String != "" {
			t.Errorf("BUG CONFIRMED: text memo 'hello world' produced fabricated ToMuxed=%q; "+
				"expected ToMuxed to be unset (null) since there is no numeric muxed account",
				output.ToMuxed.String)
		}
	})

	// --- Test 2: Hash memo should NOT produce a muxed address ---
	t.Run("hash_memo_produces_fabricated_muxed_id", func(t *testing.T) {
		hashBytes := xdr.Hash{0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
			0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f, 0x10,
			0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18,
			0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f, 0x20}
		hashMemo := xdr.Memo{Type: xdr.MemoTypeMemoHash, Hash: &hashBytes}
		hashMemoEvent := &token_transfer.TokenTransferEvent{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash",
				TransactionIndex: 1,
				OperationIndex:   &operationIndex,
				ContractAddress:  "contractaddress",
				ToMuxedInfo:      token_transfer.NewMuxedInfoFromMemo(hashMemo),
			},
			Event: &token_transfer.TokenTransferEvent_Transfer{
				Transfer: &token_transfer.Transfer{
					From: "from",
					To:   "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN7",
					Asset: &asset.Asset{
						AssetType: &asset.Asset_IssuedAsset{
							IssuedAsset: &asset.IssuedAsset{
								AssetCode: "abc",
								Issuer:    "def",
							},
						},
					},
					Amount: "100",
				},
			},
		}

		events := []*token_transfer.TokenTransferEvent{hashMemoEvent}
		outputs, err := transformEvents(events, lcm)
		if err != nil {
			t.Fatalf("transformEvents failed: %v", err)
		}
		if len(outputs) != 1 {
			t.Fatalf("expected 1 output, got %d", len(outputs))
		}

		output := outputs[0]

		// BUG DEMONSTRATION: Hash memo gets ToMuxedID="0" and a fabricated M... address
		if output.ToMuxedID.Valid && output.ToMuxedID.String == "0" {
			t.Errorf("BUG CONFIRMED: hash memo was coerced to ToMuxedID=%q; "+
				"expected ToMuxedID to be unset (null) since the memo is a hash, not a numeric ID",
				output.ToMuxedID.String)
		}

		if output.ToMuxed.Valid && output.ToMuxed.String != "" {
			t.Errorf("BUG CONFIRMED: hash memo produced fabricated ToMuxed=%q; "+
				"expected ToMuxed to be unset (null) since there is no numeric muxed account",
				output.ToMuxed.String)
		}
	})

	// --- Test 3: Control — numeric ID memo SHOULD produce a valid muxed address ---
	t.Run("numeric_id_memo_correctly_produces_muxed_address", func(t *testing.T) {
		events := []*token_transfer.TokenTransferEvent{numericIDEvent}
		outputs, err := transformEvents(events, lcm)
		if err != nil {
			t.Fatalf("transformEvents failed: %v", err)
		}
		if len(outputs) != 1 {
			t.Fatalf("expected 1 output, got %d", len(outputs))
		}

		output := outputs[0]

		// Numeric ID 42 should correctly produce ToMuxedID="42"
		if !output.ToMuxedID.Valid || output.ToMuxedID.String != "42" {
			t.Errorf("Control case failed: numeric ID memo should produce ToMuxedID=%q, got Valid=%v String=%q",
				"42", output.ToMuxedID.Valid, output.ToMuxedID.String)
		}

		// ToMuxed should be a valid M... address
		if !output.ToMuxed.Valid || output.ToMuxed.String == "" {
			t.Errorf("Control case failed: numeric ID memo should produce a valid ToMuxed M... address, got Valid=%v String=%q",
				output.ToMuxed.Valid, output.ToMuxed.String)
		}
	})

	// Suppress unused import warning
	_ = null.NewString("", false)
	_ = time.Now()
}
```

## Expected vs Actual Behavior

- **Expected**: Text and hash memo variants should not populate numeric `to_muxed` / `to_muxed_id` fields unless the upstream oneof actually contains `MuxedInfo_Id`
- **Actual**: Any non-nil `ToMuxedInfo` produces a numeric muxed address; text/hash memos are exported as `to_muxed_id="0"` and a fabricated `M...` address

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls `transformEvents()` directly with real `token_transfer.TokenTransferEvent` values using text, hash, and numeric-ID `ToMuxedInfo` variants.
2. Realistic preconditions: YES — upstream token-transfer code populates `ToMuxedInfo` from real muxed destinations or from transaction / V4 memo data, and upstream tests explicitly cover text and hash cases.
3. Bug vs by-design: BUG — the exported fields are numeric muxed-account fields, but the implementation fabricates numeric values for non-numeric oneof arms instead of preserving or omitting them.
4. Final severity: High — this is structural export corruption of non-financial fields; it silently rewrites valid metadata into plausible wrong output.
5. In scope: YES — the issue is a concrete code path in stellar-etl that emits wrong exported data under normal operation.
6. Test correctness: CORRECT — the control case proves numeric IDs still work, while text/hash cases fail only because `GetId()` on non-ID arms returns protobuf's zero value.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Only populate `ToMuxed` and `ToMuxedID` when `event.Meta.ToMuxedInfo` contains the `MuxedInfo_Id` arm. For `text` and `hash` arms, either add type-appropriate export fields or leave the numeric muxed-account fields unset.
