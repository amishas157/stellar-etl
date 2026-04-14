# H001: `manage_data` with no data-entry diff fabricates an `account_created` effect

**Date**: 2026-04-14
**Subsystem**: external-io
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a successful `manage_data` operation completes without any `LedgerEntryTypeData` change for that operation, the exporter should either emit no data effect or emit a `data_created` / `data_updated` / `data_removed` effect only when it finds the corresponding data-entry diff. It should never emit an unrelated effect type such as `account_created`.

## Mechanism

`addManageDataEffects()` initializes `effect` to `EffectType(0)` before scanning operation changes and then unconditionally calls `e.addMuxed(source, effect, details)` even when the loop never finds a `LedgerEntryTypeData` entry. In this codebase, `EffectType(0)` is not a neutral sentinel: it is `account_created`, so any successful `manage_data` shape that produces no data-entry diff would silently export a plausible but false account-creation effect row.

## Trigger

Run `export_effects` over a ledger containing a successful `manage_data` operation whose metadata has no `LedgerEntryTypeData` change for that operation index, for example a same-value rewrite or another no-op shape that core accepts without mutating the data entry. The exported effect row should show the `manage_data` entry name in `details`, but `type=0` / `type_string="account_created"` instead of a `data_*` effect or no row.

## Target Code

- `internal/transform/effects.go:addManageDataEffects:760-798` — initializes `effect := EffectType(0)`, scans for `LedgerEntryTypeData`, then always appends an effect
- `internal/transform/effects.go:add:176-184` — serializes the raw enum into exported `type` / `type_string`
- `internal/transform/schema.go:377-399` — defines `EffectAccountCreated = 0` and `EffectDataCreated/Removed/Updated = 40/41/42`
- `internal/transform/schema.go:433-456` — maps effect enum `0` to `"account_created"`

## Evidence

The only assignment that changes the default `effect` happens inside the `for _, change := range changes` loop and only after `change.Type == xdr.LedgerEntryTypeData`. If that predicate never matches, `details` still contains the `manage_data` key name and the function still emits a row, but the serialized effect is whatever the zero enum means in `EffectTypeNames` — here, `account_created`.

## Anti-Evidence

The trigger depends on a real successful `manage_data` operation that produces no data-entry diff. Upstream Horizon currently carries the same assumption in its effects processor, so this may rely on a narrow protocol edge case rather than a common ledger shape.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `addManageDataEffects()` at effects.go:760-798 confirming the code pattern: `effect` is initialized to `EffectType(0)` (which equals `EffectAccountCreated`), only reassigned inside a loop that filters for `LedgerEntryTypeData` changes, and then `e.addMuxed()` is called unconditionally. However, the `effects()` dispatcher at line 55 gates all effect generation behind `operation.transaction.Result.Successful()`, meaning only successful operations reach this code. Under the Stellar protocol, a successful `manage_data` operation ALWAYS produces exactly one `LedgerEntryTypeData` change — there is no no-op success case.

### Code Paths Examined

- `internal/transform/effects.go:effects:54-57` — gates on `Successful()`, so failed operations never reach `addManageDataEffects`
- `internal/transform/effects.go:addManageDataEffects:760-798` — confirmed `effect := EffectType(0)` initialization and unconditional `addMuxed` call after loop
- `internal/transform/schema.go:378` — confirmed `EffectAccountCreated = 0`
- `internal/transform/schema.go:397-399` — confirmed `EffectDataCreated = 40`, `EffectDataRemoved = 41`, `EffectDataUpdated = 42`

### Why It Failed

The trigger is unreachable under the Stellar protocol. A successful `manage_data` operation has exactly three outcomes: create a data entry (DataValue set, entry absent → `before=nil, after!=nil`), update a data entry (DataValue set, entry exists → `before!=nil, after!=nil`), or remove a data entry (DataValue nil, entry exists → `before!=nil, after=nil`). The only "no change" case — DataValue nil and entry absent — results in `MANAGE_DATA_NAME_NOT_FOUND`, a failed operation that is filtered out at line 55. Even a same-value rewrite updates `lastModifiedLedgerSeq` in the `LedgerEntry` header, so it still produces a `LedgerEntryTypeData` change. There is no protocol-valid path that reaches `addManageDataEffects()` without a matching data entry change.

### Lesson Learned

Stellar protocol semantics guarantee that successful `manage_data` operations always produce a `LedgerEntryTypeData` change. While the `EffectType(0)` default is technically a latent defensive-programming gap, it is unreachable for valid ledger data. Hypotheses about zero-value sentinel confusion require proving the trigger path is reachable under the protocol, not just that the code lacks a guard.
