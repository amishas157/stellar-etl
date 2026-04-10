# 011: Effects Parquet rewrites null `address_muxed` to empty strings

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformEffect()` preserves whether an effect row came from a classic account or a muxed account by storing `address_muxed` as `null.String`. The Parquet export path destroys that distinction by converting invalid `null.String{}` values into a required `string`, so classic-account effects export `address_muxed=""` while muxed-account effects still export real `M...` addresses.

This is a real structural corruption bug in normal export flow. The package's own muxed-account path-payment fixture produces both classic and muxed effect rows in the same transaction, and only the Parquet conversion collapses the classic row's null state.

## Root Cause

The effect helpers intentionally leave `AddressMuxed` invalid for classic accounts (`addUnmuxed()`) and populate it only for muxed accounts (`addMuxed()`). `EffectOutput` models that field as `null.String`, but `EffectOutputParquet` narrows it to a required `string`, and `EffectOutput.ToParquet()` writes `eo.AddressMuxed.String`, which turns `null.String{Valid:false}` into `""`.

## Reproduction

Run `export_effects --write-parquet` on any ledger range containing muxed-account effects. At the transform/JSON layer, classic-account rows carry `address_muxed=null` and muxed-account rows carry real `M...` addresses; after Parquet conversion, the classic rows instead contain `address_muxed=""`.

## Affected Code

- `internal/transform/effects.go:addUnmuxed/addMuxed:187-197` — classic-account effects keep `AddressMuxed` null while muxed-account effects populate it.
- `internal/transform/schema.go:EffectOutput:360-372` — JSON/output schema models `address_muxed` as `null.String`.
- `internal/transform/schema_parquet.go:EffectOutputParquet:246-258` — Parquet schema narrows `address_muxed` to required `string`.
- `internal/transform/parquet_converter.go:EffectOutput.ToParquet:277-289` — converter serializes `eo.AddressMuxed.String`, collapsing null to `""`.
- `cmd/export_effects.go:33-68` — production parquet export writes `EffectOutputParquet` rows.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestParquetCollapsesNullAddressMuxedToEmptyString`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Add the imports used below, then append the test body to `internal/transform/data_integrity_poc_test.go`
  3. Run `go test ./internal/transform/... -run TestParquetCollapsesNullAddressMuxedToEmptyString -v`
  4. Observe the test fail because the classic effect keeps JSON `null` but exports Parquet `""`

### Test Body

