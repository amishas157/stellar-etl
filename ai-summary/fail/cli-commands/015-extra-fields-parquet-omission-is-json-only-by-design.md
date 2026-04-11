# H015: Missing `extra-fields` columns in parquet exports

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `--extra-fields` were meant to decorate every output format, then parquet exports should contain the same extra metadata columns that JSON exports receive. Running a command with both JSON and parquet enabled should not silently drop operator-supplied metadata from one format.

## Mechanism

At first glance, the shared flag plumbing makes this look plausible: all archive commands parse `--extra-fields` and many of them can emit both JSON and parquet for the same ledger range. Since the parquet path bypasses `ExportEntry`, any extra metadata is absent there, which initially looks like a JSON/parquet parity bug.

## Trigger

Run a parquet-capable export with `--extra-fields source=batch42 --write-parquet` and compare the JSON and parquet outputs for the same row.

## Target Code

- `internal/utils/main.go:AddCommonFlags:232-245` — registers `extra-fields`
- `cmd/command_utils.go:ExportEntry:55-86` — only the JSON helper merges extra fields
- `cmd/command_utils.go:WriteParquet:162-180` — parquet path never sees the extra metadata map
- `README.md:155-165` — user-facing flag description

## Evidence

The JSON helper explicitly merges `extra` into the output map, while `WriteParquet` only serializes transform structs and has no parameter for extra metadata. That creates a visible format difference whenever an operator supplies extra fields.

## Anti-Evidence

The CLI help text and README both describe `extra-fields` as metadata appended to **output jsons**, not to all output formats. That documentation makes the JSON-only behavior an intended feature boundary rather than silent corruption of parquet output.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The observed JSON/parquet difference matches the documented contract of the flag: `extra-fields` is explicitly scoped to JSON output. Because parquet support never promised to carry those metadata columns, their absence is not a correctness bug in the current design.

### Lesson Learned

Before treating JSON/parquet asymmetry as data loss, confirm whether the flag or feature is documented as format-specific. Here the decisive clue was the explicit "append to output jsons" wording in both the flag registration and README.
