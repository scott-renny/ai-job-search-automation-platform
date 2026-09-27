# Synthetic Demo Data Specification

This directory is a placeholder for a future, fully synthetic dataset used only to produce dashboard screenshots/video for the public showcase. **No demo data exists yet, and no screenshot has been taken.** This file specifies what needs to be built and the rule that governs it.

## The rule

**Nothing derived from real production data may ever be used for a public screenshot or demo, even redacted.** Redacting a screenshot of real data is unreliable — a cropped or blurred field can still leak through metadata, adjacent context, or a reviewer's own recognition of a real company name or posting. The only acceptable approach is a **separate, fully fictional dataset**, generated specifically for this purpose, loaded into a demo instance of the dashboard that never touches real tracker data, real credentials, or real pipeline state.

## What the synthetic dataset needs

- **Fictional employers** — invented company names that don't resemble real employers I've actually applied to (avoid near-matches to real companies in the same sector).
- **Fictional roles and postings** — plausible entry-level cybersecurity/IT/cloud job titles and descriptions, written from scratch, not copied or lightly edited from any real posting.
- **Fictional URLs** — using an obviously non-resolving placeholder domain (e.g., `careers.example-corp.test`), never a real company's actual domain.
- **A realistic spread of application states** — some `drafted`, some `manual_submit_ready`, some `applied`, some `rejected`/`expired` — enough variety to demonstrate every dashboard view without every row looking identical.
- **Fictional fit scores, dates, and locations** — internally consistent (a "found" date before a "ranked" date before an "applied" date) but not tied to any real timeline.
- **No real personal data** — the demo profile shown in any settings/profile view should use a placeholder name and placeholder contact details, not my own.

## How this gets built (later phase)

1. Generate the synthetic dataset as its own small, checked-in fixture (a fake tracker file plus fake pipeline-state files), reviewed line-by-line before it's used anywhere, specifically to confirm nothing was copied from real data.
2. Point a local, throwaway instance of the dashboard at that fixture data only — never the production instance, never production files.
3. Capture screenshots/video from that throwaway instance.
4. Before publishing any image, re-review it against the same privacy checklist used for this repository's text content (no real names, no real paths, no real hostnames, no service-account names).

## Status

Not started. This file is the specification only — see the main [`README.md`](../README.md) and [`docs/architecture.md`](../docs/architecture.md) for what the demo would actually be illustrating.
