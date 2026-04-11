# 006: Effects limit counts transactions, not emitted effect rows

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_effects` documents `--limit` as a maximum number of exported effect rows, but the command applies that value to `input.GetTransactions()`, which stops after N transactions instead. The command then expands each returned transaction into all of its effects and writes every row, so a single limited transaction can still emit multiple effect records.

## Root Cause

The limit is enforced before effect expansion. `GetTransactions()` counts `len(txSlice)` against the requested limit, while `export_effects` later calls `TransformEffect()` and writes the full returned slice without any second row-level cap.

## Reproduction

During normal operation, this manifests whenever the first transaction returned for the requested range is a successful transaction that produces multiple effects, such as `create_account`. Running `export_effects --limit 1` then reads one transaction but still writes every effect row derived from that transaction.

## Affected Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — advertises `--limit` as the maximum number of `effects` to export
- `cmd/export_effects.go:21-57` — passes `limit` into `input.GetTransactions()` and writes every row returned by `TransformEffect()`
- `internal/input/transactions.go:GetTransactions:23-70` — enforces the limit on transaction count
- `internal/transform/effects.go:23-50` — expands one transaction into a slice of effect rows

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestEffectsLimitCountsTransactionsNotEffects`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestEffectsLimitCountsTransactionsNotEffects demonstrates the documented limit mismatch:
// a single transaction returned under a limit of 1 can still expand into multiple effect rows.
func TestEffectsLimitCountsTransactionsNotEffects(t *testing.T) {
	const limit = 1

	createAccountEnvelopeXDR := "AAAAAGL8HQvQkbK2HA3WVjRrKmjX00fG8sLI7m0ERwJW/AX3AAAAZAAAAAAAAAAaAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAoZftFP3p4ifbTm6hQdieotu3Zw9E05GtoSh5MBytEpQAAAACVAvkAAAAAAAAAAABVvwF9wAAAEDHU95E9wxgETD8TqxUrkgC0/7XHyNDts6Q5huRHfDRyRcoHdv7aMp/sPvC3RPkXjOMjgbKJUX7SgExUeYB5f8F"
	createAccountResultXDR := "AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAABAAAAAAAAAAA="
	createAccountFeeChangesXDR := "AAAAAgAAAAMAAAA3AAAAAAAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wsatlj11nHQAAAAAAAAABkAAAAAAAAAAQAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAEAAAA5AAAAAAAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wsatlj11nFsAAAAAAAAABkAAAAAAAAAAQAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAA=="
	createAccountHash := "0e5bd332291e3098e49886df2cdb9b5369a5f9e0a9973f0d9e1a9489c6581ba2"

	creator := xdr.MustAddress("GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H")
	created := xdr.MustAddress("GCQZP3IU7XU6EJ63JZXKCQOYT2RNXN3HB5CNHENNUEUHSMA4VUJJJSEN")
	sponsor := xdr.MustAddress("GAHK7EEG2WWHVKDNT4CEQFZGKF2LGDSW2IVM4S5DP42RBW3K6BTODB4A")
	sponsor2 := xdr.MustAddress("GACMZD5VJXTRLKVET72CETCYKELPNCOTTBDC6DHFEUPLG5DHEK534JQX")

	createAccountMeta := &xdr.TransactionMeta{
		V: 1,
		V1: &xdr.TransactionMetaV1{
			TxChanges: xdr.LedgerEntryChanges{
				{
					Type: 3,
					State: &xdr.LedgerEntry{
						LastModifiedLedgerSeq: 0x39,
						Data: xdr.LedgerEntryData{
							Type: 0,
							Account: &xdr.AccountEntry{
								AccountId:     creator,
								Balance:       800152377009533292,
								SeqNum:        25,
								InflationDest: &creator,
								Thresholds:    xdr.Thresholds{0x1, 0x0, 0x0, 0x0},
							},
						},
					},
				},
				{
					Type: 1,
					Updated: &xdr.LedgerEntry{
						LastModifiedLedgerSeq: 0x39,
						Data: xdr.LedgerEntryData{
							Type: 0,
							Account: &xdr.AccountEntry{
								AccountId:     creator,
								Balance:       800152367009533292,
								SeqNum:        26,
								InflationDest: &creator,
								Thresholds:    xdr.Thresholds{0x1, 0x0, 0x0, 0x0},
							},
						},
						Ext: xdr.LedgerEntryExt{
							V: 1,
							V1: &xdr.LedgerEntryExtensionV1{
								SponsoringId: &sponsor2,
							},
						},
					},
				},
			},
			Operations: []xdr.OperationMeta{
				{
					Changes: xdr.LedgerEntryChanges{
						{
							Type: 3,
							State: &xdr.LedgerEntry{
								LastModifiedLedgerSeq: 0x39,
								Data: xdr.LedgerEntryData{
									Type: xdr.LedgerEntryTypeAccount,
									Account: &xdr.AccountEntry{
										AccountId:     creator,
										Balance:       800152367009533292,
										SeqNum:        26,
										InflationDest: &creator,
										Thresholds:    xdr.Thresholds{0x1, 0x0, 0x0, 0x0},
									},
								},
								Ext: xdr.LedgerEntryExt{
									V: 1,
									V1: &xdr.LedgerEntryExtensionV1{
										SponsoringId: &sponsor2,
									},
								},
							},
						},
						{
							Type: 1,
							Updated: &xdr.LedgerEntry{
								LastModifiedLedgerSeq: 0x39,
								Data: xdr.LedgerEntryData{
									Type: xdr.LedgerEntryTypeAccount,
									Account: &xdr.AccountEntry{
										AccountId:     creator,
										Balance:       800152357009533292,
										SeqNum:        26,
										InflationDest: &creator,
										Thresholds:    xdr.Thresholds{0x1, 0x0, 0x0, 0x0},
									},
								},
								Ext: xdr.LedgerEntryExt{
									V: 1,
									V1: &xdr.LedgerEntryExtensionV1{
										SponsoringId: &sponsor,
									},
								},
							},
						},
						{
							Type: 0,
							Created: &xdr.LedgerEntry{
								LastModifiedLedgerSeq: 0x39,
								Data: xdr.LedgerEntryData{
									Type: xdr.LedgerEntryTypeAccount,
									Account: &xdr.AccountEntry{
										AccountId:  created,
										Balance:    10000000000,
										SeqNum:     244813135872,
										Thresholds: xdr.Thresholds{0x1, 0x0, 0x0, 0x0},
									},
								},
								Ext: xdr.LedgerEntryExt{
									V: 1,
									V1: &xdr.LedgerEntryExtensionV1{
										SponsoringId: &sponsor,
									},
								},
							},
						},
					},
				},
			},
		},
	}

	createAccountMetaB64, err := xdr.MarshalBase64(createAccountMeta)
	if err != nil {
		t.Fatalf("failed to marshal createAccountMeta: %v", err)
	}

	transaction := BuildLedgerTransaction(
		t,
		TestTransaction{
			Index:         1,
			EnvelopeXDR:   createAccountEnvelopeXDR,
			ResultXDR:     createAccountResultXDR,
			MetaXDR:       createAccountMetaB64,
			FeeChangesXDR: createAccountFeeChangesXDR,
			Hash:          createAccountHash,
		},
	)

	ledgerCloseMeta := makeLedgerCloseMeta()
	ledgerSeq := uint32(57)
	effects, err := TransformEffect(transaction, ledgerSeq, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformEffect failed: %v", err)
	}

	if len(effects) <= limit {
		t.Fatalf("expected more than %d exported effect row from %d limited transaction, got %d", limit, limit, len(effects))
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_effects --limit 1` should emit at most one effect row, matching the CLI flag text.
- **Actual**: the command limits itself to one transaction and then writes every effect generated by that transaction, so a multi-effect transaction exceeds the requested row cap.

## Adversarial Review

1. Exercises claimed bug: YES — the fixed PoC proves a realistic single transaction can expand into multiple effect rows, and the source trace shows `export_effects` applies `--limit` before that expansion.
2. Realistic preconditions: YES — `create_account`, path payments, and other successful operations routinely emit multiple effects.
3. Bug vs by-design: BUG — both the shared flag text and the command comment describe `--limit` as an effect-row cap, not a transaction cap.
4. Final severity: Medium — this is an operational correctness issue that returns extra rows without corrupting row contents.
5. In scope: YES — the command silently produces output that violates its documented export bound.
6. Test correctness: CORRECT — the test uses production `TransformEffect()` on a realistic transaction fixture; the transaction-limit side is independently verified in `GetTransactions()`.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Track emitted effect rows in `export_effects` (or introduce an effect-aware reader) and stop writing once the requested row limit is reached, matching the CLI contract.