```go
func TestParquetCollapsesNullAddressMuxedToEmptyString(t *testing.T) {
	tx := ingest.LedgerTransaction{
		Index: 0,
	}

	const resultXDR = "AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAANAAAAAAAAAAEAAAAAyOrQaxYm7nh+MP1j/CUknhr2IBC8XzaFEiNPvXq7mwMAAAAAAJmwQAAAAAFBUlMAAAAAAK6PQ/tPy+JSQczfbhuOoaozgC/BdU4p32VJfunH7VlaAAAAAACYloAAAAABQlJMAAAAAACuj0P7T8viUkHM324bjqGqM4AvwXVOKd9lSX7px+1ZWgAAAAAABJPgAAAAAMjq0GsWJu54fjD9Y/wlJJ4a9iAQvF82hRIjT716u5sDAAAAAUFSUwAAAAAAro9D+0/L4lJBzN9uG46hqjOAL8F1TinfZUl+6cftWVoAAAAAAJiWgAAAAAA="
	const metaXDR = "AAAAAQAAAAIAAAADAA0aVQAAAAAAAAAA9sYcesZsvsQUsbztxYDp55wz8tpXLOs76lqQWmNCr48AAAAXSHbi7AANFvYAAAAMAAAAAwAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAABAA0aVQAAAAAAAAAA9sYcesZsvsQUsbztxYDp55wz8tpXLOs76lqQWmNCr48AAAAXSHbi7AANFvYAAAANAAAAAwAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAABAAAACAAAAAMADRo0AAAAAQAAAAD2xhx6xmy+xBSxvO3FgOnnnDPy2lcs6zvqWpBaY0KvjwAAAAFCUkwAAAAAAK6PQ/tPy+JSQczfbhuOoaozgC/BdU4p32VJfunH7VlaAAAAAB22gaB//////////wAAAAEAAAABAAAAAAC3GwAAAAAAAAAAAAAAAAAAAAAAAAAAAQANGlUAAAABAAAAAPbGHHrGbL7EFLG87cWA6eecM/LaVyzrO+pakFpjQq+PAAAAAUJSTAAAAAAAro9D+0/L4lJBzN9uG46hqjOAL8F1TinfZUl+6cftWVoAAAAAHbHtwH//////////AAAAAQAAAAEAAAAAALcbAAAAAAAAAAAAAAAAAAAAAAAAAAADAA0aNAAAAAIAAAAAyOrQaxYm7nh+MP1j/CUknhr2IBC8XzaFEiNPvXq7mwMAAAAAAJmwQAAAAAFBUlMAAAAAAK6PQ/tPy+JSQczfbhuOoaozgC/BdU4p32VJfunH7VlaAAAAAUJSTAAAAAAAro9D+0/L4lJBzN9uG46hqjOAL8F1TinfZUl+6cftWVoAAAAAFNyTgAAAAAMAAABkAAAAAAAAAAAAAAAAAAAAAQANGlUAAAACAAAAAMjq0GsWJu54fjD9Y/wlJJ4a9iAQvF82hRIjT716u5sDAAAAAACZsEAAAAABQVJTAAAAAACuj0P7T8viUkHM324bjqGqM4AvwXVOKd9lSX7px+1ZWgAAAAFCUkwAAAAAAK6PQ/tPy+JSQczfbhuOoaozgC/BdU4p32VJfunH7VlaAAAAABRD/QAAAAADAAAAZAAAAAAAAAAAAAAAAAAAAAMADRo0AAAAAQAAAADI6tBrFibueH4w/WP8JSSeGvYgELxfNoUSI0+9erubAwAAAAFCUkwAAAAAAK6PQ/tPy+JSQczfbhuOoaozgC/BdU4p32VJfunH7VlaAAAAAB3kSGB//////////wAAAAEAAAABAAAAAACgN6AAAAAAAAAAAAAAAAAAAAAAAAAAAQANGlUAAAABAAAAAMjq0GsWJu54fjD9Y/wlJJ4a9iAQvF82hRIjT716u5sDAAAAAUJSTAAAAAAAro9D+0/L4lJBzN9uG46hqjOAL8F1TinfZUl+6cftWVoAAAAAHejcQH//////////AAAAAQAAAAEAAAAAAJujwAAAAAAAAAAAAAAAAAAAAAAAAAADAA0aNAAAAAEAAAAAyOrQaxYm7nh+MP1j/CUknhr2IBC8XzaFEiNPvXq7mwMAAAABQVJTAAAAAACuj0P7T8viUkHM324bjqGqM4AvwXVOKd9lSX7px+1ZWgAAAAB2BGcAf/////////8AAAABAAAAAQAAAAAAAAAAAAAAABTck4AAAAAAAAAAAAAAAAEADRpVAAAAAQAAAADI6tBrFibueH4w/WP8JSSeGvYgELxfNoUSI0+9erubAwAAAAFBUlMAAAAAAK6PQ/tPy+JSQczfbhuOoaozgC/BdU4p32VJfunH7VlaAAAAAHYEZwB//////////wAAAAEAAAABAAAAAAAAAAAAAAAAFEP9AAAAAAAAAAAA"
	const feeChangesXDR = "AAAAAgAAAAMADRpIAAAAAAAAAAD2xhx6xmy+xBSxvO3FgOnnnDPy2lcs6zvqWpBaY0KvjwAAABdIduNQAA0W9gAAAAwAAAADAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAEADRpVAAAAAAAAAAD2xhx6xmy+xBSxvO3FgOnnnDPy2lcs6zvqWpBaY0KvjwAAABdIduLsAA0W9gAAAAwAAAADAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAA=="

	sourceAID := xdr.MustAddress("GD3MMHD2YZWL5RAUWG6O3RMA5HTZYM7S3JLSZ2Z35JNJAWTDIKXY737V")
	tx.Envelope = xdr.TransactionEnvelope{
		Type: xdr.EnvelopeTypeEnvelopeTypeTx,
		V1: &xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: xdr.MuxedAccount{
					Type: xdr.CryptoKeyTypeKeyTypeMuxedEd25519,
					Med25519: &xdr.MuxedAccountMed25519{
						Id:      0xcafebabe,
						Ed25519: *sourceAID.Ed25519,
					},
				},
				Fee:    100,
				SeqNum: 3684420515004429,
				Operations: []xdr.Operation{
					{
						Body: xdr.OperationBody{
							Type: xdr.OperationTypePathPaymentStrictSend,
							PathPaymentStrictSendOp: &xdr.PathPaymentStrictSendOp{
								SendAsset: xdr.Asset{
									Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
									AlphaNum4: &xdr.AlphaNum4{
										AssetCode: xdr.AssetCode4{66, 82, 76, 0},
										Issuer:    xdr.MustAddress("GCXI6Q73J7F6EUSBZTPW4G4OUGVDHABPYF2U4KO7MVEX52OH5VMVUCRF"),
									},
								},
								SendAmount: 300000,
								Destination: xdr.MuxedAccount{
									Type: xdr.CryptoKeyTypeKeyTypeMuxedEd25519,
									Med25519: &xdr.MuxedAccountMed25519{
										Id:      0xcafebabe,
										Ed25519: *xdr.MustAddress("GDEOVUDLCYTO46D6GD6WH7BFESPBV5RACC6F6NUFCIRU7PL2XONQHVGJ").Ed25519,
									},
								},
								DestAsset: xdr.Asset{
									Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
									AlphaNum4: &xdr.AlphaNum4{
										AssetCode: xdr.AssetCode4{65, 82, 83, 0},
										Issuer:    xdr.MustAddress("GCXI6Q73J7F6EUSBZTPW4G4OUGVDHABPYF2U4KO7MVEX52OH5VMVUCRF"),
									},
								},
								DestMin: 10000000,
								Path: []xdr.Asset{
									{
										Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
										AlphaNum4: &xdr.AlphaNum4{
											AssetCode: xdr.AssetCode4{65, 82, 83, 0},
											Issuer:    xdr.MustAddress("GCXI6Q73J7F6EUSBZTPW4G4OUGVDHABPYF2U4KO7MVEX52OH5VMVUCRF"),
										},
									},
								},
							},
						},
					},
				},
			},
		},
	}

	if err := xdr.SafeUnmarshalBase64(resultXDR, &tx.Result.Result); err != nil {
		t.Fatalf("failed to unmarshal result: %v", err)
	}
	if err := xdr.SafeUnmarshalBase64(metaXDR, &tx.UnsafeMeta); err != nil {
		t.Fatalf("failed to unmarshal meta: %v", err)
	}
	if err := xdr.SafeUnmarshalBase64(feeChangesXDR, &tx.FeeChanges); err != nil {
		t.Fatalf("failed to unmarshal fee changes: %v", err)
	}

	effects, err := TransformEffect(tx, 20, makeLedgerCloseMeta(), networkPassphrase)
	if err != nil {
		t.Fatalf("TransformEffect failed: %v", err)
	}
	if len(effects) == 0 {
		t.Fatal("expected muxed-account fixture to emit effects")
	}

	var classicEffect *EffectOutput
	var muxedEffect *EffectOutput
	for i := range effects {
		if effects[i].AddressMuxed.Valid && muxedEffect == nil {
			muxedEffect = &effects[i]
		}
		if !effects[i].AddressMuxed.Valid && classicEffect == nil {
			classicEffect = &effects[i]
		}
	}
	if classicEffect == nil || muxedEffect == nil {
		t.Fatalf("expected both classic and muxed effects, got %#v", effects)
	}

	classicJSON, err := json.Marshal(classicEffect.AddressMuxed)
	if err != nil {
		t.Fatalf("failed to marshal classic AddressMuxed: %v", err)
	}
	if string(classicJSON) != "null" {
		t.Fatalf("expected classic AddressMuxed JSON to be null, got %s", string(classicJSON))
	}

	muxedJSON, err := json.Marshal(muxedEffect.AddressMuxed)
	if err != nil {
		t.Fatalf("failed to marshal muxed AddressMuxed: %v", err)
	}
	if string(muxedJSON) != `"`+muxedEffect.AddressMuxed.String+`"` {
		t.Fatalf("expected muxed AddressMuxed JSON to preserve %q, got %s", muxedEffect.AddressMuxed.String, string(muxedJSON))
	}

	classicParquet := classicEffect.ToParquet().(EffectOutputParquet)
	muxedParquet := muxedEffect.ToParquet().(EffectOutputParquet)
	if muxedParquet.AddressMuxed != muxedEffect.AddressMuxed.String {
		t.Fatalf("expected muxed parquet AddressMuxed %q, got %q", muxedEffect.AddressMuxed.String, muxedParquet.AddressMuxed)
	}
	if classicParquet.AddressMuxed != "" {
		t.Fatalf("expected classic parquet AddressMuxed to collapse to empty string, got %q", classicParquet.AddressMuxed)
	}

	t.Fatalf("BUG CONFIRMED: TransformEffect preserves classic address_muxed as null and muxed address_muxed as %q, but ToParquet rewrites the classic null to %q", muxedParquet.AddressMuxed, classicParquet.AddressMuxed)
}
```

