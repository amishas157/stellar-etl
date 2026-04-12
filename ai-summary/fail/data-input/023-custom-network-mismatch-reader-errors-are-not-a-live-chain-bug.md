# H023: Custom network/datastore mismatch suppresses transaction-reader hash errors

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the transaction reader cannot match transaction hashes to envelopes, transaction-based exports should fail loudly instead of silently producing empty slices or zero-valued transactions. A user should get a hard error rather than a plausible but incomplete export.

## Mechanism

Upstream `LedgerTransactionReader.Read()` returns `unknown tx hash in LedgerCloseMeta` when the stored transaction hashes do not match the envelopes it previously hashed with the provided network passphrase. `GetTransactions()`, `GetOperations()`, and `GetTrades()` only stop on `io.EOF`; any other `Read()` error is ignored, so the readers would append zero-valued transactions or skip all operations/trades instead of surfacing the mismatch.

## Trigger

Run a transaction-based export against a datastore whose ledger files do not match the selected network passphrase, such as a custom datastore path wired to one network while the CLI is configured for another.

## Target Code

- `internal/input/transactions.go:GetTransactions:50-62` — ignores non-EOF `txReader.Read()` errors and still appends the returned transaction
- `internal/input/operations.go:GetOperations:52-70` — ignores non-EOF `txReader.Read()` errors and keeps iterating
- `internal/input/trades.go:GetTrades:50-77` — ignores non-EOF `txReader.Read()` errors and keeps iterating
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction_reader.go:76-88` — returns `unknown tx hash in LedgerCloseMeta` on envelope/hash mismatch
- `internal/utils/main.go:990-1006` — normal datastore creation auto-appends the selected network name to the datastore path

## Evidence

The local input loops only check for `io.EOF`, so a real `Read()` error would be suppressed. The upstream reader's only non-EOF runtime path is the hash-mismatch branch at lines 83-88, which would cause exactly the kind of silent drop/zero-value behavior the local loops fail to guard against.

## Anti-Evidence

Normal CLI usage is protected by `CreateDatastore()`, which automatically appends `/pubnet`, `/testnet`, or `/futurenet` to the configured datastore path, so the standard network flags select matching ledger files. That means the remaining trigger requires either a custom miswired datastore layout or malformed backend data, not ordinary legitimate chain data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The silent-reader-error behavior is real, but the reachable trigger depends on custom network/datastore mismatch or corrupted backend data. With the shipped path construction, normal pubnet/testnet/futurenet exports already point at matching network data, so this does not describe a live correctness bug for legitimate chain inputs.

### Lesson Learned

When a swallowed upstream error looks serious, verify that the surrounding configuration code can actually reach that error path in standard production usage. The input loops still merit hardening, but this specific trigger is a misconfiguration/corruption trap rather than an in-scope data-integrity defect.
