# H002: Newly added signer rows are mislabeled as `UPDATED`

**Date**: 2026-04-10
**Subsystem**: utilities / data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an existing account adds a new signer, the newly introduced `account_signers` row should be exported with `ledger_entry_change = CREATED` and `deleted = false`. The exporter should distinguish a first appearance of a signer from a modification of a signer that already existed.

## Mechanism

`TransformSigners()` does not diff for post-only signers at all. Every signer present in `Post` is emitted in the first pass with the parent account's `LedgerEntryUpdated` change type, and the diff branch only adds synthetic removals for pre-only signers. That means newly added signers are serialized as ordinary updates even though there is no pre-state signer row to update.

## Trigger

Process an `AccountEntry` `LedgerEntryUpdated` change where `Pre` lacks signer `S` and `Post` includes signer `S` (for example a `SetOptions` operation that adds a new signer to an existing account). The correct signer row should be a creation, but the current output marks it as an update.

## Target Code

- `internal/transform/account_signer.go:35-52` — emits all post-state signers with `LedgerEntryChange: uint32(changeType)`
- `internal/transform/account_signer.go:57-86` — update-specific diff only handles `Pre`-only signers and never upgrades `Post`-only signers to `CREATED`

## Evidence

For update changes, `changeType` is always `LedgerEntryUpdated`, so any signer found only in `Post` still leaves the first loop with `ledger_entry_change = 1`. There is no complementary branch that checks `if _, existed := preSigners[signer]; !existed` and rewrites the row to `CREATED`.

## Anti-Evidence

Some downstream loaders may treat `CREATED` and `UPDATED` identically as upserts, which would reduce operational impact. But the export still advertises the wrong row-level change kind, and that corruption matters for consumers that analyze signer lifecycle events or count newly added signers per ledger.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformSigners` in `internal/transform/account_signer.go` processes account ledger changes and emits per-signer rows. For `LedgerEntryUpdated` changes, the first loop (lines 35-52) emits every post-state signer with `LedgerEntryChange = 1` (UPDATED). The second block (lines 57-86) diffs Pre vs Post to synthesize REMOVED rows for deleted signers, but contains no symmetric logic to relabel post-only signers as CREATED (`0`). The asymmetry is confirmed by the test file, which only exercises the deletion scenario — no test adds a signer to an existing account during an update.

### Code Paths Examined

- `internal/transform/account_signer.go:14-52` — `TransformSigners` first loop: iterates `accountEntry.SignerSummary()` (Post state) and assigns `LedgerEntryChange: uint32(changeType)` uniformly to all signers, regardless of whether they existed in Pre
- `internal/transform/account_signer.go:57-86` — diff block: iterates `preAccountEntry.SignerSummary()` and checks `if _, stillExists := postSigners[signer]; stillExists` to skip retained signers, only emitting REMOVED rows for pre-only signers. No complementary post-only check exists.
- `internal/utils/main.go:854-865` — `ExtractEntryFromChange`: returns the account-level `changeType` directly; the function works correctly but provides no per-signer granularity
- `internal/transform/account_signer_test.go:105-119` — test input for UPDATED: Pre has 2 signers + master, Post removes signer B. No test case adds a new signer to Post that wasn't in Pre.

### Findings

The bug is a confirmed asymmetry in `TransformSigners`. The code correctly synthesizes REMOVED rows for signers deleted during an account update (code comment at lines 54-56 explicitly documents this intent: "we must diff Pre vs Post to surface them explicitly"). However, it does not perform the symmetric operation for newly added signers: checking which post-state signers are absent from pre-state and relabeling them as CREATED.

Concretely, when a `SetOptions` operation adds signer S to an existing account:
- The account change type is `LedgerEntryUpdated` (1)
- Signer S appears in Post but not in Pre
- The first loop emits S with `LedgerEntryChange = 1` (UPDATED)
- The diff block only processes pre-only signers, never touching S
- Result: S is exported with `ledger_entry_change = 1` instead of `0`

This affects any downstream consumer that distinguishes CREATED from UPDATED for signer lifecycle analysis (e.g., counting newly added signers per ledger, auditing when signers first appeared on accounts).

### PoC Guidance

- **Test file**: `internal/transform/account_signer_test.go`
- **Setup**: Construct an `ingest.Change` with `ChangeType = LedgerEntryUpdated` where Pre has signer A + master, and Post has signer A + signer B + master (signer B is newly added). Use the existing `makeTestLedgerEntry()` helper, modifying Post to add a third signer not present in Pre.
- **Steps**: Call `TransformSigners(change, header)` and find the output row for signer B.
- **Assertion**: Assert that signer B's `LedgerEntryChange` equals `uint32(xdr.LedgerEntryChangeTypeLedgerEntryCreated)` (0). The current code will produce `uint32(xdr.LedgerEntryChangeTypeLedgerEntryUpdated)` (1), demonstrating the mislabeling.
