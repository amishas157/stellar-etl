# H044: Missing account sequence-extension fields should stay null, not `0`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Suspected nullability loss
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If an account lacks the V3 sequence-extension fields, the export should either preserve that absence as null or otherwise follow the repository's established account-output contract consistently.

## Mechanism

`TransformAccount()` wraps `accountEntry.SeqLedger()` and `SeqTime()` with `zero.IntFrom(...)`, which initially looked like it might fabricate `0` for accounts that do not have those extension fields populated.

## Trigger

Export account ledger changes for pre-extension or otherwise non-V3 account entries.

## Target Code

- `internal/transform/account.go:54-55, 91-93` - reads `SeqLedger()` / `SeqTime()` and serializes them through `zero.IntFrom`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/account_entry.go:89-112` - SDK helpers return `0` when V3 extensions are absent
- `testdata/changes/accounts_trustlines.golden:2` - golden output already asserts `sequence_ledger:0` and `sequence_time:0`

## Evidence

The SDK helpers do default missing V3 extensions to zero (`account_entry.go:89-112`), and `TransformAccount()` forwards those values into the output (`internal/transform/account.go:54-55, 91-93`). That made the zero-fabrication theory plausible on first read.

## Anti-Evidence

The repository's checked-in golden output already contains multiple account rows with `"sequence_ledger":0` and `"sequence_time":0` (`testdata/changes/accounts_trustlines.golden:2`, `8`, `11`). That fixture is strong evidence that zero is the intended export contract for absent sequence-extension fields in this codebase.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Whatever the abstract nullability argument might be, this repository explicitly treats missing sequence-extension values as zero in its golden outputs. That makes the current behavior intentional for this codebase rather than a silent corruption regression.

### Lesson Learned

Golden outputs are decisive for output-contract questions. If a suspicious representation is already asserted in checked-in fixtures, treat it as design intent unless there is contrary documentation inside the repository.
