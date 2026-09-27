# Security Design

This document describes the security-relevant design decisions in the AI Job Search Automation Platform: the threat model, least-privilege architecture, credential handling, fail-closed behavior, deterministic gates, browser isolation, and where human authority is deliberately preserved.

It describes real, implemented decisions. It omits exact implementation details (specific file paths, script names, account names, and privilege-grant syntax) that would increase attack surface without adding anything meaningful to a technical narrative — the pattern is what matters, not a working recipe.

## Threat model, briefly

This system combines two things that don't normally sit together: an AI agent with file and execution access, and untrusted third-party content (job postings fetched from the open web). Posting text is treated as data to evaluate, never as instructions to follow — the pipeline does not act on directions embedded in a posting, and does not fetch links found inside posting text. Separately, the system holds real credentials (for job-board accounts it may create) and performs a genuinely consequential action (submitting something on my behalf) — both get their own dedicated controls below.

## Deterministic rules over LLM judgment

Every eligibility and safety decision that can be expressed as a fixed rule is implemented as one: title/seniority filtering, commute-distance limits, per-platform submission rate limits, and CAPTCHA/hard-stop detection are all plain, tested code — not a prompt. This isn't just a cost optimization: a deterministic rule is auditable, testable, and cannot drift or be argued out of its own logic the way a model call can. LLM judgment is reserved for what genuinely requires it — fit scoring, document drafting, and reviewer critique — and even there, the *gates* around what the LLM's output is allowed to do stay deterministic.

## Fail-closed, never silently degraded

Every component that depends on something that can be unavailable — an AI provider, a credential backend, a browser session — is written to fail closed: raise a clear, typed error and leave the affected item in a recoverable state, never fall back to an insecure or unverified default. A credential backend that isn't available blocks the operation that needed it; it does not fall back to writing a credential in plaintext. A provider that fails is retried or handed to a fallback provider under an explicit, logged policy — never silently assumed to have succeeded.

## Least privilege, end to end

- **Dedicated service execution.** The pipeline and dashboard run under a dedicated, unprivileged service account — not the account I use interactively, and not root. Neither service process is ever elevated.
- **Narrow, single-purpose privilege escalation only where unavoidable.** One specific operation (encrypting/decrypting an account-creation credential using the OS's native secret-sealing mechanism) genuinely requires access this service account doesn't have. Rather than granting broad access to that mechanism, the actual production design routes that one operation through a minimal, single-purpose, root-owned helper that accepts a fixed, pre-validated request shape, operates only within one fixed directory it alone controls, and independently re-validates every input regardless of what the calling process already checked. The privilege grant that invokes it names only that exact helper, with no arguments and no wildcard — see [`engineering-stories.md`](engineering-stories.md) for how that design was arrived at.
- **Scoped, verb-limited authorization for the one privileged UI action.** The dashboard's "run the pipeline now" action needs to start one specific system service as an unprivileged process. The authorization for that action is scoped to exactly that one service, exactly the "start" action, and exactly the service account involved — not broader service-management rights, and no path to a root shell.

## Credential and token handling

Account-creation credentials and any long-lived API/OAuth tokens are stored exclusively through an OS-native, encrypted-at-rest secret mechanism — never as plaintext on disk, never as a command-line argument (which would leak into process listings or shell history), and never logged. Sensitive values enter the system only via prompted, hidden input at setup time. Non-secret bookkeeping metadata (which accounts exist, their status) is deliberately stored separately from the encrypted secrets themselves, so routine operational tooling never needs access to the secret store at all.

## Browser automation isolation

Application submission uses a self-managed, disposable browser session — a fresh, isolated profile per run, with no extensions loaded and no shared state with any browser I use interactively. This replaced an earlier design that reused a general-purpose interactive browser session; a real incident (a third-party browser extension intercepting real form-submission clicks — see [`engineering-stories.md`](engineering-stories.md)) is exactly why that isolation exists now. The automation never enters credentials into a third-party site itself for authenticated flows it can't safely complete — those are deliberately left for a human.

## Human Authority at Consequential Boundaries

The one action in this system with real, external consequences is submitting an application on my behalf. That action is gated, not gatekept by convenience: a CAPTCHA, an MFA prompt, an account-creation step, a security question, an ambiguous free-text question, incomplete required information, or any signal the posting itself may no longer be open all stop automated progress and hand the fully-prepared item to me instead. This is a deliberate design principle, not a limitation the system is working around — the system is not presented as, and does not attempt to be, fully autonomous at the one point where autonomy would matter most.

## Defense against accidental widening

Automated checks in continuous integration guard the project's own security posture against accidental regression: permission/allowlist changes, weakening of data-protection rules, and dependency-manifest changes that could smuggle in unwanted code execution are all flagged automatically if a change touches them, rather than relying on a reviewer remembering to check.

## Testing of security-relevant behavior

Gate logic, credential-store behavior, rate limiting, and service-identity/permission behavior all have dedicated automated test coverage — not just the "happy path" feature logic. See [`testing.md`](testing.md).
