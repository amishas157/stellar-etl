# H002: Newly added signer rows are mislabeled as `UPDATED`

**Date**: 2026-04-10
**Subsystem**: utilities / data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an existing account adds a new signer, the newly introduced `account_signers` row should be exported with `ledger_entry_change = CREATED` and `deleted = false`. The exporter should distinguish a first appearance of a signer from a modification of a signer that already existed.

## Mechanism

`TransformSigners()` does not diff for post-only signers at all. Every signer present in `Post` is emitted in the first pass with the parent account's `LedgerEntryUpdated` change type, and the diff branch only adds synthetic removals for pre-only signers. That means newly added signers are serialized as ordinary updates even though there is no pre-state signer row to update.

## Trigger

Process an `AccountEntry` `LedgerEntryUpdated` change where `Pre` lacks signer `S` and `Post` includes signer `S` (for example a `SetOptions` operation that adds a new signer to an existing account). The correct signer row should be a creation, but the current output marks it as an update.

## Target Code

- `internal/transform/account_signer.go:35-52` — emits all post-state signers with `LedgerEntryChange: uint32(changeType)`
- `internal/transform/account_signer.go:57-86` — update-specific diff only handles `Pre`-only signers and never upgrades `Post`-only signers to `CREATED`

## Evidence

For update changes, `changeType` is always `LedgerEntryUpdated`, so any signer found only in `Post` still leaves the first loop with `ledger_entry_change = 1`. There is no complementary branch that checks `if _, existed := preSigners[signer]; !existed` and rewrites the row to `CREATED`.

## Anti-Evidence

