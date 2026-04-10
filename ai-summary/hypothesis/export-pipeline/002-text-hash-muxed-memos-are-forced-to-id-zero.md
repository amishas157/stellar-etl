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
