# H002: Conflicting network flags silently export testnet data

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a user sets both `--testnet` and `--futurenet`, the ETL should reject the invocation as an invalid configuration instead of exporting data from one network arbitrarily. Exported JSON/Parquet should always come from the network the CLI invocation unambiguously selected.

## Mechanism

`MustCommonFlags` and `get_ledger_range_from_times` read both booleans independently and never enforce mutual exclusion. `GetEnvironmentDetails` then resolves the ambiguity with `if commonFlags.IsTest { ... } else if commonFlags.IsFuture { ... }`, which means `--testnet --futurenet` always selects testnet archives, passphrase, and core config. The CLI therefore emits a structurally valid dataset from the wrong network without warning.

## Trigger

Run any export command with both flags set, for example `stellar-etl export_transactions --testnet --futurenet --end-ledger 1000` or `stellar-etl get_ledger_range_from_times --testnet --futurenet ...`. The command will export testnet results even though the invocation also requested futurenet.

## Target Code

- `internal/utils/main.go:MustCommonFlags:460-537` — accepts both booleans without validation
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — prioritizes `IsTest` over `IsFuture`
- `cmd/export_transactions.go:Run:19-25` — consumes the resolved environment directly for transaction export
- `cmd/get_ledger_range_from_times.go:Run:42-63` — reproduces the same ambiguous network selection in range lookup

## Evidence

The utility layer defines both flags as independent booleans and does not reject the combination anywhere in the repo. `GetEnvironmentDetails` hard-selects the testnet passphrase, archive URLs, and core config first, and every export command uses that environment immediately to build its backend.

## Anti-Evidence

Operators normally set only one network flag, and the help text describes each as "instead of Mainnet," which hints they are intended to be exclusive. But the code does not enforce that intent, so a conflicting invocation still produces an apparently successful export from the wrong chain.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced both flag-parsing paths (`MustFlags` at line 320 and `MustCommonFlags` at line 460) and confirmed that `isTest` and `isFuture` are read as independent booleans with no cross-validation. Both parsers store both values into their respective structs (`FlagValues`, `CommonFlagValues`) without checking for mutual exclusion. `GetEnvironmentDetails` (line 886) uses an `if/else if` chain that always evaluates `IsTest` first, so when both are true, testnet configuration is silently returned. The same pattern exists in `get_ledger_range_from_times.go` (line 42-50) which reads both flags and passes them to `GetLedgerRange`, which constructs a `CommonFlagValues` and calls `GetEnvironmentDetails` — same silent prioritization.

### Code Paths Examined

- `internal/utils/main.go:AddCommonFlags:232-246` — registers `--testnet` and `--futurenet` as independent boolean flags with no `MarkMutuallyExclusive` or equivalent
- `internal/utils/main.go:MustFlags:320-440` — reads both flags, stores in `FlagValues`, no validation of `IsTest && IsFuture`
- `internal/utils/main.go:MustCommonFlags:460-537` — reads both flags, stores in `CommonFlagValues`, no validation of `IsTest && IsFuture`
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — `if IsTest` checked first; `else if IsFuture` never reached when both true; returns testnet passphrase, archive URLs, and core config
- `cmd/get_ledger_range_from_times.go:Run:42-50` — reads both booleans independently and passes them to `input.GetLedgerRange`
- `internal/input/ledger_range.go:GetLedgerRange:32-68` — constructs `CommonFlagValues{IsTest, IsFuture}` and calls `GetEnvironmentDetails`, same silent prioritization

### Findings

The bug is confirmed: no code path anywhere in the repository validates that `--testnet` and `--futurenet` are mutually exclusive. When both are set, every export command and `get_ledger_range_from_times` will silently use testnet configuration (passphrase, archive URLs, core config). The exported dataset is structurally valid but from the wrong network.

Severity is downgraded from High to Medium because:
1. The trigger requires explicit operator error (passing both flags), which is unlikely in routine use
2. The output is internally consistent — it is valid testnet data, not corrupted data
3. The practical risk is in scripted/automated invocations where a flag combination bug could go unnoticed

Despite the unlikely trigger, this is a real missing-validation bug where the code silently resolves an ambiguous configuration instead of failing fast.

### PoC Guidance

- **Test file**: `internal/utils/main_test.go` (or create a new test function)
- **Setup**: Construct a `CommonFlagValues` with both `IsTest: true` and `IsFuture: true`
- **Steps**: Call `GetEnvironmentDetails(commonFlags)` and inspect the returned `EnvironmentDetails`
- **Assertion**: Assert that the returned `Network` is `"testnet"` (not `"futurenet"`) AND that the `NetworkPassphrase` matches the testnet passphrase — confirming the silent prioritization. Alternatively, assert that a proper validation function (if added) returns an error when both flags are set.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/utils/data_integrity_poc_test.go
**Test Name**: "TestConflictingNetworkFlagsSilentlySelectTestnet"
**Test Language**: Go

