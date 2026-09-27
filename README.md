# AI Job Search Automation Platform

**Engineering a reliable, security-conscious AI workflow for real-world job-search automation**

*A portfolio engineering project by **Scott Renny**.*

> This is a documentation-only public showcase. It describes the architecture, engineering decisions, and operational lessons behind a real production system I built and run — it is not the production source repository, and it contains no production code, production data, or upstream source code. See [Provenance](#provenance--what-this-repository-is) below.

## What is this?

A production automation system that discovers job postings, filters them against explicit eligibility rules, ranks fit using an LLM, generates tailored application documents from my own verified background, routes the result through an automated reviewer pass and document validation, and — only when a chain of deterministic safety gates is satisfied — submits the application. When any gate detects ambiguity (a CAPTCHA, a security question, an incomplete field, an unclear posting), the system deliberately stops and hands control back to a human instead of guessing.

It runs unattended, on a fixed schedule, as a set of systemd-managed services on Linux, with its own dedicated service identity, encrypted credential storage, and a live operational dashboard.

## Why I built it

Manually sourcing, evaluating, tailoring documents for, and tracking job applications doesn't scale, and doing it well takes real repeated effort. I wanted to apply actual production-engineering discipline — checkpointing, fallback, least privilege, auditability, test coverage — to my own job search, rather than writing a fragile one-off script, and to do it in a way that could never fabricate anything about my own experience.

## What engineering problems it solves

- Reliable long-running automation that survives process crashes, session limits, and provider outages without losing work or duplicating it
- Turning ad hoc LLM judgment calls into deterministic, testable rules wherever a fixed rule is sufficient
- Running privileged operations (credential storage, service scheduling) under least privilege rather than broad, convenient access
- Keeping a human in control of the one action in the system that actually matters — submitting something on my behalf

## What makes it more than an AI script

The AI calls (job-fit scoring, document drafting, reviewer critique) are a small part of a larger system: deterministic pre-filters that never touch an LLM, a two-provider fallback path with its own token-budget and quota handling, checkpointed multi-stage orchestration that resumes cleanly after a crash, atomic state management across multiple concurrent identities, a credential-security architecture built around least privilege, and a full automated test suite covering the operational and security-relevant behavior, not just the "happy path." The full breakdown is in [`docs/architecture.md`](docs/architecture.md).

## What I personally engineered

This project started from an open-source, manually-driven job-application framework that handled CV/cover-letter drafting through interactive AI-assistant commands. I designed and built the production automation layer on top of it from scratch: the unattended multi-stage pipeline, the deterministic eligibility and safety gates, the AI-provider fallback and quota-handling logic, the browser-automation layer, the credential-security architecture, the atomic state-management layer, the systemd deployment, and the operational dashboard. None of that has an equivalent in the original project. See [Provenance](#provenance--what-this-repository-is) for the exact framing, and [`docs/portfolio-case-study.md`](docs/portfolio-case-study.md) for the full story.

## How AI is used responsibly

Application documents are generated from verified candidate information under explicit truthfulness constraints and pass through an automated reviewer critique before document assembly — a pattern I call **Grounded Application Document Generation**. The underlying experience, education, certifications, and skills are real; AI assists with tailoring how they're presented to a specific opportunity, never with inventing what they are. Full explanation, including honest limitations, in [`docs/grounded-generation.md`](docs/grounded-generation.md).

## Where human control remains

I call this design principle **Human Authority at Consequential Boundaries**: the system can carry an eligible application all the way through to automated submission when every deterministic gate is satisfied, but a CAPTCHA, an MFA prompt, an account-creation step, a security question, an ambiguous free-text question, or incomplete required information all deliberately stop automated progress and route the item to me instead. That is a design decision, not a gap — the system is not, and is not presented as, fully autonomous.

## How it's deployed

Runs as a set of systemd services (a scheduled pipeline unit plus a dashboard service) under a dedicated, unprivileged service account on a Linux server — currently a home-lab server I call **Atlas**, part of my broader personal Cyber Operations Center environment. The application itself has no dependency on that specific host; it's designed to be portable to any Linux system with the same runtime.

## Security and reliability lessons from operating it

Real incidents shaped real design changes: a background pipeline run killed by its parent process's cleanup led to proper service-managed execution; a permission-reset bug caused by a Python stdlib default led to a shared atomic-write layer; a first-draft privileged-credential design was caught in my own security review as overly broad and replaced before it ever shipped. Eight of the strongest stories, told problem → diagnosis → solution → lesson, are in [`docs/engineering-stories.md`](docs/engineering-stories.md).

## Learn more

| Doc | Covers |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | Pipeline stages, components, data flow, deployment model |
| [`docs/architecture-diagram.md`](docs/architecture-diagram.md) | Mermaid diagram + spec for a polished graphical version |
| [`docs/security.md`](docs/security.md) | Least privilege, credential handling, fail-closed design, human-authority boundaries |
| [`docs/reliability.md`](docs/reliability.md) | Checkpointing, atomic state, provider fallback, rate limiting, recovery |
| [`docs/testing.md`](docs/testing.md) | Testing philosophy and current, accurately-reported test statistics |
| [`docs/grounded-generation.md`](docs/grounded-generation.md) | How AI-assisted document generation stays truthful, and its real limits |
| [`docs/engineering-stories.md`](docs/engineering-stories.md) | Eight sanitized real-incident deep dives |
| [`docs/portfolio-case-study.md`](docs/portfolio-case-study.md) | Long-form case study: problem, evolution, architecture, decisions, lessons |
| [`demo/README.md`](demo/README.md) | Plan for a synthetic-data dashboard demo (no real data) |

## Provenance & what this repository is

This showcase is curated documentation, not a mirror of the production codebase — it contains no production source code, no production data, and no upstream source code. Starting from an open-source job-application framework as a foundation, I designed and built the production automation and security-engineering layer that transformed it into the system described here. See [`docs/portfolio-case-study.md`](docs/portfolio-case-study.md) for the full context.

## License

Copyright © 2026 Scott Renny. All rights reserved.

This applies to the original showcase documentation, diagrams, and other original material in this repository. It does not apply to, and does not imply any ownership over, the upstream open-source framework referenced under [Provenance](#provenance--what-this-repository-is) — this repository contains no upstream source code. No open-source license has been selected for this repository at this time; if original source code is published here later, its licensing will be decided separately.
