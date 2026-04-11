# H003: `trustline_flags_updated` effects export a non-canonical maintain-liabilities key

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `SetTrustLineFlags` operation toggles the "authorized to maintain liabilities" bit, the resulting `trustline_flags_updated` effect should export that boolean under the same canonical key shape used by the rest of the effect payload, i.e. `authorized_to_maintain_liabilites_flag` (matching the legacy Horizon field spelling and the neighboring `authorized_flag` / `clawback_enabled_flag` keys).

## Mechanism

`setTrustLineFlagDetails()` emits the maintain-liabilities bit under `authorized_to_maintain_liabilites`, dropping the `_flag` suffix that the same effect family uses for the other trustline booleans. Downstream consumers looking for the standard effect key never see the field, so these rows silently lose that flag transition even though the exported JSON still looks structurally valid.

## Trigger

Export any ledger containing a `SetTrustLineFlags` operation that sets or clears `AuthorizedToMaintainLiabilities`. The emitted `history_effects.details` object will contain `authorized_to_maintain_liabilites: true|false` instead of the expected `authorized_to_maintain_liabilites_flag`.

## Target Code

- `internal/transform/effects.go:1127-1135` — trustline flag helper emits `authorized_flag`, `authorized_to_maintain_liabilites`, and `clawback_enabled_flag`
- `testdata/effects/10_ledgers_effects.golden:4` — checked-in golden output already contains `authorized_to_maintain_liabilites`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/protocols/horizon/effects/main.go:574-580` — upstream Horizon effect model uses `authorized_to_maintain_liabilites_flag`

## Evidence

Inside the same helper, the other trustline booleans carry `_flag` suffixes, so the maintain-liabilities field is the lone outlier. The golden fixtures confirm that production output already serializes the outlier key, which means the mismatch is user-visible today rather than a latent code-only typo.

## Anti-Evidence

The legacy Horizon spelling itself preserves the `liabilites` typo, so the spelling alone is not the core issue here. The concrete export bug is that the ETL drops the `_flag` suffix entirely, making its key shape inconsistent with both sibling fields and the legacy Horizon contract.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete live effects export path: `TransformEffect()` → operation-type dispatch → `setTrustLineFlagsEffects()` / `allowTrustEffects()` → `addTrustLineFlagsEffect()` → `setTrustLineFlagDetails()`. At line 1132 of `effects.go`, the maintain-liabilities flag is emitted under key `"authorized_to_maintain_liabilites"` while its siblings use `"authorized_flag"` and `"clawback_enabled_flag"`. The upstream Horizon SDK's `TrustlineFlagsUpdated` struct uses the JSON tag `"authorized_to_maintain_liabilites_flag"` (with `_flag` suffix). Golden test fixtures confirm the wrong key is already present in production output.

### Code Paths Examined

- `internal/transform/effects.go:23` — `TransformEffect()` is the public entry point, called by the effects export command
- `internal/transform/effects.go:1094-1098` — `setTrustLineFlagsEffects()` calls `addTrustLineFlagsEffect()` with the operation's set/clear flags
- `internal/transform/effects.go:714,723,728` — `allowTrustEffects()` also calls `addTrustLineFlagsEffect()`, so the bug affects both `SetTrustLineFlags` and `AllowTrust` operations
- `internal/transform/effects.go:1101-1125` — `addTrustLineFlagsEffect()` builds details map and delegates to `setTrustLineFlagDetails()`
- `internal/transform/effects.go:1127-1137` — `setTrustLineFlagDetails()` emits three keys: `authorized_flag` (correct), `authorized_to_maintain_liabilites` (MISSING `_flag`), `clawback_enabled_flag` (correct)
- `go-stellar-sdk/protocols/horizon/effects/main.go:579` — Horizon SDK uses `authorized_to_maintain_liabilites_flag` as the canonical JSON key
- `testdata/effects/10_ledgers_effects.golden:4,8,12,...` — 20+ golden entries confirm the wrong key `authorized_to_maintain_liabilites` is emitted in production

### Findings

The bug is confirmed and actively producing incorrect output. The key `"authorized_to_maintain_liabilites"` at line 1132 is missing the `_flag` suffix that its two siblings have and that the upstream Horizon SDK expects. This affects two separate call sites: `SetTrustLineFlags` operations (line 1097) and legacy `AllowTrust` operations (lines 714, 723, 728). Any downstream system querying for the canonical key `authorized_to_maintain_liabilites_flag` will get no result, silently losing the maintain-liabilities flag transition data. The previously investigated fail/014 concerned a typo in the UNUSED `transactionOperationWrapper.Details()` path in `operation.go` — this is a distinct bug in the LIVE `effects.go` path.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go` — append a new test case
- **Setup**: Construct a `SetTrustLineFlags` operation with `AuthorizedToMaintainLiabilities` set, wrap in a minimal `LedgerTransaction`
- **Steps**: Call `TransformEffect()` and find the `trustline_flags_updated` effect in the output
- **Assertion**: Assert that `details["authorized_to_maintain_liabilites_flag"]` is present (it will be absent, confirming the bug). Also assert that `details["authorized_to_maintain_liabilites"]` is what's actually emitted (demonstrating the inconsistency).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestPoCTrustlineFlagsEffectMaintainLiabilitiesKey"
**Test Language**: Go

