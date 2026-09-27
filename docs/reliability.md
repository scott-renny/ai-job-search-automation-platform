# Reliability & Operations

This document covers how the AI Job Search Automation Platform stays correct and available across crashes, restarts, provider outages, and concurrent access from multiple identities: checkpointing, atomic state management, provider fallback, quota and rate-limit handling, bounded retries, service execution, and recovery.

## Checkpointing and resumability

Every multi-stage orchestrator (discovery, ranking, drafting/assembly, submission) persists its own progress after each unit of work, not just at the end of a run. If a process is killed — a crash, a scheduled stop, a quota exhaustion mid-batch — the next run resumes exactly where it left off. No stage ever redoes work that already succeeded, and no stage ever fabricates a result for work it didn't actually complete; an interrupted item is simply left at its last successfully-completed state for the next run to pick up.

## Atomic state management

All runtime state (dedup records, pipeline status, per-item progress) is written atomically: a write goes to a temporary location and is atomically swapped into place, so a reader never observes a partially-written file. This project's state files are read and written by more than one OS identity (an interactive account and the dedicated service account) — a genuinely tricky case, since a naive atomic-write pattern can silently reset a file's permissions the moment a different identity next rewrites it, breaking every other reader with no visible error. That failure mode was found and fixed in production; see [`engineering-stories.md`](engineering-stories.md) for the full story. The fix is now a single shared, tested primitive every writer in the system uses, so the bug class can't quietly reappear in new code.

## AI provider fallback

Job-fit ranking and document drafting each have a primary and a secondary AI provider. On a confirmed failure of the primary — not a guess, an actual verified failure signal — the request is retried against the secondary automatically. Every provider decision, for every single item, is logged (which provider, success or failure), so provider health and cost are fully auditable after the fact rather than living only in a scrollback buffer. If both providers fail for a given item, that item is left untouched for a later attempt — never a fabricated score or a fabricated draft standing in for a real one.

## Quota and rate-limit handling

- **Token-budget guarding.** Calls to the secondary provider carry their own per-call token-budget check, plus a rolling-window throttle for batch operations, so a burst of otherwise-small calls can't silently exceed a provider's own rate ceiling.
- **Platform-wide submission limits.** Some application-tracking platforms enforce a submission limit shared across every employer using that platform, not just per job. That's tracked as its own piece of state, independent of the per-job tracker, because a per-job record has no way to represent a limit that spans unrelated jobs. See [`engineering-stories.md`](engineering-stories.md).

## Bounded retries, real terminal states

Nothing retries forever. A posting that repeatedly fails to fetch, or a document that repeatedly fails automated assembly/validation, is retried up to a fixed bound and then moved to an explicit terminal state with a recorded reason — never silently retried indefinitely, and never silently dropped. A real production incident (postings retried on every single run indefinitely, accumulating tens of thousands of failed attempts) is exactly why every stage in this pipeline now has a bounded-retry-then-terminal-state pattern, not just the stage where the incident was first found.

## Service-managed execution

The pipeline runs as a properly scheduled system service, not a process spawned by and dependent on an interactive session. This followed a real incident where a background automation run was killed by the same cleanup logic that tears down an interactive session's child processes mid-run, losing hours of progress with no way to resume other than starting over. The current design has the process's lifetime owned entirely by the OS service manager: it survives the terminal, the interactive session, or the tool that originally triggered a run being closed.

The scheduling design itself went through a deliberate simplification: an earlier "restart on any failure, indefinitely" daemon pattern was replaced with a fixed-schedule, run-to-completion unit with a small, bounded number of automatic retries on failure — a design that surfaces a genuine problem (as a visibly failed, inspectable run) instead of masking it behind infinite silent restarts.

## Recovery and observability

- A live status/heartbeat file reflects the current stage of any in-progress run, consumed by both the operational dashboard and periodic reports.
- A full run log is captured per execution.
- A daily report is generated from real on-disk state only — discovery yield, qualification counts, verified submissions, current backlog size — never sample or placeholder figures, and never silently regenerated in a way that could duplicate a day's numbers.

## What's still evolving

Not every piece of the interactive workflow has been fully migrated into the standalone, unattended runtime. A couple of secondary checks (bulk-posting detection, a periodic source-health check) currently still rely on the original interactive workflow rather than the standalone pipeline — a deliberate, disclosed scope decision, not an oversight, and neither blocks the unattended pipeline from completing a normal run.
