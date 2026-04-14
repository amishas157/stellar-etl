# H003: Archive export commands silently skip ledger 1 when `--start-ledger` is omitted

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: High
**Impact**: wrong export boundary / missing genesis rows
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For archive-based commands whose `start-ledger` flag description says the export period defaults to the genesis ledger, omitting `--start-ledger` should begin at ledger `1`, not silently exclude the genesis ledger from the exported range.

## Mechanism

`AddArchiveFlags()` hard-codes the default `start-ledger` to `2`, but the flag description still says "Defaults to genesis ledger." All archive export commands consume that default through `MustArchiveFlags()` and then iterate from the returned `start` value without any later ledger-1 correction, so a caller who relies on the flag contract gets a normal-looking export that begins at ledger 2.

## Trigger

Run a ledger export without `--start-ledger`, for example `stellar-etl export_ledgers --end-ledger 5`. The command will begin at ledger `2`, so the output omits the genesis ledger row even though the shared flag contract says the start defaults to the genesis ledger.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — sets `start-ledger` default to `2` while describing it as "Defaults to genesis ledger"
- `internal/utils/main.go:MustArchiveFlags:541-562` — returns that default unchanged to callers
- `cmd/export_ledgers.go:21-31` — representative archive-export command that uses the returned `startNum` directly
- `internal/input/ledgers.go:GetLedgers:22-90` — iterates from `start` exactly as provided, with no special-case to include ledger `1`

## Evidence

The shared flag registration is internally inconsistent: the description says genesis, but the actual default is numeric `2`. Downstream command code does not reinterpret that default; it simply passes `startNum` into the input readers, which loop `for seq := start; seq <= end; seq++`.

## Anti-Evidence

The README table also shows the default value as `2`, and captive-core itself cannot start from ledger `1`, so maintainers may have intentionally chosen `2` as a practical default. Even so, for archive commands that can otherwise accept an explicit `--start-ledger 1`, the current default still produces a success-shaped export boundary that contradicts the flag's verbal contract.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/fail/utilities/summary.md` entry 005
**Failed At**: reviewer

### Trace Summary

Traced `AddArchiveFlags()` at `internal/utils/main.go:251` which sets `start-ledger` default to `2` with description "Defaults to genesis ledger". Verified `MustArchiveFlags()` returns this value unchanged. Confirmed all archive export commands pass `startNum` directly to input readers which iterate `for seq := start; seq <= end; seq++`. However, this exact hypothesis was already investigated and rejected under the utilities subsystem.

### Code Paths Examined

- `internal/utils/main.go:AddArchiveFlags:251` — default is `2`, description says "genesis ledger"
- `internal/input/ledger_range.go:55-57` — comment explains: "Ledger sequence 2 is the start ledger because the genesis ledger (ledger 1), has a close time of 0 in Unix time"
- `internal/input/ledger_range.go:80` — "the second ledger has a real close time, unlike the 1970s close time of the genesis ledger"
- `internal/utils/main.go:738` — `ValidateLedgerRange` acknowledges "genesis ledger is ledger 1" but does not enforce starting from it

### Why It Failed

This is a duplicate of `fail/utilities/summary.md` entry 005, which concluded: ledger 1 (genesis) has a Unix close time of 0 (epoch 1970), contains no transactions, no operations, and no trades. The entire codebase consistently treats ledger 2 as the practical start — `ledger_range.go` explicitly starts binary search from ledger 2 and the graph's `BeginPoint` is set to ledger 2. The default of `2` is an intentional design choice. The flag description "Defaults to genesis ledger" is a documentation inaccuracy, not a data-correctness bug, since omitting ledger 1 produces no incorrect output — it simply omits a row with no meaningful content.

### Lesson Learned

Check `fail/` directories across all subsystems (not just the hypothesis's own subsystem) for prior investigations of the same root cause. This genesis-ledger-skip hypothesis was previously filed under `utilities` rather than `cli-commands`.