### Demonstration

The test constructs a `SetTrustLineFlags` operation that sets `AuthorizedToMaintainLiabilities`, runs it through the live `effects()` code path, and inspects the emitted effect details map. It confirms that the canonical key `authorized_to_maintain_liabilites_flag` is ABSENT while the non-canonical key `authorized_to_maintain_liabilites` (missing `_flag` suffix) IS present with value `true`. This proves the key naming inconsistency with both sibling keys (`authorized_flag`, `clawback_enabled_flag`) and the upstream Horizon SDK.

### Test Body

```go
package transform

import (
	"fmt"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stretchr/testify/assert"
)

// TestPoCTrustlineFlagsEffectMaintainLiabilitiesKey demonstrates that
// setTrustLineFlagDetails emits the maintain-liabilities flag under the key
// "authorized_to_maintain_liabilites" instead of the canonical
// "authorized_to_maintain_liabilites_flag" used by the Horizon SDK and by the
// sibling keys "authorized_flag" and "clawback_enabled_flag" in the same helper.
func TestPoCTrustlineFlagsEffectMaintainLiabilitiesKey(t *testing.T) {
	tt := assert.New(t)

	// Construct a SetTrustLineFlags operation that sets AuthorizedToMaintainLiabilities
	aid := xdr.MustAddress("GDRW375MAYR46ODGF2WGANQC2RRZL7O246DYHHCGWTV2RE7IHE2QUQLD")
	source := aid.ToMuxedAccount()
	trustor := xdr.MustAddress("GAUJETIZVEP2NRYLUESJ3LS66NVCEGMON4UDCBCSBEVPIID773P2W6AY")
	setFlags := xdr.Uint32(xdr.TrustLineFlagsAuthorizedToMaintainLiabilitiesFlag)

	op := xdr.Operation{
		SourceAccount: &source,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeSetTrustLineFlags,
			SetTrustLineFlagsOp: &xdr.SetTrustLineFlagsOp{
				Trustor:    trustor,
				Asset:      xdr.MustNewCreditAsset("USD", "GDRW375MAYR46ODGF2WGANQC2RRZL7O246DYHHCGWTV2RE7IHE2QUQLD"),
				SetFlags:   setFlags,
				ClearFlags: 0,
			},
		},
	}

	operation := transactionOperationWrapper{
		index: 0,
		transaction: ingest.LedgerTransaction{
			UnsafeMeta: xdr.TransactionMeta{
				V:  2,
				V2: &xdr.TransactionMetaV2{},
			},
		},
		operation:      op,
		ledgerSequence: 1,
		ledgerClosed:   genericCloseTime.UTC(),
	}

	effects, err := operation.effects()
	tt.NoError(err)
	tt.NotEmpty(effects, "expected at least one effect")

	// Find the trustline_flags_updated effect emitted by addTrustLineFlagsEffect
	var found bool
	for _, eff := range effects {
		if eff.Type != int32(EffectTrustlineFlagsUpdated) {
			continue
		}
		details := eff.Details

		// The canonical key (matching Horizon SDK and sibling keys) should be
		// "authorized_to_maintain_liabilites_flag", but the code emits
		// "authorized_to_maintain_liabilites" (missing _flag suffix).

		// BUG: the canonical key is ABSENT
		_, hasCanonical := details["authorized_to_maintain_liabilites_flag"]
		if hasCanonical {
			// If this branch is reached, the bug has been fixed
			t.Log("canonical key authorized_to_maintain_liabilites_flag IS present — bug may be fixed")
			continue
		}

		// BUG CONFIRMED: the non-canonical key IS present
		val, hasNonCanonical := details["authorized_to_maintain_liabilites"]
		if !hasNonCanonical {
			continue
		}

		found = true
		t.Logf("BUG CONFIRMED: effect details contain non-canonical key 'authorized_to_maintain_liabilites' = %v", val)
		t.Logf("Expected canonical key 'authorized_to_maintain_liabilites_flag' is ABSENT")
		t.Logf("Sibling keys use _flag suffix: 'authorized_flag', 'clawback_enabled_flag'")
		t.Logf("Horizon SDK uses: authorized_to_maintain_liabilites_flag")
		t.Logf("Full effect details: %v", details)

		// Assert the inconsistency: non-canonical key exists but canonical does not
		tt.True(hasNonCanonical, "non-canonical key should be present (confirming bug)")
		tt.False(hasCanonical, "canonical key should be absent (confirming bug)")
		tt.Equal(true, val, "maintain liabilities flag value should be true")

		// Verify sibling keys DO use _flag suffix for contrast
		_, hasAuthorizedFlag := details["authorized_flag"]
		_, hasClawbackFlag := details["clawback_enabled_flag"]
		// At least one sibling should NOT be present here since we only set
		// AuthorizedToMaintainLiabilities. But if they were, they'd have _flag.
		t.Logf("authorized_flag present: %v, clawback_enabled_flag present: %v",
			hasAuthorizedFlag, hasClawbackFlag)

		// The key naming is inconsistent: maintain_liabilites is missing _flag
		// while authorized_flag and clawback_enabled_flag both have it.
		break
	}

	if !found {
		// Print all effects for debugging
		for i, eff := range effects {
			t.Logf("Effect[%d]: type=%d (%s), details=%v",
				i, eff.Type, eff.TypeString, eff.Details)
		}
		t.Fatal("did not find trustline_flags_updated effect with maintain-liabilities key")
	}

	// Final assertion: verify the bug by showing the key mismatch
	// The code at effects.go:1132 emits "authorized_to_maintain_liabilites"
	// but should emit "authorized_to_maintain_liabilites_flag" to match
	// its siblings and the Horizon SDK.
	fmt.Println("=== POC RESULT: Bug confirmed ===")
	fmt.Println("Key emitted:   authorized_to_maintain_liabilites")
	fmt.Println("Key expected:  authorized_to_maintain_liabilites_flag")
	fmt.Println("Sibling keys:  authorized_flag, clawback_enabled_flag (both have _flag suffix)")
}
```

