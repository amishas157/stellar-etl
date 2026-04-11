# H002: Config-setting export zeroes `ContractParallelCompute` rows

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `ConfigSettingEntry` uses `config_setting_id = 14` (`ConfigSettingIdConfigSettingContractParallelComputeV0`), the export should preserve `ledger_max_dependent_tx_clusters` instead of emitting a row whose only meaningful value is the ID. A row carrying a non-zero parallel-compute limit should round-trip that limit into JSON/Parquet output.

## Mechanism

The XDR union has a dedicated `ContractParallelCompute` arm, but `TransformConfigSetting()` never calls `GetContractParallelCompute()` and `ConfigSettingOutput` has no field for `LedgerMaxDependentTxClusters`. Because the export still writes a `ConfigSettingOutput` row for the entry, downstream consumers receive a valid-looking row that silently discards the active arm's only value.

## Trigger

Export any ledger-entry change whose config-setting payload is:

1. `ConfigSettingId = ConfigSettingIdConfigSettingContractParallelComputeV0`
2. `LedgerMaxDependentTxClusters > 0`

The emitted row will retain `config_setting_id = 14` but lose the cluster limit entirely.

## Target Code

- `internal/transform/config_setting.go:24-107` — transform never calls `GetContractParallelCompute()`
- `internal/transform/config_setting.go:116-178` — output struct construction has no parallel-compute field
- `internal/transform/schema.go:564-627` — `ConfigSettingOutput` has no column for `ledger_max_dependent_tx_clusters`
- `cmd/export_ledger_entry_changes.go:245-256` — config-setting rows are still exported for every config-setting entry
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:59704-59706` — `ConfigSettingContractParallelComputeV0` contains `LedgerMaxDependentTxClusters`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61430-61431` — config-setting ID 14 maps to the `ContractParallelCompute` arm
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:61931-61939` — SDK exposes `GetContractParallelCompute()`, but the transform ignores it

## Evidence

Unlike older arms, `ContractParallelCompute` is absent from both the transform logic and the schema definition. The current code therefore has no path that can propagate `LedgerMaxDependentTxClusters` into exported output even though the XDR union arm and getter already exist in the SDK.

## Anti-Evidence

If current ledgers do not yet emit `config_setting_id = 14`, the issue will not show up in today's sample exports. But once the arm appears, the export path will not fail loudly; it will emit a normal-looking row with the arm payload missing.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-integrity/005-config-parallel-compute-empty-shell.md.gh-published`
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug as the already-published finding `data-integrity/005-config-parallel-compute-empty-shell`. Both identify that `TransformConfigSetting()` never calls `GetContractParallelCompute()`, that `ConfigSettingOutput` has no field for `LedgerMaxDependentTxClusters`, and that config_setting_id=14 rows are emitted as empty shells with only metadata. The published finding already includes a complete PoC test, adversarial review, and suggested fix.

### Code Paths Examined

- `internal/transform/config_setting.go:24-107` — confirmed no call to `GetContractParallelCompute()`
- `internal/transform/schema.go:563-627` — confirmed no `LedgerMaxDependentTxClusters` field in `ConfigSettingOutput`
- `ai-summary/success/data-integrity/005-config-parallel-compute-empty-shell.md.gh-published` — identical root cause, mechanism, affected code, and impact

### Why It Failed

This is a duplicate of an already-confirmed and published finding. The published success file `data-integrity/005-config-parallel-compute-empty-shell` covers the identical root cause (missing `GetContractParallelCompute()` call), the identical affected schema (no destination column for `ledger_max_dependent_tx_clusters`), and the identical impact (empty shell rows for config_setting_id=14). No new information is contributed by this hypothesis.

### Lesson Learned

Cross-check `ai-summary/success/` across all subsystems (not just the target subsystem) before submitting, as the same underlying bug may have been discovered and published under a different subsystem classification (here `data-integrity` vs `data-transform`).
