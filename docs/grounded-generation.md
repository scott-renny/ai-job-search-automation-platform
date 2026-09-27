# Grounded Application Document Generation

## What this means, precisely

**Application documents are generated from verified candidate information under explicit truthfulness constraints and pass through an automated reviewer critique before document assembly.**

That single sentence is the accurate description of how this system uses AI to produce tailored résumés and cover letters, and it's worth being precise about what it does and doesn't claim.

## What's real, and what AI does

My actual experience, education, certifications, skills, and projects are the fixed input — a verified profile I maintain and control, not something the AI infers or invents. What AI assists with is **tailoring the presentation** of that real material to a specific job posting: choosing which real experience is most relevant, reframing real accomplishments in the target role's terminology, and drafting a cover-letter narrative that connects real background to a specific opportunity. AI never adds a skill, a credential, a job, a metric, or an accomplishment that isn't already in the verified source material.

## The two-layer defense

1. **Explicit truthfulness constraints at generation time.** The system is instructed, in detail, on what it may and may not do: never invent a technology, certification, metric, or accomplishment; never upgrade a skill obtained through education or a personal project into professional/production experience; never invent dates, percentages, or outcomes; and to surface a genuine gap honestly rather than paper over it.
2. **Automated reviewer critique.** Before a document is finalized, a second, independently-prompted AI pass reviews the draft against the actual posting and the candidate's real, verified background — a second check by a differently-focused process, not a rubber stamp on the first pass's own output.

## What this is *not*

Being precise here matters, because it would be easy to overstate this and I want to avoid that:

- **This is not a claim that fabrication is technically impossible.** The safeguard is a disciplined combination of explicit instructions and an independent reviewer pass — not a formal, deterministic fact-verification engine that mechanically checks every generated sentence against a ground-truth database.
- **There is no automated fact-checker in the sense of a system that independently verifies each claim against a source of truth token-by-token.** The verification that exists is instructional (the generation process is told exactly what it may not do) and procedural (a second AI pass reviews the result), not a formally verified guarantee.
- **This does not mean every generated statement has been mechanically proven true.** It means the generation process is constrained, and a second pass exists specifically to catch what the first pass might have gotten wrong.

## Why this level of honesty matters here

A system that tailors how real experience is presented is a legitimate and useful tool. A system that quietly invents experience is a liability to the person using it and a real problem for anyone who trusts its output. Being exact about which one this is — rather than rounding up to "AI verifies everything" — is itself part of the engineering discipline this project is trying to demonstrate.

## Document-quality validation (a separate, genuinely mechanical layer)

Distinct from the *content* truthfulness question above, the *document itself* does go through fully mechanical validation once content is finalized: compiled-PDF layout checks (page count, no orphaned section titles) and a text-layer extraction check that verifies the document reads correctly to an applicant-tracking-system parser. Those checks are deterministic and give a real pass/fail — it's the underlying content-truthfulness question where the safeguard is process-and-review, not a mechanical proof.
