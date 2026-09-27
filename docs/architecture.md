# Architecture

This document describes the sanitized system architecture of the AI Job Search Automation Platform: pipeline stages, major components, data flow, and the deployment model. It describes real production architecture; it does not include any production source code, configuration, or data.

See [`architecture-diagram.md`](architecture-diagram.md) for the visual version.

## Design philosophy

Two rules shape every stage of this pipeline:

1. **Use a deterministic rule wherever a fixed rule can substitute for a judgment call.** Title-based seniority filtering, commute-distance limits, per-platform submission rate limits, and posting-expiry detection are all plain code, not LLM prompts — cheaper, testable, and immune to prompt drift.
2. **Fail closed, never silently degrade.** Every stage that depends on an external service (an LLM provider, a credential backend, a browser session) either succeeds, retries within a bound, or stops and leaves the item in a recoverable, well-defined state — never a guess presented as a result.

## Pipeline stages

**1. Discovery** — Pulls new postings from configured job-source integrations on a schedule. Runs as a standalone process with no dependency on an interactive session, so a crash or restart resumes from a checkpoint instead of starting over.

**2. Deduplication** — Every posting is checked against a persistent, multi-key store (URL, employer+title, and a stable per-source identifier) before it's treated as new. Deduplication happens again independently at ranking and drafting time, so a posting can never be reprocessed even if it re-enters the discovery stream.

**3. Deterministic eligibility gates** — Fast, pre-LLM filters applied before a posting is ever fetched or scored: seniority/title filtering (rejecting postings clearly above an entry-level bar), employment-type filtering (internship/co-op/temporary roles), a straight-line commute-distance gate for on-site/hybrid roles, and occupation-relevance filtering. Anything that fails here never consumes an LLM call or a fetch.

**4. AI-assisted ranking** — Postings that clear the deterministic gates are scored for fit by an LLM against an explicit rubric. A primary/secondary provider pattern (see [`reliability.md`](reliability.md)) means a single provider outage doesn't stall the pipeline — every ranking decision is logged with which provider produced it.

**5. Grounded document generation** — Tailored CV and cover-letter content is generated from a verified candidate-information source, under explicit truthfulness constraints. See [`grounded-generation.md`](grounded-generation.md) for the full explanation of what "grounded" means here and its limits.

**6. Reviewer critique** — A second, independent AI pass reviews the drafted content against the actual posting and the candidate's real background before anything is finalized — a second, differently-prompted check on the same material, not a rubber stamp.

**7. Document assembly & validation** — Final documents are mechanically assembled and compiled to PDF, then validated: page-count and layout checks (an entry's title must never orphan across a page break), and a text-layer extraction check that verifies the document reads correctly to an ATS parser (contact details present as real text, correct reading order, no corrupted glyphs). A validation failure escalates to a review queue rather than shipping a broken document.

**8. Submission safety gates** — Before any submission attempt: domain/platform validation, a CAPTCHA/MFA/account-creation/security-question hard-stop scan (both a text scan and a DOM-level scan, since some of these render with no visible text at all), a required-field completeness check (a field the pipeline didn't confidently populate itself is never guessed), a per-platform submission rate limit, and a live posting-still-open check.

**9. Browser automation OR human review** — If every gate above clears, a disposable, extension-free browser session completes the submission mechanically — it never invents a field value; every value traces back to verified candidate data or the posting itself. If any gate is unclear or fails, the item is staged for manual completion instead, with everything already prepared. This branch point is the **Human Authority at Consequential Boundaries** principle in practice — see [`security.md`](security.md).

**10. Application tracking** — Every outcome (submitted, staged for review, rejected by a gate, expired) is recorded in a single system-of-record, with status transitions that are gated and duplicate-proof.

**11. Dashboard / analytics / reporting** — A live operational dashboard and a periodic report surface pipeline health, backlog, and derived statistics — every number shown is a real derivation from on-disk state, never sample or placeholder data.

## Supporting systems

- **Persistent checkpoint state** — every multi-stage orchestrator records its own progress after each unit of work, so a kill/restart resumes rather than repeats.
- **Provider fallback** — ranking and drafting stages have a primary AI provider and an automatic fallback on a confirmed failure, with per-call token-budget guards and full audit logging of which provider handled which item.
- **Credential boundary** — account-creation credentials and API tokens are stored through an OS-native encrypted-secret mechanism, never in plaintext, accessed only through a narrowly-scoped, least-privilege path. Full detail in [`security.md`](security.md).
- **Service execution** — the pipeline and dashboard run as independently scheduled/managed system services under a dedicated, unprivileged service account — not as a process tied to an interactive login session.
- **Observability** — structured run logs, a provider-decision audit trail, and a daemon status/heartbeat file back every dashboard view and daily report.

## Deployment model

The platform runs as two independently managed services on Linux:

- A scheduled pipeline service that runs the full discovery-through-submission flow on a fixed cadence, plus on-demand
- A dashboard service, bound to the local machine only, providing the live operational view and a small set of explicit, gated write actions (e.g., marking a manually-completed application as submitted)

Both run under a dedicated service account with no more privilege than each specific task requires — not the account I use interactively. Currently deployed on a home-lab Linux server (**Atlas**), part of my broader personal Cyber Operations Center environment, but the application has no hard dependency on that specific host — it's designed to be portable to any comparable Linux environment.

```
Cyber Operations Center (homelab environment)
        │
        └── Atlas (Linux server)
                │
                └── AI Job Search Automation Platform (this project)
```
