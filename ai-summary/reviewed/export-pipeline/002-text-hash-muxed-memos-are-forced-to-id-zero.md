# H002: Text and hash `to_muxed_id` memos are exported as fake muxed-account ID 0

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a V4 SEP-41 token event carries a `to_muxed_id` memo in text or hash form, `export_token_transfers` should either preserve that memo in a type-appropriate field or leave `to_muxed` / `to_muxed_id` unset. It should not synthesize a numeric muxed account unless the upstream event metadata actually contains the numeric-ID arm of the muxed oneof.

## Mechanism

`transformEvents()` only checks whether `event.Meta.ToMuxedInfo` is non-nil, then always calls `GetId()` and builds an `M...` address from that numeric result. In the upstream protobuf, `MuxedInfo` is a oneof with `text`, `id`, and `hash` arms, and `GetId()` returns `0` whenever the active arm is text or hash. That means every text/hash memo is silently rewritten into `to_muxed_id="0"` plus a fabricated muxed address for account ID 0, destroying the real memo type and replacing it with believable but wrong output.

## Trigger

Run `export_token_transfers` on a ledger containing a V4 SEP-41 transfer whose event data map includes `to_muxed_id: "hello world"` or `to_muxed_id: <32-byte hash>`. The exported row will contain `to_muxed_id="0"` and a derived `to_muxed` address even though the source event had no numeric muxed ID.

## Target Code

- `internal/transform/token_transfer.go:95-105` — non-nil `ToMuxedInfo` is always treated as the numeric-ID arm
- `internal/transform/token_transfer.go:108-126` — fabricated `to_muxed` / `to_muxed_id` values are emitted
- `internal/transform/schema.go:675-676` — exported fields imply a numeric muxed-account interpretation

## Evidence

The upstream V4 parser explicitly accepts three `to_muxed_id` shapes in `parseV4MapDataForTokenEvents()`: `u64`, `bytes`, and `string`. Its generated protobuf exposes those as `MuxedInfo_Id`, `MuxedInfo_Hash`, and `MuxedInfo_Text`, and `GetId()` returns `0` unless the `id` arm is active. The upstream tests cover all three cases and assert that `"hello world"` populates `GetText()` while a 32-byte value populates `GetHash()`, so this repository is definitely receiving non-ID memos on a live code path.

## Anti-Evidence

Numeric `to_muxed_id` memos still work: the `id` arm survives through `GetId()` and produces a legitimate muxed address. The corruption is limited to text/hash memos introduced by the V4 SEP-41 event format.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The upstream SDK's `setDestinationMuxedInfo()` populates `EventMeta.ToMuxedInfo` from two sources: (1) muxed account addresses → `MuxedInfo_Id`, or (2) transaction memos via `NewMuxedInfoFromMemo()` → `MuxedInfo_Text` for text memos, `MuxedInfo_Hash` for hash/return-hash memos, and `MuxedInfo_Id` for ID memos. In stellar-etl's `transformEvents()` (token_transfer.go:98-106), the code checks only `ToMuxedInfo != nil` and then unconditionally calls `GetId()`, which is a protobuf accessor that returns `0` for non-ID oneof arms. This fabricates a muxed address with ID 0 and sets `to_muxed_id="0"` for every text or hash memo, producing plausible but incorrect output.

### Code Paths Examined

- `go-stellar-sdk/processors/token_transfer/token_transfer_event.go:181-202` — `setDestinationMuxedInfo()` routes to `NewMuxedInfoFromMemo(txMemo)` for non-muxed destination accounts, which creates `MuxedInfo_Text` or `MuxedInfo_Hash` arms for text/hash memos
- `go-stellar-sdk/processors/token_transfer/muxed_info.go:9-42` — `NewMuxedInfoFromMemo()` constructs `MuxedInfo` with `Content` set to `MuxedInfo_Text`, `MuxedInfo_Hash`, or `MuxedInfo_Id` depending on memo type
- `go-stellar-sdk/processors/token_transfer/token_transfer_event.pb.go:628-636` — `GetId()` returns `0` when `Content` is not `*MuxedInfo_Id` (standard protobuf zero-value return)
- `internal/transform/token_transfer.go:98-106` — unconditionally calls `GetId()` on non-nil `ToMuxedInfo`, producing `muxedID=0` for text/hash arms, then builds a fabricated `M...` address and sets `toMuxedID="0"`
- `internal/transform/token_transfer.go:108-126` — the fabricated `toMuxed` and `toMuxedID` values are written into the output struct
- `internal/transform/schema.go:675-676` — `ToMuxed` and `ToMuxedID` fields in `TokenTransferOutput` are typed as `null.String`, implying a numeric muxed-account interpretation with no provision for text/hash memo types

### Findings

The bug is confirmed. The complete chain is:

1. A Stellar transaction with a text or hash memo triggers a token transfer event
2. `setDestinationMuxedInfo()` calls `NewMuxedInfoFromMemo(txMemo)`, which creates a `MuxedInfo` with `Content` = `MuxedInfo_Text{Text: "..."}` or `MuxedInfo_Hash{Hash: [...]}`
3. `ToMuxedInfo` on the `EventMeta` is non-nil
4. `transformEvents()` enters the `if event.Meta.ToMuxedInfo != nil` branch (line 98)
5. `event.Meta.ToMuxedInfo.GetId()` returns `0` because the active arm is Text or Hash, not Id (line 99)
6. A `strkey.MuxedAccount` is constructed with `SetID(0)` and the destination account (lines 100-103)
7. `toMuxed` is set to this fabricated `M...` address (line 104)
8. `toMuxedID` is set to `"0"` (line 105)

The fabricated data is indistinguishable from a legitimate muxed account with ID 0, making it dangerous for downstream consumers. Any compliance or analytics system filtering for muxed account activity will get false positives, and the actual memo content (text string or hash bytes) is silently discarded.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go` (create if needed, or append to existing test file)
- **Setup**: Construct a `TokenTransferEvent` with `Meta.ToMuxedInfo` set to `NewMuxedInfoFromText("hello world")` (text arm) and another with `NewMuxedInfoFromMemo()` using a hash memo. Also construct one with `NewMuxedInfoFromId(42)` as a control case.
- **Steps**: Call `transformEvents()` with each event. Extract the `ToMuxed` and `ToMuxedID` fields from the output.
- **Assertion**: For the text and hash cases, assert that `ToMuxed` and `ToMuxedID` are NOT populated (they should be `null.String{}` since there's no numeric muxed ID). For the ID case, assert `ToMuxedID` equals `"42"` and `ToMuxed` is a valid `M...` address. Currently the text/hash cases incorrectly produce `ToMuxedID="0"` and a fabricated `M...` address.
