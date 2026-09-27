# Case Study: AI Job Search Automation Platform

*Engineering a reliable, security-conscious AI workflow for real-world job-search automation*

## Provenance

Starting from an open-source job-application framework as a foundation, I designed and built the production automation and security-engineering layer that transformed it into the system described here.

That original framework provided a manually-driven, interactive workflow for evaluating a job posting and drafting a tailored CV and cover letter. Everything else in this case study — the unattended multi-stage pipeline, the deterministic eligibility and safety gates, the AI-provider fallback and quota handling, the browser-automation layer, the credential-security architecture, the atomic state-management layer, the systemd deployment, and the operational dashboard — was engineered on top of it, with no equivalent in the original project.

## Problem

Actively job searching well takes real, repeated effort: finding relevant postings across scattered sources, evaluating fit honestly, producing a genuinely tailored (not generic) application for each one, tracking status across dozens of open threads, and doing all of it consistently over weeks or months. Doing this manually doesn't scale, and doing it with a fragile personal script trades one set of problems for another — silent failures, lost work, and no real safety net around the one action that actually matters: submitting something on my behalf.

I wanted to apply real production-engineering discipline to this problem, not a one-off automation script — and to do it in a way that could never fabricate anything about my own experience.

## Evolution

The system did not start where it is now. Early automation ran as a sequence of AI-agent tool calls inside an interactive coding-assistant session — convenient to build, but fragile in ways that only showed up under real, sustained use: a shared usage quota that could be exhausted mid-batch with no fallback, and a process lifecycle tied to a session that could end for reasons unrelated to whether the work was actually done. Both failure modes happened for real, more than once, and both drove the same underlying architectural shift: move the automation out of the interactive session entirely, into standalone, checkpointed processes with their own provider fallback, and eventually into properly scheduled, OS-managed services. The eight incidents in [`engineering-stories.md`](engineering-stories.md) trace that evolution in detail.

## Architecture

At a high level, the pipeline discovers postings, deduplicates them against persistent state, filters them through deterministic eligibility rules before any AI call is made, scores the survivors for fit with an LLM, generates tailored application documents from verified candidate information, runs an independent AI reviewer pass, mechanically assembles and validates the final PDFs, and then either submits the application automatically (if every safety gate clears) or stages it for me to complete by hand. Full detail in [`architecture.md`](architecture.md) and the diagram in [`architecture-diagram.md`](architecture-diagram.md).

The design consistently favors deterministic, testable rules over LLM judgment wherever a fixed rule can substitute — a seniority filter, a commute-distance limit, a per-platform rate limit, and a CAPTCHA/hard-stop scan are all plain code, not prompts. AI is reserved for the parts of the problem that genuinely need it: fit scoring, document drafting, and an independent reviewer critique of that drafting.

## Difficult engineering problems

Some of the hardest problems weren't in the AI logic at all — they were in making an unattended, multi-process, multi-identity system actually reliable:

- A background automation run killed mid-batch by the same cleanup logic that tears down an interactive session's own child processes.
- A permission-reset bug where runtime state files silently became unreadable to one system identity the moment a *different* identity next rewrote them — invisible until multiple identities were genuinely in play.
- A hidden CAPTCHA widget that a text-only detection scan couldn't see, because it never rendered matching visible text at all.
- A real application platform enforcing a submission limit shared across every employer on that platform — a constraint the per-job data model had no way to represent until a separate, platform-scoped counter was added.

Full write-ups, sanitized of any identifying detail, are in [`engineering-stories.md`](engineering-stories.md).

## Security and reliability decisions

The single decision I'm most willing to talk about in detail: an early design for handling a privileged credential-encryption operation would have granted the automation's service account a broad, general-purpose capability, because the privilege-grant mechanism being used didn't actually restrict *which* target the granted command could act on. That was caught on my own review before it ever reached production, and replaced with a minimal, single-purpose, root-owned helper that accepts no arguments and re-validates its own input regardless of what called it. I think the willingness to document a rejected first design — not just the final one — is itself evidence of the judgment a security-conscious engineering process is supposed to produce. Full detail in [`security.md`](security.md).

Beyond that: the service runs under a dedicated, least-privilege account (never the interactive login account, never root); every dependency on an external service fails closed rather than degrading silently; and the one consequential action in the system — submitting an application — has a deliberately preserved human-authority boundary. A CAPTCHA, an MFA prompt, a security question, or incomplete information all correctly stop automated progress rather than being guessed past.

## Results

The pipeline runs in production, on a fixed schedule, as systemd-managed services. Some eligible applications have completed through fully automated submission once every deterministic gate cleared; others — correctly — were routed to human-supervised completion when a gate detected a CAPTCHA, a security question, or an ambiguous requirement the system wasn't built to guess at. That split is the intended behavior, not an incomplete feature. The automated test suite currently stands at 2,286 passing tests, 8 environment-dependent skips, and 143 passing parameterized subtests, with 2 failing tests that are themselves environment-dependent rather than logic defects — see [`testing.md`](testing.md) for the exact, unrounded numbers and why.

## Lessons learned

- **Availability is bounded by whatever you depend on that you don't control.** Tying long-running automation to an interactive session's lifecycle or quota is a single point of failure hiding in plain sight.
- **A retry without a bound isn't resilience.** Every retry loop needs an explicit, terminal answer to "what happens if this never succeeds."
- **The most convincing security story isn't "we designed it right the first time."** It's catching a design mistake in review and being able to explain exactly why the fix is narrower and better — before anything shipped.
- **Precision about what a safeguard actually does is part of the engineering, not just the writing.** Describing an AI-assisted, reviewer-checked generation process as if it were a mechanical fact-verifier would be a step backward from the actual, honest design.
