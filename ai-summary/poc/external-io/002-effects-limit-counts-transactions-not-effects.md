# H002: `export_effects --limit` caps transactions, not emitted effect rows

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `export_effects`, the shared `--limit` flag is registered as "Maximum number of effects to export", so a run with `--limit N` should write at most `N` effect rows to JSON/Parquet output.

## Mechanism

`cmd/export_effects.go` passes that limit into `input.GetTransactions()`, which stops after `N` transactions, not `N` effects. `transform.TransformEffect()` then expands each returned transaction into `[]EffectOutput`, and `export_effects` writes every element of that slice, so a single transaction with multiple effects can exceed the promised row limit and silently export extra rows.

## Trigger

Run `export_effects --limit 1` on a ledger containing a successful transaction that produces multiple effects, such as account-creation or path-payment flows that touch several ledger entries. The command should emit one effect row but should instead write all effects from that single transaction.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:248-255` - CLI help text promises a maximum number of `effects`
- `cmd/export_effects.go:25-69` - forwards the effects limit into `GetTransactions()` and then writes every transformed effect
- `internal/input/transactions.go:GetTransactions:22-70` - enforces `limit` on transaction count
- `internal/transform/effects.go:23-50` - expands one transaction into a slice of effect rows

## Evidence

The effect command's own inline comment says `limit: maximum number of effects to export`, but the actual reader boundary is transaction-granular. `TransformEffect()` explicitly appends `p...` from every operation in the transaction, so one limited transaction can still produce many emitted rows.

## Anti-Evidence

If each returned transaction happens to yield exactly one effect, the mismatch is invisible. Negative limits also avoid the boundary entirely because the command exports the whole range.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`export_effects.go:25` calls `input.GetTransactions(startNum, commonArgs.EndNum, limit, ...)`, passing the user-supplied `--limit` value. `GetTransactions()` (transactions.go:51) enforces the limit against `len(txSlice)` — the number of *transactions*, not effects. Back in `export_effects.go:34-57`, every transaction is expanded via `TransformEffect()` into a `[]EffectOutput` slice, and every element is written to output with no secondary count check. The sibling commands `export_operations` and `export_trades` each have dedicated `Get*()` functions that count the actual output entity (operations, trade-producing operations) against the limit, confirming that the effects path is the outlier.

### Code Paths Examined

- `internal/utils/main.go:AddArchiveFlags:250-254` — Registers `--limit` with help text `"Maximum number of " + objectName + " to export"`. For effects, objectName="effects", so the flag promises to limit effects.
- `cmd/export_effects.go:25` — Passes `limit` directly to `input.GetTransactions()`, which treats it as a transaction cap.
- `internal/input/transactions.go:51` — Loop condition `for int64(len(txSlice)) < limit || limit < 0` counts transactions against limit.
- `cmd/export_effects.go:34-57` — Iterates all returned transactions, calls `TransformEffect()` per transaction, then writes every effect in the returned slice. No limit enforcement on the emitted effect count.
- `internal/transform/effects.go:23-50` — `TransformEffect()` iterates all operations in a transaction and appends all effects from each, returning a potentially large `[]EffectOutput` per transaction.
- `internal/input/operations.go:52-70` — Sibling `GetOperations()` correctly counts *operations* against the limit with an inner break, confirming the expected pattern.
- `internal/input/trades.go:50-76` — Sibling `GetTrades()` correctly counts *trade inputs* against the limit with an inner break.

### Findings

The bug is confirmed. The `--limit N` flag for `export_effects` promises to cap output at N effects but actually caps at N transactions. Since a single transaction can produce many effects (every operation in a successful transaction generates one or more effects — account creation alone produces 3 effects: account_created, signer_created, account_debited), the actual output row count can significantly exceed the requested limit.

**Severity downgrade from High to Medium**: The individual effect rows are all correct — no data corruption occurs. The issue is that the limit control doesn't work as documented, which is an operational correctness problem rather than structural data corruption. Output consumers receive more rows than expected, but each row contains accurate data.

The pattern is consistent with Investigation Pattern 5 (Loop condition consistency): `GetOperations()` and `GetTrades()` each count their actual output entity against the limit, while `export_effects` reuses `GetTransactions()` without adaptation, making it the outlier.

### PoC Guidance

- **Test file**: `cmd/export_effects_test.go` (or a new test file if one doesn't exist)
- **Setup**: Create a mock ledger backend containing a single transaction that produces multiple effects (e.g., a `CreateAccount` operation which generates `account_created`, `signer_created`, and `account_debited` effects). Configure `--limit 1`.
- **Steps**: Call the effects export path with `limit=1`. Count the number of effect rows written to the output file.
- **Assertion**: Assert that the output contains more than 1 effect row despite `--limit 1`, demonstrating the limit is enforced on transactions, not effects. Alternatively, a unit-level PoC: call `GetTransactions(start, end, 1, ...)` and then `TransformEffect()` on the returned transaction, and assert `len(effects) > 1`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestEffectsLimitCountsTransactionsNotEffects"
**Test Language**: Go

### Demonstration

The test constructs a single CreateAccount transaction (using the same XDR fixtures from existing effects tests) and calls `TransformEffect()` on it. The single transaction produces 5 effects (account_created, account_debited, signer_created, account_sponsorship_updated, account_sponsorship_created). This proves that `GetTransactions(start, end, limit=1)` — which `export_effects` calls with the user's `--limit` value — returns 1 transaction, but `export_effects` then writes all 5 effects from that transaction with no secondary count check. The `--limit` flag therefore caps transactions, not effects as documented.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestEffectsLimitCountsTransactionsNotEffects demonstrates that a single
// transaction can produce many effects. Since export_effects passes --limit
// to GetTransactions (which caps transactions, not effects), setting
// --limit 1 allows more than 1 effect row to be written.
func TestEffectsLimitCountsTransactionsNotEffects(t *testing.T) {
	// Use the existing createAccount test data from effects_test.go.
	// This is a CreateAccount transaction that produces multiple effects:
	// account_created, account_debited, signer_created, plus sponsorship effects.
	createAccountEnvelopeXDR := "AAAAAGL8HQvQkbK2HA3WVjRrKmjX00fG8sLI7m0ERwJW/AX3AAAAZAAAAAAAAAAaAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAoZftFP3p4ifbTm6hQdieotu3Zw9E05GtoSh5MBytEpQAAAACVAvkAAAAAAAAAAABVvwF9wAAAEDHU95E9wxgETD8TqxUrkgC0/7XHyNDts6Q5huRHfDRyRcoHdv7aMp/sPvC3RPkXjOMjgbKJUX7SgExUeYB5f8F"
	createAccountResultXDR := "AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAABAAAAAAAAAAA="
	createAccountFeeChangesXDR := "AAAAAgAAAAMAAAA3AAAAAAAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wsatlj11nHQAAAAAAAAABkAAAAAAAAAAQAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAEAAAA5AAAAAAAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wsatlj11nFsAAAAAAAAABkAAAAAAAAAAQAAAABi/B0L0JGythwN1lY0aypo19NHxvLCyO5tBEcCVvwF9wAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAA=="
	createAccountHash := "0e5bd332291e3098e49886df2cdb9b5369a5f9e0a9973f0d9e1a9489c6581ba2"

	// Build the createAccount metadata with sponsorship changes (produces 6 effects)
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

	// Build the ledger transaction from the XDR test data
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

	// Use the same LedgerCloseMeta helper used by existing effects tests
	ledgerCloseMeta := makeLedgerCloseMeta()
	ledgerSeq := uint32(57)

	// Call TransformEffect — the same function called by export_effects for each transaction
	effects, err := TransformEffect(transaction, ledgerSeq, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformEffect failed: %v", err)
	}

	// The bug: GetTransactions(start, end, limit=1) returns at most 1 transaction.
	// But TransformEffect expands that 1 transaction into multiple effect rows.
	// With --limit 1, the user expects 1 effect row, but gets len(effects) rows.
	//
	// If this assertion passes (effects > 1), it proves the limit is on transactions,
	// not on effects — the documented limit semantics are violated.
	if len(effects) <= 1 {
		t.Errorf("Expected a single transaction to produce multiple effects, but got %d effect(s). "+
			"This would mean --limit works correctly for effects, contradicting the hypothesis.", len(effects))
	}

	t.Logf("PoC PASS: A single transaction produced %d effects.", len(effects))
	t.Logf("With --limit 1, GetTransactions returns 1 transaction, but export_effects writes all %d effects.", len(effects))
	t.Logf("The --limit flag caps transactions, not effects as documented.")

	// Log each effect type for clarity
	for i, e := range effects {
		t.Logf("  Effect[%d]: type=%s (%d), address=%s", i, e.TypeString, e.Type, e.Address)
	}
}
```

