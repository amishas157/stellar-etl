PUBLISHED: amendment
ISSUE: #26
URL: https://github.com/amishas157/stellar-etl/issues/26
RELATED_TO: Ledger-entry Parquet rewrites null sponsor values to empty strings — parquet_converter.go writes .Sponsor.String directly into a non-optional string column, collapsing null.String{Valid:false} to "" for unsponsored accounts, account signers, trustlines, and offers
