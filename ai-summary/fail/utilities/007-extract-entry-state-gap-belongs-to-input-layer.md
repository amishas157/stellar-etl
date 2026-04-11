# H007: `ExtractEntryFromChange` state-gap belongs to the input layer, not utilities

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a shared helper is responsible for decoding live ledger-entry changes, it should either support `STATE` changes or receive inputs that provably exclude them. A ledger-entry export should not silently mis-handle a legitimate state snapshot.

## Mechanism

`ExtractEntryFromChange()` only accepts created, updated, removed, and restored changes. I initially suspected that a live `LedgerEntryState` change could flow through this helper into account/trustline/config-setting exports and cause plausible-but-wrong rows or dropped records.

## Trigger

Run a ledger-entry export over a ledger whose raw close meta contains `LedgerEntryState` changes for one of the exported ledger-entry types.

## Target Code

- `internal/utils/main.go:ExtractEntryFromChange:854-864` — rejects all change types outside created/updated/removed/restored
- `internal/input/changes.go:extractBatch:101-145` — runs every live change through `ingest.ChangeCompactor` before utilities helpers see it
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor.go:AddChange:97-109` — the compactor itself rejects unknown change states

## Evidence

The utility helper really does return an error on `LedgerEntryState`. The upstream SDK compactor also rejects that change type, and `extractBatch()` ignores the `AddChange()` error instead of surfacing it.

## Anti-Evidence

Current production callers of `ExtractEntryFromChange()` consume compacted change streams, not raw state-change streams. That means the reachable corruption point, if one exists, is the ignored `AddChange()` error in `internal/input/changes.go`, not the utilities helper itself.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This investigation did not land on a utilities-owned bug. `ExtractEntryFromChange()` is stricter than I first expected, but the live path that would make that strictness matter is upstream in the input layer's compaction/error-handling behavior.

### Lesson Learned

When a utilities helper looks incomplete, verify whether an upstream normalizer constrains the helper's input first. Here, the sharper question is whether the compactor error path is mishandled, not whether `ExtractEntryFromChange()` itself should decode raw `STATE` changes.
