# H068: Revoke-sponsorship subtype gap is future-only

**Date**: 2026-04-14
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For every currently valid `REVOKE_SPONSORSHIP` operation, the exporter should emit subtype-specific detail fields describing either the revoked ledger entry or the revoked signer.

## Mechanism

Both revoke-sponsorship detail formatters switch only on the two known `RevokeSponsorshipType` values and have no default branch. If a legitimate current ledger could carry a third subtype, the exporter would still emit an otherwise plausible operation row but with missing subtype details, creating silent structural corruption instead of a hard failure.

## Trigger

Process a `REVOKE_SPONSORSHIP` operation whose `Type` is not `RevokeSponsorshipLedgerEntry` or `RevokeSponsorshipSigner`.

## Target Code

- `internal/transform/operation.go:912-922` — primary transform only handles the two known revoke-sponsorship subtypes
- `internal/transform/operation.go:1568-1578` — wrapper details path mirrors the same two-case switch
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:27577-27600` — current `RevokeSponsorshipType` enum only defines two values

## Evidence

Unlike the other enum fallbacks, these switches do not panic or return an error. A real missing subtype would therefore degrade into sparse detail maps that still look like valid exported rows.

## Anti-Evidence

The current generated XDR enum has only two revoke-sponsorship subtypes, and both are implemented in both transform paths. No present-day legitimate ledger can produce the silent omission scenario without a future protocol change.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The silent-omission path requires a third revoke-sponsorship subtype, but the current SDK/XDR only defines the two arms already handled by the exporter.

### Lesson Learned

Switches without defaults are only actionable when the current enum already has an uncovered value. Future-only sparse-detail risks should be documented, but they are not live data-correctness bugs on today's chain formats.
