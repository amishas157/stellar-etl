# H004: Config-setting export drops `tx_max_footprint_entries` from ledger-cost-ext rows

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `config_setting_id = 15` (`ConfigSettingIdConfigSettingContractLedgerCostExtV0`), the exported row should preserve both members of the XDR arm: `tx_max_footprint_entries` and `fee_write_1kb`. A ledger-cost-ext row with a non-zero footprint-entry cap should not lose that cap while still exporting the sibling fee field.

## Mechanism

`TransformConfigSetting()` does call `GetContractLedgerCostExt()`, but it only reads `FeeWrite1Kb`. The output schema likewise contains `fee_write_1kb` but no field for `TxMaxFootprintEntries`, so current exports silently retain half of the arm and discard the other half.

## Trigger

Export any config-setting ledger entry where:

1. `ConfigSettingId = ConfigSettingIdConfigSettingContractLedgerCostExtV0`
2. `TxMaxFootprintEntries > 0`
3. `FeeWrite1Kb` is also populated

The emitted row will preserve `fee_write_1kb` but will have no column containing the footprint-entry limit.

## Target Code

- `internal/transform/config_setting.go:35,52` — transform extracts `GetContractLedgerCostExt()` but uses only `FeeWrite1Kb`
- `internal/transform/config_setting.go:138-140` — row construction writes `FeeWrite1Kb` and never exposes `TxMaxFootprintEntries`
- `internal/transform/schema.go:583-588,602-627` — output schema has `fee_write_1kb` but no `tx_max_footprint_entries`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:59996-59999` — `ConfigSettingContractLedgerCostExtV0` defines both `TxMaxFootprintEntries` and `FeeWrite1Kb`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61432-61433` — config-setting ID 15 maps to the `ContractLedgerCostExt` arm

## Evidence

This path is especially easy to miss because it is only a partial omission: the row still exports one valid field from the active arm, so the output looks plausible. But the sibling `TxMaxFootprintEntries` value has no destination anywhere in the transform or schema.

## Anti-Evidence

If networks happen to keep `TxMaxFootprintEntries = 0`, the missing column may not be noticed in ad hoc samples. Even then, the omission remains live because the transform has no way to preserve a non-zero value once the network starts using one.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/export-pipeline/005-config-ledger-cost-ext-drops-footprint-cap.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug as the already-published finding `export-pipeline/005-config-ledger-cost-ext-drops-footprint-cap`. Both identify that `TransformConfigSetting()` calls `GetContractLedgerCostExt()` but only copies `FeeWrite1Kb`, that `ConfigSettingOutput` has no `tx_max_footprint_entries` field, and that config_setting_id=15 rows silently drop the footprint cap while still exporting the sibling fee. The published finding already includes a complete PoC test and suggested fix.

### Code Paths Examined

- `internal/transform/config_setting.go:34-52` — confirmed `GetContractLedgerCostExt()` is called but only `FeeWrite1Kb` is extracted
- `internal/transform/schema.go:563-627` — confirmed no `tx_max_footprint_entries` field in `ConfigSettingOutput`
- `ai-summary/success/export-pipeline/005-config-ledger-cost-ext-drops-footprint-cap.md.gh-published` — identical root cause, mechanism, affected code, and impact

### Why It Failed

This is a duplicate of an already-confirmed and published finding. The published success file `export-pipeline/005-config-ledger-cost-ext-drops-footprint-cap` covers the identical root cause (only `FeeWrite1Kb` extracted from `ContractLedgerCostExt`), the identical affected schema (no destination column for `tx_max_footprint_entries`), and the identical impact (incomplete but plausible rows for config_setting_id=15). No new information is contributed by this hypothesis.

### Lesson Learned

Cross-check `ai-summary/success/` across all subsystems (not just the target subsystem) before submitting, as the same underlying bug may have been discovered and published under a different subsystem classification (here `export-pipeline` vs `data-transform`).
