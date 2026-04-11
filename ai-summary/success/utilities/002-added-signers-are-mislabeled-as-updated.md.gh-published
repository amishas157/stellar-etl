# 002: Added signers are mislabeled as updated

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: utilities
**Final review by**: gpt-5.4, high

## Summary

`TransformSigners` emits every signer in an account `UPDATED` change with `ledger_entry_change = UPDATED`, even when a signer appears for the first time in the post-state. As a result, the `account_signers` export silently misclassifies newly added signer rows as updates instead of creations.

## Root Cause

The first loop in `TransformSigners` assigns the parent account change type to every post-state signer row. The update-specific diff that follows only synthesizes `REMOVED` rows for pre-only signers; it never checks for post-only signers and relabels them as `CREATED`.

## Reproduction

Create `internal/transform/data_integrity_poc_test.go` with the PoC below and run `go test ./internal/transform/... -run TestAddedSignerMislabeledAsUpdated -v`. The test constructs a realistic account update where signer B is absent from `Pre` and present in `Post`, then shows that the production transform exports signer B as `UPDATED`.

## Affected Code

- `internal/transform/account_signer.go:35-52` — emits all post-state signers with the parent account `changeType`
- `internal/transform/account_signer.go:57-86` — update-specific diff only emits synthetic `REMOVED` rows for pre-only signers
- `internal/utils/main.go:1051-1115` — `AccountSignersChanged` only invokes signer export when signer membership/weight/sponsorship actually changed
- `cmd/export_ledger_entry_changes.go:148-156` — exports `TransformSigners` output directly into `account_signers`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestAddedSignerMislabeledAsUpdated`
- **Test language**: `go`
- **How to run**: Create the target test file with the test body below, then run `go test ./internal/transform/... -run TestAddedSignerMislabeledAsUpdated -v`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestAddedSignerMislabeledAsUpdated(t *testing.T) {
	preLedgerEntry := makeTestLedgerEntry()
	preLedgerEntry.Data.Account.Signers = preLedgerEntry.Data.Account.Signers[:1]
	preLedgerEntry.Data.Account.Ext.V1.Ext.V2.SignerSponsoringIDs = preLedgerEntry.Data.Account.Ext.V1.Ext.V2.SignerSponsoringIDs[:1]

	postLedgerEntry := makeTestLedgerEntry()
	postLedgerEntry.LastModifiedLedgerSeq = 30705279

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

	for _, signer := range signers {
		if signer.Signer != "GAFAWDAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABNDC" {
			continue
		}

		if signer.Deleted {
			t.Fatalf("newly added signer unexpectedly marked deleted: %+v", signer)
		}

		if want := uint32(xdr.LedgerEntryChangeTypeLedgerEntryCreated); signer.LedgerEntryChange != want {
			t.Fatalf("newly added signer labeled %d, want %d (CREATED): %+v", signer.LedgerEntryChange, want, signer)
		}

		return
	}

	t.Fatal("newly added signer not found in transform output")
}
```

## Expected vs Actual Behavior

- **Expected**: A signer that is absent from `Pre` and present in `Post` should export as a new `account_signers` row with `ledger_entry_change = CREATED` and `deleted = false`.
- **Actual**: The production transform exports that signer with `ledger_entry_change = UPDATED`, even though there is no prior signer row to update.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls `TransformSigners` directly with a real `ingest.Change` using the production update path.
2. Realistic preconditions: YES — an existing account can add a signer via `SetOptions`, producing exactly this pre/post shape.
3. Bug vs by-design: BUG — the exporter already synthesizes per-signer `REMOVED` rows and only runs when signer state changed, so `ledger_entry_change` is intended to describe signer-row lifecycle, not merely mirror the parent account change.
4. Final severity: High — the exported non-financial lifecycle field is wrong and can corrupt downstream signer-creation analytics or audits.
5. In scope: YES — this is a concrete ETL data-correctness defect in production code.
6. Test correctness: CORRECT — the test fails only if a post-only signer row is mislabeled; it does not assert on mocked or tautological state.
7. Alternative explanations: NONE — the wrong value comes from `TransformSigners` assigning the account-level `UPDATED` change type to all post-state signers.
8. Novelty: NOVEL

## Suggested Fix

During `LedgerEntryUpdated` handling, diff `Pre` and `Post` symmetrically: retain `UPDATED` for signers present in both states, emit `REMOVED` for pre-only signers, and relabel post-only signers as `CREATED`.
