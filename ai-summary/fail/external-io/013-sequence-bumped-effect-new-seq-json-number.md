# H002: Sequence-bumped effect `details.new_seq` loses the canonical string encoding

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: identifier precision / JSON contract drift
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_effects` emits a `sequence_bumped` effect, the new sequence number should be exported losslessly in the canonical JSON shape, e.g. `"new_seq":"9007199254740993"`. That matches Horizon's effect schema and preserves exact sequence values for downstream reconciliation.

## Mechanism

The sequence-bumped effect builder creates `details := map[string]interface{}{"new_seq": afterAccount.SeqNum}` with a raw `int64`. `ExportEntry()` then marshals that generic map as a JSON number, while Horizon's `SequenceBumped` effect model declares `new_seq` with `json:",string"`. A legitimate bump-sequence operation can drive `SeqNum` above `2^53`, so the exported field becomes the wrong JSON type and is vulnerable to downstream rounding or schema mis-inference.

## Trigger

1. Construct a valid bump-sequence transaction that raises an account sequence number above `9007199254740992`, for example to `9007199254740993`.
2. Run `export_effects` across the resulting ledger.
3. Observe that the `sequence_bumped` effect row contains `"new_seq":9007199254740993` as a JSON number instead of the lossless string form.

## Target Code

- `internal/transform/effects.go:addBumpSequenceEffects:808-823` — emits `new_seq` into a generic map as raw `int64`

## Evidence

The live effect path stores `afterAccount.SeqNum` directly in the `details` map at line 820. The upstream Horizon effect schema defines `SequenceBumped.NewSeq int64 \`json:"new_seq,string"\`` in `protocols/horizon/effects/main.go`, and stellar-etl's sibling operation-details path already stringifies the same semantic value as `bump_to` via `fmt.Sprintf`, which shows the repo already treats large sequence numbers as string-worthy on adjacent surfaces.

## Anti-Evidence

Top-level effect columns outside `details` are unaffected, and consumers that explicitly coerce JSON numbers to 64-bit integers may still recover the exact value. But the exported JSON shape still diverges from the canonical Horizon contract on a field whose source value can be legitimately pushed past the safe IEEE-754 integer range.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially equivalent to 012-trade-effect-seller-muxed-id-json-number-is-schema-defined
**Failed At**: reviewer

### Trace Summary

Traced `addBumpSequenceEffects()` at effects.go:801-827. At line 820, `afterAccount.SeqNum` (type `xdr.SequenceNumber`, which is `xdr.Int64`/`int64`) is stored directly into `map[string]interface{}{"new_seq": afterAccount.SeqNum}`. This flows through `addMuxed()` → `add()` at line 176-184 into `EffectOutput.Details` (type `map[string]interface{}`), then through `ExportEntry()` at command_utils.go:55-80 which uses `UseNumber()` for its internal round-trip but ultimately emits a bare JSON number. The mechanism is exactly as described.

### Code Paths Examined

- `internal/transform/effects.go:addBumpSequenceEffects:801-827` — stores `afterAccount.SeqNum` (int64) as raw value in generic details map at line 820
- `internal/transform/effects.go:add:176-184` — assigns details map directly into `EffectOutput.Details`
- `internal/transform/schema.go:361-372` — `EffectOutput.Details` is `map[string]interface{}`, no typed schema for `new_seq`
- `cmd/command_utils.go:ExportEntry:55-80` — `UseNumber()` preserves precision within Go but final JSON output is still a bare number
- `internal/transform/effects_test.go:1706-1708` — test asserts `"new_seq": xdr.SequenceNumber(300000000000)`, confirming the repo expects numeric type

### Why It Failed

This is the same class of issue as H012 (seller_muxed_id bare JSON number), which was thoroughly investigated through the full pipeline and rejected at final review as **by design**. The specific reasons apply identically here:

1. **By design**: The repo's own test at effects_test.go:1707 explicitly asserts `"new_seq": xdr.SequenceNumber(300000000000)` — the expected type is the raw XDR numeric type, not a string. The `Details` field is a generic `map[string]interface{}` with no typed schema constraining `new_seq` to string encoding.

2. **No ETL-side corruption**: `ExportEntry()` uses `UseNumber()` internally, and the final JSON output contains the exact decimal digits of the int64 value. The emitted text `"new_seq":9007199254740993` is numerically exact. Precision loss only occurs when a downstream consumer chooses a float64-based JSON decoder.

3. **Horizon comparison is out of scope**: The hypothesis frames this as divergence from Horizon's `json:",string"` convention. But stellar-etl is a separate project with its own published schema. Comparing to Horizon's contract is an interoperability concern, not a data correctness bug within stellar-etl itself.

4. **Practical trigger is speculative**: While `BumpSequenceOp` can technically target any int64, mainnet sequence numbers are currently ~50 million range — orders of magnitude below 2^53. No evidence of real-world sequence numbers exceeding the safe integer boundary.

### Lesson Learned

The "bare JSON number for large int64/uint64 in generic details map" pattern has been definitively classified as by-design behavior for this repo (see H012 rejection). Future hypotheses about other fields in the `Details` map (effect or operation details) following this same pattern should be considered duplicative unless they can demonstrate that the ETL itself produces incorrect values, not merely that downstream float64-based parsers may round them.