## Expected vs Actual Behavior

- **Expected**: Classic-account effects should remain nullable in Parquet so `address_muxed` stays distinguishable from real muxed `M...` values.
- **Actual**: Classic-account effects export `address_muxed=""`, while muxed-account effects export real `M...` values.

## Adversarial Review

1. Exercises claimed bug: YES — the validated PoC builds a real muxed path-payment transaction, runs `TransformEffect()`, then runs the production `ToParquet()` converter on both classic and muxed effect rows.
2. Realistic preconditions: YES — muxed-account path payments are standard Stellar transactions, and the package already carries this exact fixture in its effect tests.
3. Bug vs by-design: BUG — the ETL explicitly chose nullable semantics in `EffectOutput`, so only the Parquet layer is discarding information.
4. Final severity: High — this is silent structural corruption of a non-financial effect field used for account attribution analytics.
5. In scope: YES — it is a concrete transform/export code path that produces plausible but wrong Parquet data.
6. Test correctness: CORRECT — the PoC proves the transform layer emits distinct JSON-layer states before conversion and that only the Parquet output collapses them.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Make `EffectOutputParquet.AddressMuxed` nullable, such as `*string` with an optional Parquet tag, and emit `nil` when `AddressMuxed.Valid` is false. Audit the rest of `parquet_converter.go` for the same nullable-to-required-string pattern so other muxed or sponsor fields do not silently flatten to empty strings.