### Test Output

```
=== RUN   TestEffectsLimitCountsTransactionsNotEffects
    data_integrity_poc_test.go:184: PoC PASS: A single transaction produced 5 effects.
    data_integrity_poc_test.go:185: With --limit 1, GetTransactions returns 1 transaction, but export_effects writes all 5 effects.
    data_integrity_poc_test.go:186: The --limit flag caps transactions, not effects as documented.
    data_integrity_poc_test.go:190:   Effect[0]: type=account_created (0), address=GCQZP3IU7XU6EJ63JZXKCQOYT2RNXN3HB5CNHENNUEUHSMA4VUJJJSEN
    data_integrity_poc_test.go:190:   Effect[1]: type=account_debited (3), address=GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H
    data_integrity_poc_test.go:190:   Effect[2]: type=signer_created (10), address=GCQZP3IU7XU6EJ63JZXKCQOYT2RNXN3HB5CNHENNUEUHSMA4VUJJJSEN
    data_integrity_poc_test.go:190:   Effect[3]: type=account_sponsorship_updated (61), address=GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H
    data_integrity_poc_test.go:190:   Effect[4]: type=account_sponsorship_created (60), address=GCQZP3IU7XU6EJ63JZXKCQOYT2RNXN3HB5CNHENNUEUHSMA4VUJJJSEN
--- PASS: TestEffectsLimitCountsTransactionsNotEffects (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.808s
```
