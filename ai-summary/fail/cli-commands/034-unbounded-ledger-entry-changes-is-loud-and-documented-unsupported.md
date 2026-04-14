# H034: `export_ledger_entry_changes` silently enters broken unbounded mode when `--end-ledger` is omitted

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: empty or partial continuous change exports
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_entry_changes` claims to support an unbounded mode when `--end-ledger` is omitted, the command should switch to an unbounded ledger range and keep exporting later batches rather than preparing a bounded range ending at `0`.

## Mechanism

The command calls `backend.PrepareRange(ctx, ledgerbackend.BoundedRange(startNum, commonArgs.EndNum))` before it rewrites `EndNum == 0` to `math.MaxInt32`, so the initial prepare step receives `[start,0]`. That suggests a broken control-flow path where the command could look like it entered continuous mode but actually operate on an invalid bounded range.

## Trigger

Run `export_ledger_entry_changes` with only `--start-ledger` and the additional config flags the command requires for "continuous" operation.

## Target Code

- `cmd/export_ledger_entry_changes.go:26-30` — long command description says omitting end-ledger continues exporting
- `cmd/export_ledger_entry_changes.go:67-74` — prepares `BoundedRange(start, end)` before rewriting `end==0`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/range.go:76-83` — bounded and unbounded ranges are distinct constructors
- `README.md:301-303` — marks unbounded mode as "Currently Unsupported"

## Evidence

The command does prepare a bounded range with `end==0` before any unbounded fallback is applied, and the ledger-backend API clearly distinguishes `BoundedRange` from `UnboundedRange`.

## Anti-Evidence

This path is not silent or success-shaped. The README already documents unbounded mode as currently unsupported, and the command's current implementation fails loudly rather than producing plausible bounded output that looks correct.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The discrepancy is real, but it is both documented as unsupported and expected to fail loudly during backend preparation or the first ledger fetch. That makes it an unsupported-mode/control-flow bug, not a silent data-integrity issue.

### Lesson Learned

For command modes already documented as unsupported, a broken prepare path is not enough on its own. To qualify for this objective, the command must continue and emit success-shaped wrong data rather than loudly rejecting or crashing.
