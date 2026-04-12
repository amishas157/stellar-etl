# H002: Revoke-sponsorship details drop Soroban ledger-key targets

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If Stellar supported `revoke_sponsorship` for Soroban ledger keys such as `ContractData`, `ContractCode`, `ConfigSetting`, or `Ttl`, the operation details export should include identifying fields for the targeted key instead of emitting an almost empty `details` object.

## Mechanism

This looked viable because both `addLedgerKeyToDetails()` and `addLedgerKeyDetails()` only serialize classic ledger-key types. But after tracing the live operation surface, the suspected gap is unreachable: the current SDK revoke-sponsorship builder/parser only enumerates the classic target set (account, trustline, offer, data, claimable balance, signer), and current public revoke-sponsorship docs describe that same classic set. Without a legitimate Soroban-key revoke path, the missing detail branches do not corrupt any current export.

## Trigger

1. Inspect `internal/transform/operation.go` and note that revoke-sponsorship details handle only classic ledger keys.
2. Compare against the current SDK revoke-sponsorship surface in `txnbuild/revoke_sponsorship.go`.
3. Check the current public revoke-sponsorship documentation and observe that Soroban ledger-key targets are not documented as supported.

## Target Code

- `internal/transform/operation.go:468-510` — `addLedgerKeyToDetails()` only handles classic ledger-key types
- `internal/transform/operation.go:912-922` — free-function revoke-sponsorship export uses `addLedgerKeyToDetails()`
- `internal/transform/operation.go:1568-1578` — wrapper revoke-sponsorship export uses `addLedgerKeyDetails()`
- `internal/transform/operation.go:2080-2124` — `addLedgerKeyDetails()` likewise only handles classic ledger-key types
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/txnbuild/revoke_sponsorship.go:13-20` — supported revoke-sponsorship types enum
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/txnbuild/revoke_sponsorship.go:58-159` — builder only constructs classic ledger-key revocations
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/txnbuild/revoke_sponsorship.go:167-213` — decoder only accepts the same classic ledger-key set

## Evidence

The operation helpers clearly lack Soroban-key serialization branches. That initially suggested missing data for newer ledger-entry types. But the current SDK construction/parsing layer for revoke-sponsorship never exposes `ContractData`, `ContractCode`, `ConfigSetting`, or `Ttl` targets, and I did not find a live, documented transaction shape that would route such keys through the exporter.

## Anti-Evidence

At the raw XDR level, `RevokeSponsorshipOp` still carries a generic `LedgerKey`, so the transform code shape does look expandable. That generic union is why this was worth checking even though the live builder/parser surface still constrains the supported targets.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected omission depends on a live operation variant that the current SDK and public operation docs do not expose. Because legitimate revoke-sponsorship traffic does not currently target Soroban ledger keys, the missing detail branches are unreachable for current exports.

### Lesson Learned

Before treating a missing detail serializer as a live export bug, confirm that the operation builder/parser surface and current protocol documentation actually admit the target variant. Generic XDR unions often outlive the supported transaction shapes visible to production exporters.
