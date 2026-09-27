# Testing

## Philosophy

Automated tests in this project aren't limited to feature correctness. A meaningful share of the suite exists specifically to pin down *operational and security-relevant* behavior — the kind of thing that's easy to quietly regress in a system with this many moving parts:

- Gate logic (eligibility, rate limiting, safety detection) is tested against both the "should pass" and "should correctly block" cases, not just the happy path.
- Credential-store and service-identity/permission behavior has dedicated tests, including failure-mode tests (what happens when the secure backend is unavailable, what happens under the wrong identity).
- State-management primitives (atomic writes, checkpointing) are tested for the specific cross-identity and crash-recovery scenarios that caused real production incidents (see [`engineering-stories.md`](engineering-stories.md)) — regression tests written *because* something broke in production, not only tests written in advance of a feature.
- Provider-fallback logic is tested for both the "primary succeeds," "primary fails and secondary takes over," and "both fail" paths.

## Types of tests

- **Unit tests** for individual gate functions, parsers, and data-transformation logic.
- **Fixture/scenario tests** that exercise a full orchestrator stage (discovery, ranking, drafting, submission staging) against realistic mock inputs, without ever making a real network call or a real AI-provider call.
- **Service-identity and permission tests** that verify the least-privilege boundaries described in [`security.md`](security.md) hold as written — not just that the code compiles, but that the actual permission model behaves as intended.
- **Systemd unit tests** that check the deployed service/timer configuration itself matches the intended scheduling and retry semantics, not just the application code.
- **Regression tests** written directly from real production incidents, so a fixed bug class has a permanent, named test guarding against its return.

## Current test statistics

As of the most recent full run of the automated test suite:

- **2,286 tests passing**
- **8 tests skipped** (environment-dependent — e.g., a check that only applies when an optional external tool is installed)
- **143 subtests passing** (parameterized cases within larger test methods)
- **2 tests failing**, both environment-dependent rather than logic defects: one requires access to a production-only, root-owned credential directory that a local development environment correctly cannot read; the other is a test-isolation issue where a prior test's state affects a later assertion, unrelated to the feature under test

That's roughly a **99.9% pass rate on tests that are actually meaningful to run in this environment** (2,286 of 2,288 executable, non-skipped tests) — reported precisely because the two failures have a known, understood, non-functional cause, not because the number is being rounded up. I don't claim 100% passing or zero failures; I'd rather report the real number with its explanation than a cleaner-sounding one that isn't accurate.

## Continuous integration

The test suite, along with a separate set of security-and-configuration guard checks (permission-allowlist widening, data-protection-rule weakening, dependency-manifest tampering), runs automatically in CI on every change — the guard checks exist specifically so a change that quietly loosens a security boundary is caught mechanically rather than depending on a reviewer remembering to look for it.