### Demonstration

The test constructs a `CommonFlagValues` with both `IsTest: true` and `IsFuture: true`, then calls `GetEnvironmentDetails`. The function silently returns testnet configuration (Network="testnet", Passphrase="Test SDF Network ; September 2015") without any error or warning, proving that the `if/else if` chain prioritizes testnet and completely ignores the conflicting futurenet flag. No validation rejects the ambiguous input.

### Test Body

```go
package utils

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/network"
)

// TestConflictingNetworkFlagsSilentlySelectTestnet demonstrates that when both
// IsTest and IsFuture are true, GetEnvironmentDetails silently returns testnet
// configuration instead of rejecting the conflicting input.
func TestConflictingNetworkFlagsSilentlySelectTestnet(t *testing.T) {
	// Construct input with both network flags set — an ambiguous configuration
	conflictingFlags := CommonFlagValues{
		IsTest:   true,
		IsFuture: true,
	}

	details := GetEnvironmentDetails(conflictingFlags)

	// The function silently resolves the conflict by prioritizing testnet.
	// This proves the bug: futurenet was also requested but is ignored.
	if details.Network != "testnet" {
		t.Errorf("Expected Network to be silently resolved to 'testnet', got %q", details.Network)
	}

	if details.NetworkPassphrase != network.TestNetworkPassphrase {
		t.Errorf("Expected testnet passphrase, got %q", details.NetworkPassphrase)
	}

	// Verify that futurenet configuration is NOT returned despite IsFuture being true
	futurenetPassphrase := "Test SDF Future Network ; October 2022"
	if details.NetworkPassphrase == futurenetPassphrase {
		t.Errorf("Got futurenet passphrase — expected testnet to win the silent priority")
	}

	if details.Network == "futurenet" {
		t.Errorf("Got futurenet network — expected testnet to win the silent priority")
	}

	t.Logf("BUG CONFIRMED: Both IsTest=true and IsFuture=true accepted without error.")
	t.Logf("GetEnvironmentDetails silently returned testnet config: Network=%q, Passphrase=%q",
		details.Network, details.NetworkPassphrase)
	t.Logf("Futurenet flag was silently ignored — no error, no warning.")
}
```

### Test Output

```
=== RUN   TestConflictingNetworkFlagsSilentlySelectTestnet
    data_integrity_poc_test.go:41: BUG CONFIRMED: Both IsTest=true and IsFuture=true accepted without error.
    data_integrity_poc_test.go:42: GetEnvironmentDetails silently returned testnet config: Network="testnet", Passphrase="Test SDF Network ; September 2015"
    data_integrity_poc_test.go:44: Futurenet flag was silently ignored — no error, no warning.
--- PASS: TestConflictingNetworkFlagsSilentlySelectTestnet (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/utils	0.792s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-10
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC actually exercise the claimed issue?** Yes. The test calls `GetEnvironmentDetails(CommonFlagValues{IsTest: true, IsFuture: true})` and reaches the `if commonFlags.IsTest { ... } else if commonFlags.IsFuture { ... }` branch in `internal/utils/main.go:886-904`, proving testnet wins when both flags are set.
2. **Are the preconditions realistic?** Partially. A user can pass both CLI flags, but this requires an explicitly conflicting invocation rather than a normal export path.
3. **Is the behavior a bug or by design?** By design. The README documents the exact precedence rule: “Adding both flags will default to testnet. Each stellar-etl command can only run from one network at a time.” (`README.md:141-143`). The implementation in `GetEnvironmentDetails` matches that documented contract.
4. **Does the impact match the claimed severity?** No security or data-integrity severity applies because the observed output is the documented network-selection result, not corrupted export data.
5. **Is the finding in scope?** The code path is real, but it does not demonstrate silent data corruption under the project framing; it demonstrates documented CLI precedence for an invalid/ambiguous flag combination.
6. **Is the test itself correct?** Yes. The test is simple and directly verifies the precedence behavior.
7. **Can the results be explained without the claimed issue?** Yes. The result is fully explained by documented testnet-over-futurenet precedence, so the PoC does not establish a bug.
8. **Is this finding novel?** Irrelevant once the behavior is rejected as intended/documented.

### Rejection Reason

The PoC reproduces documented behavior, not a defect. `GetEnvironmentDetails` deterministically prefers testnet when both flags are set, and the README explicitly tells operators that this combination defaults to testnet.

### Failed Checks

- 3. Bug vs by-design
- 7. Alternative explanation
