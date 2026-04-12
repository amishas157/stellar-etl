# H021: Simultaneous `--testnet` and `--futurenet` silently pick the wrong network

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: wrong-network export selection
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Because `--testnet` and `--futurenet` refer to different Stellar networks, a command receiving both flags might be expected to reject the combination rather than silently exporting one network's data. At minimum, any precedence rule should be explicit and consistent wherever environment selection happens.

## Mechanism

`GetEnvironmentDetails()` checks `IsTest` before `IsFuture`, so a command with both flags exports testnet data and ignores futurenet. If that precedence were undocumented, users could plausibly believe they were exporting futurenet while receiving valid-but-wrong testnet rows.

## Trigger

Run any export command with both `--testnet` and `--futurenet`, for example:

`stellar-etl export_transactions -s 1000 -e 1001 --testnet --futurenet`

## Target Code

- `internal/utils/main.go:GetEnvironmentDetails:886-914` — applies `IsTest` before `IsFuture`
- `internal/utils/main.go:AddCommonFlags:232-246` — exposes both network-selection booleans independently
- `cmd/export_transactions.go:19-25` — representative caller that consumes the resulting environment details

## Evidence

The two flags are not marked mutually exclusive anywhere in Cobra, and the selector is a simple `if commonFlags.IsTest { ... } else if commonFlags.IsFuture { ... }`. That means dual-flag invocations deterministically export testnet data.

## Anti-Evidence

`README.md:141-143` explicitly documents the exact behavior: "Adding both flags will default to testnet." The implementation in `GetEnvironmentDetails()` matches that stated contract.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected ambiguity is already documented as intentional behavior, so the exporter's actual network selection matches the published contract rather than violating it.

### Lesson Learned

Before escalating flag-precedence surprises, check whether README or help text already defines the precedence rule. A deterministic but perhaps surprising branch order is not a data-integrity bug when the CLI documents that exact outcome.
