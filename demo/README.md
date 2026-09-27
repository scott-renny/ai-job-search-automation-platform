# Synthetic Dashboard Demo

A self-contained, fully synthetic demonstration of the operational dashboard described in [`../docs/architecture.md`](../docs/architecture.md). **Every company, role, date, status, and figure on this page is fabricated.** Nothing here was copied, derived, redacted, or adapted from real production data — it's a fresh, independent static file built to demonstrate what the real dashboard's views look like, without touching real data at all.

Open [`dashboard-demo.html`](dashboard-demo.html) directly in a browser — it's a single static HTML file with inline CSS, no backend, no build step, no dependency on any other file in this repository.

## The rule this followed

**Nothing derived from real production data may ever be used for a public screenshot or demo, even redacted.** Redacting a screenshot of real data is unreliable — a cropped or blurred field can still leak through metadata, adjacent context, or a reviewer's own recognition of a real company name or posting. The safer approach taken here goes one step further than originally planned: rather than pointing a throwaway instance of the real dashboard application at a fake data file, this is a **completely independent, hand-built static page** — there is no code path by which it could ever touch a real credential, a real tracker row, or real pipeline state, because it doesn't run any of that code at all.

## What it demonstrates

One scrollable page (sidebar nav jumps to each section via anchor links), covering every view called for in the platform's real design:

- **Overview** — KPI cards (applications this week vs. target, manual-review queue, active backlog, automated submissions, total tracked)
- **Discovery & Ranking** — a table of fabricated postings with source, discovery date, fit tier, and pipeline status
- **Pipeline & Status** — a funnel view of how many fabricated items sit at each pipeline stage
- **Manual Review** — explicitly framed around **Human Authority at Consequential Boundaries**: each row shows a fabricated item that was stopped by a real category of gate (CAPTCHA, security question, incomplete field, ambiguous question), with prepared materials waiting on a human
- **Recent Activity** — a fabricated timestamped log mixing discovery, gating, ranking, drafting, review, and submission events
- **Analytics** — a donut chart (outcome mix), a weekly bar chart (applications vs. target), and a horizontal bar chart (source breakdown), all fabricated
- **System Health** — service-status pills, a fabricated AI-provider fallback log, and the one genuinely real figure on the page: the audited test-suite count (cross-referenced to [`../docs/testing.md`](../docs/testing.md))

A **"Synthetic demo data"** badge appears in the topbar and again in the sidebar footer, and a one-line disclaimer sits in the page footer — visible without dominating the page.

## The fictional dataset used

Eight invented companies, none resembling any real employer: Northgate Analytics, Cobalt Ridge Security, Meridian Cloud Systems, Lumen Bridge Technologies, Vantage Point Networks, BrightPath Digital, Solace Grid Security, Anchorwell Systems. Roles are generic, plausible entry-level security/IT/cloud titles (SOC Analyst I, IT Security Analyst, Cloud Support Engineer, etc.). Sources are described generically ("Public job board," "Company careers page," "Aggregator API") rather than naming any specific real platform. Dates are relative and illustrative ("Demo Day −1," "Demo Day −2") rather than real calendar dates. All counts, percentages, and log entries are invented and internally consistent, not derived from any real run.

## Screenshots

Rendered from the static file above, for anyone browsing this repo who wants a quick look without opening the HTML file themselves:

- [`screenshots/dashboard-overview-fold.png`](screenshots/dashboard-overview-fold.png) — above-the-fold view
- [`screenshots/dashboard-overview.png`](screenshots/dashboard-overview.png) — full-page view (all sections)
- [`screenshots/architecture-visual-preview.png`](screenshots/architecture-visual-preview.png) — rendered preview of [`../docs/architecture-visual.svg`](../docs/architecture-visual.svg)

## Status

Done, for Phase 2. Not yet used anywhere public — no image from this directory has been published, posted, or embedded outside this local repository.
