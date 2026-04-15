# H003: `export_assets` marks assets as seen before confirming the JSON row was written

**Date**: 2026-04-15
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_assets` should only record an asset ID in `seenIDs` after the asset row has been durably emitted, so a later occurrence of the same asset can still rescue the export if the first output attempt failed.

## Mechanism

The command sets `seenIDs[transformed.AssetID] = true` before calling `ExportEntry(...)`. `ExportEntry` already logs and suppresses file-write failures, so a transient JSON write error can still return `nil`; once that happens, later operations for the same asset are skipped as duplicates even though no JSON row was actually persisted, and the optional parquet slice can still retain the first transformed row, creating JSON/parquet divergence for the same export.

## Trigger

Export a range containing at least two qualifying operations for the same asset, enable `--write-parquet`, and inject a post-open JSON write failure on the first occurrence (for example, a quota/full-disk/FUSE-backed write error that `ExportEntry` logs but does not return). The second occurrence is skipped because `seenIDs` was already set by the failed first attempt.

## Target Code

- `cmd/export_assets.go:assetsCmd.Run:39-69` — deduplicates by `AssetID`, sets `seenIDs` before `ExportEntry`, and appends parquet rows after the same call
- `cmd/command_utils.go:ExportEntry:55-86` — logs `Write`/`WriteString` failures but still returns `nil`

## Evidence

The ordering is `seenIDs[...] = true` first, then `ExportEntry(...)`, then optional parquet append. That means any first-write failure hidden by `ExportEntry` permanently suppresses later retries for that asset within the same run, even though later payment or offer operations could have emitted the same asset successfully.

## Anti-Evidence

This requires both a duplicated asset within the selected ledger range and an I/O failure that takes the existing `ExportEntry` swallow path; if the first occurrence writes successfully, the deduplication is correct and desirable.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — derivative of success/cli-commands/002-export-entry-swallows-write-errors.md.gh-published
**Failed At**: reviewer

### Trace Summary

Traced the full dedup + export path in `export_assets.go` lines 44-70. Confirmed that `seenIDs[transformed.AssetID] = true` is set at line 58 before `ExportEntry` at line 59, and that `ExportEntry` (command_utils.go:78-85) logs write errors but returns nil at line 86. The compound scenario the hypothesis describes — JSON missing a row while Parquet retains it, with later occurrences deduped — is real but entirely predicated on ExportEntry's error-swallowing behavior, which is the exact bug already confirmed and published in success/cli-commands/002.

### Code Paths Examined

- `cmd/export_assets.go:assetsCmd.Run:58` — `seenIDs[transformed.AssetID] = true` set before ExportEntry call
- `cmd/export_assets.go:assetsCmd.Run:59-64` — ExportEntry called, error check branches to continue (but error is nil due to swallowing)
- `cmd/export_assets.go:assetsCmd.Run:67-69` — Parquet append occurs after ExportEntry returns nil, creating divergence
- `cmd/command_utils.go:ExportEntry:78-86` — Write/WriteString errors logged but nil returned (the root cause from success/002)

### Why It Failed

This hypothesis is a **downstream consequence of the already-confirmed ExportEntry error-swallowing bug** (success/cli-commands/002). The entire compound effect — JSON/Parquet divergence plus permanent dedup suppression — is triggered exclusively through ExportEntry returning nil for failed writes, which IS success/002's bug. The seenIDs ordering itself (set before ExportEntry) is not independently triggerable: ExportEntry's only actual error return path (second `json.Marshal` at line 73-75) is practically unreachable for valid transform outputs, since it marshals a `map[string]interface{}` just decoded from a prior successful marshal. Once success/002 is fixed (ExportEntry returns write errors), the code at line 60 catches the error, `continue`s at line 63 (skipping parquet append), and both JSON and Parquet are consistently missing the row — eliminating the divergence. The seenIDs ordering would still be suboptimal (blocking later retry), but only under the already-handled error path, not silently.

### Lesson Learned

When a hypothesis chains on top of an already-confirmed bug (ExportEntry swallowing), it is a derivative consequence, not a novel independent finding. The fix for the root cause (success/002) eliminates the compound effect. Check whether the hypothesized mechanism can be triggered independently of any already-confirmed bug before escalating ordering issues in callers of known-broken functions.
