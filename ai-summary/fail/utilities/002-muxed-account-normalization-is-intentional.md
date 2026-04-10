# H002: Muxed account normalization is intentional and lossless in current exports

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For transactions and operations that use muxed accounts, the ETL should preserve both the canonical account identity and the muxed `M...` form where appropriate. Exported rows should not silently drop the muxed identifier if downstream users need it.

## Mechanism

`GetAccountAddressFromMuxedAccount` calls `MuxedAccount.ToAccountId()`, which strips the muxed ID and returns the underlying `G...` account. That initially looked like a source-account corruption bug for muxed transactions and operations.

## Trigger

Export a transaction or operation whose source account or fee account is encoded as a muxed account.

## Target Code

- `internal/utils/main.go:GetAccountAddressFromMuxedAccount:50-53` — canonicalizes muxed accounts to `AccountId`
- `github.com/stellar/go-stellar-sdk/xdr/muxed_account.go:ToAccountId:177-191` — explicitly drops the muxed memo ID
- `internal/transform/transaction.go:TransformTransaction:273-280` — populates `AccountMuxed` when the source is muxed
- `internal/transform/operation.go:TransformOperation:40-47` — populates `SourceAccountMuxed` when the source is muxed

## Evidence

The utility helper always returns the canonical `G...` account even for `M...` inputs. That is a real normalization step, not a pass-through.

## Anti-Evidence

The transform layer already exports dedicated muxed-account fields (`AccountMuxed`, `FeeAccountMuxed`, `SourceAccountMuxed`) alongside the canonical account fields. The upstream SDK comment on `ToAccountId()` explicitly says it drops the memo ID by design.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Current export paths intentionally emit both representations: the base `G...` account for canonical joins and the `M...` address in dedicated muxed columns, so no identifier is lost.

### Lesson Learned

Muxed-account normalization is only a bug when a caller fails to persist the raw muxed address anywhere. In this repo, the utility helper is paired with explicit muxed fields at the relevant call sites.
