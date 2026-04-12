# H021: `AccountSignersChanged` looked order-sensitive, but no live reorder trigger emerged

**Date**: 2026-04-12
**Subsystem**: utilities
**Severity**: High
**Impact**: signer change classification
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If an account update only reordered signer storage while leaving signer keys, weights, and sponsors unchanged, `AccountSignersChanged()` should return `false` so `export_ledger_entry_changes` would not emit synthetic `account_signers` rows for a non-change. A pure storage-order shuffle should not appear as a signer update in exported history.

## Mechanism

The helper first compares signer weights through `SignerSummary()` maps, but then compares sponsorship descriptors positionally by slice index. That makes the function look vulnerable to false positives if the same logical signer set were ever serialized in a different order between `Pre` and `Post`.

## Trigger

Construct an account update whose `Pre` and `Post` signer sets are identical except for array order, while sponsorship assignments remain logically the same. If that state were reachable, `AccountSignersChanged()` would likely report a change because it compares `SignerSponsoringIDs()` entries by index.

## Target Code

- `internal/utils/main.go:AccountSignersChanged:1051-1119` — compares signer weights by map, then sponsorships by positional slice index
- `cmd/export_ledger_entry_changes.go:148-156` — emits signer rows whenever `AccountSignersChanged()` returns true
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/account_entry.go:61-86` — SDK models signer sponsorships as a slice aligned with `account.Signers`

## Evidence

The suspicious part is real: `AccountSignersChanged()` iterates `preSignerSponsors[i]` and `postSignerSponsors[i]`, so a reorder-only difference would be observed as a sponsorship change even when the key/sponsor mapping was logically unchanged. The SDK also makes sponsorship descriptors positional relative to the signer slice, which is exactly the kind of representation that makes order sensitivity easy to miss.

## Anti-Evidence

I did not find a live protocol path or exporter input that can produce a reorder-only account update with identical signer semantics. The SDK itself treats sponsorship descriptors as positionally coupled to `account.Signers`, and the downstream signer export sorts rows before writing, so the only plausible trigger is a hypothetical storage permutation rather than a demonstrated on-chain state transition.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The analysis never established a realistic Stellar/account-entry transition that reorders signer arrays without also changing the effective signer-to-sponsor mapping. Without a reachable trigger, the positional comparison remains a speculative implementation concern rather than a live data-integrity bug.

### Lesson Learned

For signer helpers, positional slice comparisons are only actionable if the surrounding protocol can emit logically equivalent signer sets in different orders. When the SDK models one slice as inherently aligned to another, a reorder-only hypothesis needs concrete protocol evidence before it qualifies as a finding.
