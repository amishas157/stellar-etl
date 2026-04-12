# H066: Operation detail account fields lose canonical `G...` addresses for muxed accounts

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For operations involving muxed accounts, the exported detail payload should preserve the canonical underlying `G...` account in the base field (`from`, `to`, `funder`, etc.) and, when present, add separate muxed metadata fields rather than overwriting the canonical account. Downstream consumers should be able to join on the base account field without losing muxed context.

## Mechanism

Many operation branches delegate account-field population to `addAccountAndMuxedAccountDetails()`, so I suspected that helper might write the muxed `M...` address directly into the base field and thereby replace the canonical account ID. That would silently corrupt payment-, path-payment-, sponsorship-, and clawback-style detail payloads for muxed accounts.

## Trigger

Run `TransformOperation()` on any payment- or sponsorship-style operation whose source or destination is a muxed account, then inspect the emitted `details` map for `from`, `to`, `funder`, or related account keys.

## Target Code

- `internal/transform/operation.go:addAccountAndMuxedAccountDetails:423-439` — shared helper for operation detail account fields
- `internal/transform/operation.go:extractOperationDetails:596-617` — representative payment/create-account call sites using the helper
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20240111173100-ed7ae81c8546/services/horizon/internal/ingest/processors/operations_processor.go:addAccountAndMuxedAccountDetails:315-326` — upstream Horizon helper with the same pattern

## Evidence

The helper is shared across several operation-detail branches, so a mistake there would affect many exported operation types at once. Muxed-account handling is subtle enough that replacing canonical addresses with `M...` strings would be easy to miss in downstream analytics.

## Anti-Evidence

The helper sets the base field from `a.ToAccountId().Address()` first, which preserves the canonical `G...` account, and only then appends `*_muxed` and `*_muxed_id` siblings for `KEY_TYPE_MUXED_ED25519`. The upstream Horizon processor uses the same canonical-plus-muxed-sibling strategy.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

There is no canonical-address loss here: the base detail key already holds the underlying `G...` address, and muxed context is exported separately in sibling fields. The implementation matches upstream behavior rather than flattening muxed data incorrectly.

### Lesson Learned

Before escalating muxed-account concerns, inspect the shared helper rather than inferring behavior from scattered call sites. This codebase often preserves canonical and muxed representations side-by-side, following the same pattern as upstream Horizon.
