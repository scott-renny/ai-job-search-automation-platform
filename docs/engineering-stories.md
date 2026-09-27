# Engineering Stories

Eight real production incidents from building and operating the AI Job Search Automation Platform, each told as problem → diagnosis → solution → lesson. Details that would identify a specific employer, job posting, or application outcome have been removed; the technical substance is unchanged.

---

## 1. Interactive-session quota and process fragility

**Problem.** Early automation ran as a set of AI-agent tool calls dispatched from within one interactive coding-assistant session, all sharing that session's own usage quota.

**Symptoms.** A batch of parallel discovery tasks all failed simultaneously mid-run when the session hit its own quota limit — with no fallback and nothing left running to notice or recover.

**Diagnosis.** The failure mode wasn't a bug in any individual task; it was architectural. Anything that depends on one interactive session's live quota has exactly one point of failure, and that session has no way to detect its own exhaustion from the inside.

**Solution.** Rewrote each pipeline stage as a standalone process: no dependency on an interactive session, a primary AI provider with an automatic fallback to a secondary one on a confirmed failure, and its own checkpoint so a kill mid-run resumes rather than restarts.

**Lesson.** A system's availability can't exceed the availability of whatever process happens to be driving it. Moving the automation out of an interactive session's lifecycle, and giving it its own fallback path, turned a single point of failure into a resumable, provider-redundant pipeline.

---

## 2. Background process-tree termination

**Problem.** A background automation run was launched as a child process of an interactive session.

**Symptoms.** The run was killed hours in, mid-batch, by the exact same cleanup logic that tears down an interactive session's child processes when the session ends — with no distinction between "the user is done" and "a long automation run is still in progress."

**Diagnosis.** Anything spawned as a child of an interactive process inherits that process's lifecycle whether or not that's the intended behavior. A multi-hour unattended job has no business being a child of a session that might end for unrelated reasons.

**Solution.** Moved to a properly detached execution model, and eventually to full OS-service-managed execution (systemd): the pipeline's lifetime is owned by the service manager, not by whatever process happened to start it.

**Lesson.** "Runs in the background" and "runs independently" are not the same thing. True independence means the process's lifetime is owned by the operating system, not implied by how it happened to be launched.

---

## 3. Cross-identity atomic-write permission reset

**Problem.** Runtime state files were read and written by two different OS identities: the interactive account and a dedicated, unprivileged service account running the scheduled automation.

**Symptoms.** Files that had been readable by both identities silently became unreadable to one of them after the *other* identity next rewrote the file — with no error raised anywhere, because the reading code treated "permission denied" exactly like "file doesn't exist yet."

**Diagnosis.** A standard atomic-write pattern (write to a temp file, then atomically move it into place) creates that temp file with a fixed, restrictive permission mode by default, regardless of the process's own configured permissions — a detail that's invisible until two different identities are involved.

**Solution.** Built a single shared atomic-write utility that explicitly preserves the destination file's existing ownership/permission mode across the swap, and migrated every runtime-state writer in the system onto it, so the bug class can't be silently reintroduced by a future writer copying the naive pattern.

**Lesson.** A correctness bug that only appears under multi-identity access is easy to miss in single-user development and easy to mistake for "it just doesn't exist yet" in production. Centralizing the fix in one tested primitive, rather than patching each call site, is what actually prevents recurrence.

---

## 4. A security review catching an overly broad privilege design — before it shipped

**Problem.** Account-creation credentials needed to be encrypted using the host's native secret-sealing mechanism, but the service account running the automation didn't have permission to invoke that mechanism directly.

**First design (rejected).** The first fix considered was a privilege-escalation rule granting the service account permission to run the encryption tool directly. On review, that design was rejected: the way that particular privilege-grant mechanism matches commands doesn't restrict *which* target file or option follows the granted command — meaning the rule as drafted would have granted a general-purpose "decrypt or encrypt anything on the host using this mechanism" capability, not "manage this one application's own credentials."

**Solution.** Replaced it with a minimal, single-purpose, root-owned helper process: it takes no command-line arguments at all, reads a strict, fixed-shape request, re-validates every field itself regardless of what the caller already checked, and operates only inside one directory it alone controls. The privilege grant names only that exact helper — nothing broader.

