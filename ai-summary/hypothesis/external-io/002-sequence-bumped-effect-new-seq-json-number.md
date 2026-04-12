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
