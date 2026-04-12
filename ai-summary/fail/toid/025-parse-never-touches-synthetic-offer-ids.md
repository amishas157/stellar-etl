# H025: `toid.Parse` misdecodes synthetic offer IDs into bogus ledger/tx/op tuples

**Date**: 2026-04-12
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Synthetic offer IDs created for trade rows should either never be fed into TOID decoding or should be decoded with `DecodeOfferID`, not with `Parse`. If a caller accidentally interprets a synthetic offer ID as a TOID, the decoded `(ledger, transaction, operation)` tuple would be meaningless and any derived output would be wrong.

## Mechanism

`EncodeOfferId` reserves the high bits of an `int64` to tag synthetic offer IDs, while `Parse` blindly interprets those same bits as TOID ledger/transaction/operation fields. Because trade rows contain both `history_operation_id` and synthetic `buying_offer_id` values side-by-side, it looked plausible that some caller might reuse `Parse` on the wrong identifier family.

## Trigger

Find a production path that parses a trade `buying_offer_id` or `selling_offer_id` with `toid.Parse(...)`, then compare the decoded tuple against the original operation that created the synthetic offer ID.

## Target Code

- `internal/toid/synt_offer_id.go:28-41` — synthetic offer IDs intentionally occupy the top bits of the shared `int64` space
- `internal/toid/main.go:165-168` — `Parse` has no type-discriminator guard and will decode any positive `int64`
- `cmd/export_trades.go:40` — only production `Parse` caller in this repository

## Evidence

The encoding schemes are mechanically incompatible: `EncodeOfferId` sets bit 62 as a type tag, while `Parse` treats bits 32-63 as the ledger field. A mistaken cross-use would yield plausible-looking but incorrect TOID components without any explicit error.

## Anti-Evidence

Repository-wide search found only one production `Parse` call, and it parses `tradeInput.OperationHistoryID`, not any offer ID field. No transform or command decodes synthetic offer IDs with `Parse`, and `DecodeOfferID` itself has no production callers either.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The dangerous misdecode exists only as a latent helper mismatch; there is no live caller that feeds synthetic offer IDs into `Parse`, so current exports cannot corrupt rows through this mechanism.

### Lesson Learned

When one subsystem stores multiple identifier families in `int64`, helper misuse is only a live bug if a real caller crosses the families. Search for actual decode call sites before escalating an encoding-scheme overlap into a data-corruption finding.