**Lesson.** The value of a security review isn't that mistakes never happen — it's catching them before they ship. Being willing to document and talk about a rejected first design, rather than only presenting the final one, is itself a demonstration of the judgment that matters in security engineering.

---

## 5. A browser extension interfering with real application submissions

**Problem.** Automated form-filling for application submission was originally driven through a general-purpose, extension-based browser-automation tool running in a regular, shared browser profile.

**Symptoms.** A third-party autofill extension present in that shared profile intercepted real "Apply" button clicks on more than one job-application platform, surfacing its own account-setup prompt instead of the employer's actual form.

**Diagnosis.** Any automation that shares a browser profile with normal daily use inherits every extension and every bit of state in that profile — state the automation never asked for and can't fully control.

**Solution.** Replaced it with a self-managed automation tool that launches a fresh, disposable browser profile per session, with extensions explicitly disabled and no shared state with any browser used interactively.

**Lesson.** Isolation isn't just a security property — it's a correctness property. Sharing execution context with unrelated software (even something as ordinary as a browser extension) is a real, demonstrated source of automation failures, not just a theoretical risk.

---

## 6. A CAPTCHA a text-only scan couldn't see

**Problem.** The safety gate that detects "this form can't be safely auto-submitted" (CAPTCHA, MFA, blocked account creation) originally scanned only the page's visible text for known keywords.

**Symptoms.** A live CAPTCHA on a real application form went undetected by that scan, because the CAPTCHA widget was present in the page's structure without rendering any matching visible text at all.

**Diagnosis.** A text-keyword scan can only ever catch what's written in words on the page. Some UI elements (CAPTCHA widgets especially) are identifiable in the page's underlying structure — element attributes, embedded-frame sources — well before, or entirely without, any human-readable text appearing.

**Solution.** Added a second, independent detection pass that inspects the page's actual structure (element identifiers, classes, embedded-frame sources) for known CAPTCHA-vendor signatures, run alongside the original text scan rather than replacing it.

**Lesson.** A detection mechanism is only as good as the signal it actually looks at. When a real miss reveals a blind spot, the right fix is usually a second, differently-shaped check — not a bigger keyword list for the same kind of check.

---

## 7. A platform-wide rate limit that no single job's record could represent

**Problem.** The system's per-application tracking is one record per job. One assumption baked into that design was that any submission limit would be scoped the same way — per job, or at most per employer.

**Symptoms.** A real application-hosting platform's own posted policy turned out to cap submissions across *every employer using that platform*, within a rolling time window — a constraint that spans records the tracker had no way to relate to each other.

**Diagnosis.** The tracker's data model was never wrong for what it was designed to track (one job, one outcome) — the rate limit simply didn't fit that shape at all, and no amount of per-record logic could represent it.

**Solution.** Added a separate, platform-scoped counter, checked before a submission attempt and re-checked immediately before the final submit action, entirely independent of the per-job tracker.

**Lesson.** When a new constraint doesn't fit an existing data model, the right move is usually a new, narrowly-scoped piece of state next to it — not contorting the original model to cover a case it was never designed for.

---

## 8. Runaway retries on postings that could never succeed

**Problem.** A posting that failed to fetch (a dead link, a blocked page, a malformed listing) was simply retried again on every subsequent pipeline run, with no limit.

**Symptoms.** A real backlog of postings accumulated tens of thousands of failed-fetch attempts across repeated runs — the same postings, failing the same way, every single time, forever.

**Diagnosis.** "Retry on failure" without a bound isn't resilience — it's an unbounded loop that happens to be spread across scheduled runs instead of a single process. Nothing distinguished "this will probably work next time" from "this can never succeed."

**Solution.** Added a bounded failure counter per item and a genuine terminal state once that bound is reached — the item stops being retried and is recorded as such, rather than being silently reattempted forever or silently dropped.

**Lesson.** Every retry loop needs an explicit answer to "and then what, if it never succeeds?" A terminal state isn't a failure to handle gracefully — it's the thing that makes the retry itself meaningful.