### Test Output

```
=== RUN   TestPoCTrustlineFlagsEffectMaintainLiabilitiesKey
    data_integrity_poc_test.go:83: BUG CONFIRMED: effect details contain non-canonical key 'authorized_to_maintain_liabilites' = true
    data_integrity_poc_test.go:84: Expected canonical key 'authorized_to_maintain_liabilites_flag' is ABSENT
    data_integrity_poc_test.go:85: Sibling keys use _flag suffix: 'authorized_flag', 'clawback_enabled_flag'
    data_integrity_poc_test.go:86: Horizon SDK uses: authorized_to_maintain_liabilites_flag
    data_integrity_poc_test.go:87: Full effect details: map[asset_code:USD asset_issuer:GDRW375MAYR46ODGF2WGANQC2RRZL7O246DYHHCGWTV2RE7IHE2QUQLD asset_type:credit_alphanum4 authorized_to_maintain_liabilites:true trustor:GAUJETIZVEP2NRYLUESJ3LS66NVCEGMON4UDCBCSBEVPIID773P2W6AY]
    data_integrity_poc_test.go:99: authorized_flag present: false, clawback_enabled_flag present: false
=== POC RESULT: Bug confirmed ===
Key emitted:   authorized_to_maintain_liabilites
Key expected:  authorized_to_maintain_liabilites_flag
Sibling keys:  authorized_flag, clawback_enabled_flag (both have _flag suffix)
--- PASS: TestPoCTrustlineFlagsEffectMaintainLiabilitiesKey (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.855s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC actually exercise the claimed issue?** Yes. The test reaches the live `TransformEffect()`/`effects()` trustline-flags path and demonstrates that `internal/transform/effects.go` emits `authorized_to_maintain_liabilites` without the `_flag` suffix.
2. **Are the preconditions realistic?** Yes. A real `SetTrustLineFlags` or `AllowTrust` operation that toggles `AuthorizedToMaintainLiabilities` will hit this code path.
3. **Is the behavior a bug or by design?** Not established as a local bug. `stellar-etl` matches the upstream `stellar/go` effect processors and Horizon ingest processors exactly: they emit the same `authorized_to_maintain_liabilites` key and have tests asserting that shape. The PoC only proves a mismatch with the `protocols/horizon` struct tag, not that the ETL diverges from the production effect generator it mirrors.
4. **Does the impact match the claimed severity?** No confirmed local data-corruption impact. The finding hinges on treating the SDK protocol struct as the canonical export contract, but the live upstream processor path uses the same key as the ETL.
5. **Is the finding in scope?** The only concrete inconsistency left is between upstream `stellar/go` components (`protocols/horizon` vs processor/resource-adapter pipeline). The task explicitly excludes upstream SDK bugs.
6. **Is the test itself correct?** The test is mechanically correct but not sufficient for confirmation. It asserts absence of a reviewer-chosen key rather than validating the ETL against a proven local contract, and it passes even when the ETL is faithfully mirroring upstream behavior.
7. **Can the results be explained WITHOUT the claimed issue?** Yes. The observed output is fully explained by compatibility with the upstream effect processor implementation that `stellar-etl` was derived from.
8. **Is this finding novel?** Novel locally, but not confirmable as an ETL bug.

### Rejection Reason

The PoC demonstrates current key shape, but not a local defect. `stellar-etl` emits the same `authorized_to_maintain_liabilites` field name as the upstream `stellar/go` effects processors and Horizon ingest pipeline, so the claim that `authorized_to_maintain_liabilites_flag` is the canonical export key is not established for this repository. What remains is an upstream inconsistency between processor output and the `protocols/horizon` model, which is out of scope here.

### Failed Checks

3, 4, 5, 6, 7
