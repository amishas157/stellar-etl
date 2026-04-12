# H002: Oversized token-transfer amounts overflow to +Inf and ExportEntry writes `{}`

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_token_transfer` should either preserve the event row with its exact `amount_raw` and a sane derived `amount`, or fail the row explicitly. A legitimate large token transfer must not degrade into an empty JSON object, because downstream consumers lose the transaction hash, contract id, participants, and raw amount together.

## Mechanism

`transformEvents()` parses the token-transfer amount string with `strconv.ParseFloat(..., 64)` and ignores overflow errors. When a large Soroban token amount string exceeds `float64` range, `Amount` becomes `+Inf`; then `ExportEntry()` logs the `json.Marshal(entry)` failure, continues with an empty byte slice, decodes into an empty map, and finally writes `{}` instead of returning an error. The command therefore reports success-shaped progress while silently replacing a real transfer event with an empty row.

## Trigger

1. Process a token-transfer event whose amount string is larger than `float64` can represent, for example a decimal string with hundreds of digits.
2. Run `export_token_transfer`.
3. The output row for that event becomes `{}` (plus any extra fields), because `Amount=+Inf` makes the first marshal fail and `ExportEntry()` falls through to an empty map.

## Target Code

- `internal/transform/token_transfer.go:47-73` — amount strings are parsed into `float64` with ignored errors
- `internal/transform/token_transfer.go:108-126` — the potentially infinite `Amount` is stored on the output row
- `cmd/command_utils.go:55-86` — marshal/unmarshal failures are logged but the function still writes the now-empty map
- `cmd/export_token_transfers.go:46-54` — the command treats the returned byte count as a successful export

## Evidence

The token-transfer path is one of the few exports that starts from an arbitrary decimal string rather than an XDR-bounded `int64`. `strconv.ParseFloat` returns `+Inf` with `ErrRange` on overflow, and this code discards that error in every event arm. `ExportEntry()` then explicitly logs the first marshal error without returning, so the control flow continues into an empty `map[string]interface{}` and emits `{}`.

## Anti-Evidence

Classic SAC/native transfer amounts that stay within ordinary `float64` range will only show the already-known rounding behavior, not full-row erasure. This path needs an unusually large but still legitimate token amount, so it hides behind small-fixture tests and normal mainnet-sized samples.