Some downstream loaders may treat `CREATED` and `UPDATED` identically as upserts, which would reduce operational impact. But the export still advertises the wrong row-level change kind, and that corruption matters for consumers that analyze signer lifecycle events or count newly added signers per ledger.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformSigners` in `internal/transform/account_signer.go` processes account ledger changes and emits per-signer rows. For `LedgerEntryUpdated` changes, the first loop (lines 35-52) emits every post-state signer with `LedgerEntryChange = 1` (UPDATED). The second block (lines 57-86) diffs Pre vs Post to synthesize REMOVED rows for deleted signers, but contains no symmetric logic to relabel post-only signers as CREATED (`0`). The asymmetry is confirmed by the test file, which only exercises the deletion scenario — no test adds a signer to an existing account during an update.

### Code Paths Examined

- `internal/transform/account_signer.go:14-52` — `TransformSigners` first loop: iterates `accountEntry.SignerSummary()` (Post state) and assigns `LedgerEntryChange: uint32(changeType)` uniformly to all signers, regardless of whether they existed in Pre
- `internal/transform/account_signer.go:57-86` — diff block: iterates `preAccountEntry.SignerSummary()` and checks `if _, stillExists := postSigners[signer]; stillExists` to skip retained signers, only emitting REMOVED rows for pre-only signers. No complementary post-only check exists.
- `internal/utils/main.go:854-865` — `ExtractEntryFromChange`: returns the account-level `changeType` directly; the function works correctly but provides no per-signer granularity
- `internal/transform/account_signer_test.go:105-119` — test input for UPDATED: Pre has 2 signers + master, Post removes signer B. No test case adds a new signer to Post that wasn't in Pre.

### Findings

The bug is a confirmed asymmetry in `TransformSigners`. The code correctly synthesizes REMOVED rows for signers deleted during an account update (code comment at lines 54-56 explicitly documents this intent: "we must diff Pre vs Post to surface them explicitly"). However, it does not perform the symmetric operation for newly added signers: checking which post-state signers are absent from pre-state and relabeling them as CREATED.

Concretely, when a `SetOptions` operation adds signer S to an existing account:
- The account change type is `LedgerEntryUpdated` (1)
- Signer S appears in Post but not in Pre
- The first loop emits S with `LedgerEntryChange = 1` (UPDATED)
- The diff block only processes pre-only signers, never touching S
- Result: S is exported with `ledger_entry_change = 1` instead of `0`

This affects any downstream consumer that distinguishes CREATED from UPDATED for signer lifecycle analysis (e.g., counting newly added signers per ledger, auditing when signers first appeared on accounts).

### PoC Guidance

- **Test file**: `internal/transform/account_signer_test.go`
- **Setup**: Construct an `ingest.Change` with `ChangeType = LedgerEntryUpdated` where Pre has signer A + master, and Post has signer A + signer B + master (signer B is newly added). Use the existing `makeTestLedgerEntry()` helper, modifying Post to add a third signer not present in Pre.
- **Steps**: Call `TransformSigners(change, header)` and find the output row for signer B.
- **Assertion**: Assert that signer B's `LedgerEntryChange` equals `uint32(xdr.LedgerEntryChangeTypeLedgerEntryCreated)` (0). The current code will produce `uint32(xdr.LedgerEntryChangeTypeLedgerEntryUpdated)` (1), demonstrating the mislabeling.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestAddedSignerMislabeledAsUpdated"
**Test Language**: Go

### Demonstration

The test constructs an account update where Pre has signer A + master and Post has signer A + signer B (newly added) + master. After calling `TransformSigners`, the output row for signer B has `LedgerEntryChange = 1` (UPDATED) instead of the correct `LedgerEntryChange = 0` (CREATED). This confirms that newly added signers are mislabeled as updated because the diff logic only handles removed signers but not added ones.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestAddedSignerMislabeledAsUpdated demonstrates that when an account update
// adds a new signer (present in Post but absent from Pre), the output row for
// that signer has LedgerEntryChange = 1 (UPDATED) instead of the correct
// LedgerEntryChange = 0 (CREATED).
func TestAddedSignerMislabeledAsUpdated(t *testing.T) {
	sponsor, _ := xdr.AddressToAccountId("GBADGWKHSUFOC4C7E3KXKINZSRX5KPHUWHH67UGJU77LEORGVLQ3BN3B")

	// Pre: signer A (Ed25519{4,5,6}, weight=10) + master key — no signer B
	preLedgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 30705278,
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeAccount,
			Account: &xdr.AccountEntry{
				AccountId:     testAccount1ID,
				Balance:       10959979,
				SeqNum:        117801117454198833,
				NumSubEntries: 141,
				InflationDest: &testAccount2ID,
				Flags:         4,
				HomeDomain:    "examplehome.com",
				Thresholds:    xdr.Thresholds([4]byte{2, 1, 3, 5}),
				Ext: xdr.AccountEntryExt{
					V: 1,
					V1: &xdr.AccountEntryExtensionV1{
						Liabilities: xdr.Liabilities{
							Buying:  1000,
							Selling: 1500,
						},
						Ext: xdr.AccountEntryExtensionV1Ext{
							V: 2,
							V2: &xdr.AccountEntryExtensionV2{
								SignerSponsoringIDs: []xdr.SponsorshipDescriptor{
									&sponsor,
								},
							},
						},
					},
				},
				Signers: []xdr.Signer{
					{
						Key: xdr.SignerKey{
							Type:    xdr.SignerKeyTypeSignerKeyTypeEd25519,
							Ed25519: &xdr.Uint256{4, 5, 6},
						},
						Weight: 10,
					},
				},
			},
		},
	}

	// Post: signer A + signer B (Ed25519{10,11,12}, weight=20) — B is newly added
	postLedgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 30705279,
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeAccount,
			Account: &xdr.AccountEntry{
				AccountId:     testAccount1ID,
				Balance:       10959979,
				SeqNum:        117801117454198833,
				NumSubEntries: 142,
				InflationDest: &testAccount2ID,
				Flags:         4,
				HomeDomain:    "examplehome.com",
				Thresholds:    xdr.Thresholds([4]byte{2, 1, 3, 5}),
				Ext: xdr.AccountEntryExt{
					V: 1,
					V1: &xdr.AccountEntryExtensionV1{
						Liabilities: xdr.Liabilities{
							Buying:  1000,
							Selling: 1500,
						},
						Ext: xdr.AccountEntryExtensionV1Ext{
							V: 2,
							V2: &xdr.AccountEntryExtensionV2{
								SignerSponsoringIDs: []xdr.SponsorshipDescriptor{
									&sponsor,
									nil,
								},
							},
						},
					},
				},
				Signers: []xdr.Signer{
					{
						Key: xdr.SignerKey{
							Type:    xdr.SignerKeyTypeSignerKeyTypeEd25519,
							Ed25519: &xdr.Uint256{4, 5, 6},
						},
						Weight: 10,
					},
					{
						Key: xdr.SignerKey{
							Type:    xdr.SignerKeyTypeSignerKeyTypeEd25519,
							Ed25519: &xdr.Uint256{10, 11, 12},
						},
						Weight: 20,
					},
				},
			},
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryUpdated,
		Type:       xdr.LedgerEntryTypeAccount,
		Pre:        &preLedgerEntry,
		Post:       &postLedgerEntry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	signers, err := TransformSigners(change, header)
	if err != nil {
		t.Fatalf("TransformSigners returned unexpected error: %v", err)
	}

	// Find signer B in the output. Signer B's address is derived from Ed25519{10,11,12}.
	// We identify it as the signer with weight=20 that is NOT the master key and NOT signer A.
	var signerBFound bool
	for _, s := range signers {
		if s.Weight == 20 && !s.Deleted {
			signerBFound = true
			// Signer B was newly added (absent in Pre, present in Post).
			// Correct behavior: LedgerEntryChange should be CREATED (0).
			// Actual behavior:  LedgerEntryChange is UPDATED (1).
			if s.LedgerEntryChange == uint32(xdr.LedgerEntryChangeTypeLedgerEntryUpdated) {
				t.Errorf("BUG CONFIRMED: newly added signer B has LedgerEntryChange=%d (UPDATED), "+
					"expected %d (CREATED). Added signers are mislabeled as updated.",
					s.LedgerEntryChange,
					uint32(xdr.LedgerEntryChangeTypeLedgerEntryCreated))
			}
			break
		}
	}

	if !signerBFound {
		t.Fatal("signer B (weight=20) not found in output")
	}
}
```

### Test Output

```
=== RUN   TestAddedSignerMislabeledAsUpdated
    data_integrity_poc_test.go:144: BUG CONFIRMED: newly added signer B has LedgerEntryChange=1 (UPDATED), expected 0 (CREATED). Added signers are mislabeled as updated.
--- FAIL: TestAddedSignerMislabeledAsUpdated (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.786s
```
