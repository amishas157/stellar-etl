# H003: Config-setting Parquet narrows uint64 window samples into signed negatives

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW`, both JSON and Parquet
exports should preserve every unsigned 64-bit sample in
`bucket_list_size_window` / `live_soroban_state_size_window`. A valid on-chain
window sample above `math.MaxInt64` should remain a large positive unsigned
value, not wrap into a negative Parquet element.

## Mechanism

The XDR getter returns `[]Uint64`, and `TransformConfigSetting()` preserves that
as `[]uint64` in JSON. But `ConfigSettingOutputParquet` models both repeated
columns as plain `[]int64`, and `ToParquet()` casts each element with `int64(v)`
before writing it. Any sample above `9223372036854775807` therefore flips sign,
and the repeated Parquet columns have no `UINT_64` annotation that could recover
the original unsigned meaning downstream.

## Trigger

Export config settings for a ledger whose
`CONFIG_SETTING_LIVE_SOROBAN_STATE_SIZE_WINDOW` row contains at least one window
sample above `math.MaxInt64`. The JSON export will preserve the positive
`uint64`, while Parquet will emit the same sample as a negative `int64` element
in `bucket_list_size_window` and `live_soroban_state_size_window`.

## Target Code

- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:61921` — `GetLiveSorobanStateSizeWindow()` returns `[]Uint64`
- `internal/transform/config_setting.go:101-107` — transform builds unsigned `[]uint64` slices from the XDR window values
- `internal/transform/config_setting.go:171-172` — JSON output preserves those unsigned slices
- `internal/transform/schema_parquet.go:362-363` — Parquet schema narrows both repeated columns to plain `[]int64`
- `internal/transform/parquet_converter.go:341-345` — converter casts each sample with `int64(v)`
- `internal/transform/parquet_converter.go:404` — the same narrowed slice is written into `live_soroban_state_size_window`

## Evidence

This is a real type mismatch, not just a naming alias: the source is explicitly
`Uint64`, the JSON schema is explicitly `[]uint64`, and the Parquet schema is
explicitly `[]int64`. Unlike timestamp-backed casts previously rejected as
domain-unreachable, these window samples are generic size counters with no
protocol-level signed bound below the full `uint64` range.

## Anti-Evidence

The trigger requires unusually large governance-configured window samples, so it
may not appear on today's networks. But once such a config value is present,
Parquet has no way to preserve it faithfully because the conversion is signed
and the repeated field lacks unsigned logical typing.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (fail/022 covered the aliasing typo, not the uint64→int64 wrapping; fail/058 covered sequence_time narrowing, a different entity)
**Failed At**: reviewer

### Trace Summary

Traced from `GetLiveSorobanStateSizeWindow()` returning `[]Uint64` through `TransformConfigSetting()` (config_setting.go:101-107) building `[]uint64` slices, into `ToParquet()` (parquet_converter.go:339-345) casting each element to `int64(v)`, and the Parquet schema (schema_parquet.go:362-363) declaring `[]int64` with `repetitiontype=REPEATED` but no `convertedtype=UINT_64`. The code path is real — the developers even acknowledge the risk with a comment at line 343: "This can cause a drop in precision of data."

### Code Paths Examined

- `internal/transform/config_setting.go:101-107` — Builds `[]uint64` from XDR `[]Uint64` window values; confirmed unsigned types preserved in JSON
- `internal/transform/schema.go:620-621` — JSON schema: `BucketListSizeWindow []uint64`, `LiveSorobanStateSizeWindow []uint64`
- `internal/transform/schema_parquet.go:362-363` — Parquet schema: `[]int64` with `repetitiontype=REPEATED` but NO `convertedtype=UINT_64` (unlike scalar uint64 fields in the same struct which do have it)
- `internal/transform/parquet_converter.go:341-345` — `int64(v)` cast on each element; explicit comment acknowledging precision loss risk
- `internal/transform/parquet_converter.go:403-404` — Both Parquet fields assigned from the same `BucketListSizeWindowInt` slice

### Why It Failed

The mechanism is technically correct but the trigger is **domain-unreachable**. The window samples represent snapshots of the total live Soroban state size in bytes. `math.MaxInt64` is approximately 9.2 × 10^18 bytes (9.2 exabytes). No blockchain — including Stellar — can physically store exabytes of state data. The Stellar network's total state is currently measured in gigabytes, many orders of magnitude below the wrapping threshold.

This follows the same pattern as rejected fail/058 (account `sequence_time` uint64→int64 narrowing), which was dismissed because "reaching `math.MaxInt64` requires a far-future chain state not achievable on realistic ledgers." The same reasoning applies here: byte sizes of blockchain state cannot realistically reach the unsigned range that would trigger sign wrapping.

The hypothesis's argument that these are "generic size counters with no protocol-level signed bound" is technically true but ignores the physical constraint: you need actual bytes of stored data to produce these values, and exabytes of blockchain state is not achievable.

### Lesson Learned

Not every `uint64 → int64` cast is a live data-integrity bug. For size/byte-count fields, the physical constraints of blockchain storage bound the domain well below `MaxInt64`. The domain-reachability analysis from fail/058 (timestamps) applies equally to byte-size counters — verify that the domain can realistically produce values in the dangerous range before treating the narrowing as actionable corruption.
