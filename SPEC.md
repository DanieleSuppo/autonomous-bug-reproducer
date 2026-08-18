# SPEC: Autonomous Bug Reproducer MVP

## Problem Statement

Developers and maintainers need a trustworthy, independently executable reproduction of a reported bug at a known repository revision before attempting a repair. Existing bug reports can be ambiguous, incomplete, or detached from executable conditions. A generated failing test is insufficient when it fails for setup, incidental, or artificially introduced reasons.

ABR must turn one immutable bug report and one immutable Target Repository revision into a bounded, inspectable investigation. It must distinguish a valid reproduction from an unsupported environment, a material blocker, a non-reproduction after two independent investigations, and an ABR system failure. The result must be measurable without relying on hidden agent memory.

## Solution

ABR is a TypeScript/Node host with an explicit CLI invocation that creates one bounded Run. The CLI translates GitHub-first inputs into a provider-neutral RunRequest, and a deep Run Orchestrator owns the lifecycle from snapshot through terminal disposition.

The Run Orchestrator derives a Repository Execution Contract, assesses eligibility against a Linux OCI execution profile, drives independent investigators and verifiers in isolated contexts, emits a sanitized immutable Run Record, and optionally publishes a concise Result Projection back to the originating GitHub Issue. Successful Runs produce a revision-bound Reproduction Bundle; all terminal Runs produce inspectable evidence.

The MVP is GitHub-first but separates Bug Source, Repository Provider, and Result Sink adapters. It uses a versioned Benchmark Case and hidden Evaluation Oracle to measure outcomes and failure modes without exposing ground truth to ABR actors.

## User Stories

1. As a developer, I want to explicitly invoke ABR for a GitHub Issue and GitHub repository, so that investigation starts only when requested.
2. As a developer, I want the bug report available to a Run to be snapshotted immutably, so that later Issue edits cannot change an active investigation.
3. As a developer, I want ABR to resolve a repository reference to an exact commit SHA, so that evidence is tied to a known code revision.
4. As a developer, I want an omitted interactive revision to resolve to the default-branch HEAD at Run start, so that the target is convenient but still immutable.
5. As an evaluator, I want benchmark cases to require an explicit immutable revision, so that benchmark outcomes are comparable.
6. As a developer, I want ABR to assess whether the case fits its supported execution profile, so that unsupported cases are not reported as failed reproductions.
7. As a developer, I want ABR to inspect repository-local documentation, configuration, scripts, lockfiles, and CI evidence, so that normal setup and test workflows are discovered where derivable.
8. As a developer, I want missing external prerequisites surfaced honestly, so that ABR never fabricates credentials, data, or infrastructure.
9. As an evaluator, I want every Run to use finite, recorded investigation constraints, so that no investigation is unbounded or opaque.
10. As a developer, I want material hypotheses, experiments, and observations recorded, so that the investigation is evidence-driven rather than a one-shot generation.
11. As a developer, I want ABR to prefer a failing automated test when practical, so that the result integrates naturally with the Target Repository workflow.
12. As a developer, I want an executable scenario when a normal test is inappropriate, so that supported non-test behavior can still be reproduced and verified.
13. As a developer, I want ABR to reject human-only instructions as a successful result, so that a reproduction is independently executable.
14. As a maintainer, I want a successful proof to run against untouched production code at the resolved revision, so that ABR cannot manufacture the reported failure.
15. As a developer, I want every Candidate Package independently verified, so that an investigator cannot self-certify a reproduction.
16. As a developer, I want verification to test semantic correspondence with the bug report, so that an unrelated failing assertion is rejected.
17. As a developer, I want a second independent investigation after a primary exhaustion, so that NOT_REPRODUCED is based on more than one reasoning path.
18. As a developer, I want NOT_REPRODUCED described as a bounded investigation result, so that it is never interpreted as proof that the bug does not exist.
19. As a developer, I want INCONCLUSIVE to name a material blocker and the attempts made, so that difficult cases do not use it as a generic escape hatch.
20. As a developer, I want ABR to request human or source information only after reasonable inference and bounded experimentation, so that escalation is minimal and actionable.
21. As a developer, I want new external information to create a fresh Run, so that prior conclusions and inputs are never silently resumed or mutated.
22. As an evaluator, I want every terminal Run to emit a self-contained structured Run Record, so that provenance, constraints, evidence, outcome, resources, and artifact references are inspectable.
23. As a developer, I want every REPRODUCED result to include a portable Reproduction Bundle, so that another engineer can apply and execute it outside ABR.
24. As an operator, I want Run Records and artifacts retained through a configurable Store policy, so that deployments may choose ephemeral or persistent evidence without hidden behavior.
25. As an evaluator, I want each Run to start fresh while prior evidence remains explicit and non-authoritative, so that persistence never becomes implicit investigator memory.
26. As a maintainer, I want a compact material trace for every Run and optional richer audit retention, so that inspection is useful without retaining irrelevant raw activity by default.
27. As an operator, I want sanitization before persistence and publication, so that credentials and sensitive data are not intentionally exposed.
28. As a developer, I want optional concise GitHub Issue write-back, so that terminal results or precise information requests reach the report when enabled.
29. As a developer, I want reproduction success to be independent of labels, assignments, Issue state changes, or pull requests, so that ABR does not manage the Issue lifecycle.
30. As a developer, I want a confidence assessment with its evidence basis, so that confidence remains advisory and never replaces verification.
31. As an evaluator, I want Benchmark Case input separated from a hidden Evaluation Oracle, so that investigators and verifiers cannot see ground truth.
32. As an evaluator, I want benchmark coverage that includes outcomes other than REPRODUCED, so that negative, blocked, and unsupported behavior is measurable.
33. As an evaluator, I want product-level false positives measured separately from verifier-rejected candidates, so that normal candidate rejection is not misreported as a final failure.
34. As an evaluator, I want setup, validity, false positives and misses, escalation, second-investigator behavior, resource use, latency, and failure modes reported separately, so that performance is not collapsed into a single success rate.
35. As a maintainer, I want the MVP to meet these behaviors without repair, repair pull requests, production debugging, generalized issue triage, or a generic agent platform, so that the product remains reproduction-focused.

## Implementation Decisions

### Invocation and identity

- The MVP invocation is a synchronous `abr run` CLI that accepts separate GitHub Issue and repository references, an optional revision, a required configuration profile, an optional opaque request key, publication mode, and text or JSON output mode.
- The CLI adapts inputs to a provider-neutral RunRequest. GitHub Action, API, and UI surfaces may create the same RunRequest and consume the same RunResult later without depending on the CLI.
- A request key is idempotent only for the same canonical request content. Reusing it with different content is validation failure; omitting it intentionally starts a new Run.
- A valid terminal domain result returns process success. Invalid invocation or inability to produce a well-formed result returns process failure. Detailed process exit codes remain an implementation concern.

### Primary seam and modules

- Run Orchestrator is the single high-level module and test seam. Its interface accepts a RunRequest and returns one RunResult containing the terminal disposition, resolved revision, Run Record or configured reference, artifact references, and publication status.
- The Run Orchestrator owns immutable snapshotting, lifecycle transitions, eligibility, policy enforcement, coordination, terminalization, and Result Projection sequencing. Callers do not coordinate investigators or interpret partial state.
- The orchestrator receives adapters for Bug Source, Repository Provider, Execution Environment, Investigator, Verifier, Run/Artifact Store, Sanitizer, Result Sink, clock, and identifiers. These adapters vary at real integration seams while the lifecycle interface stays small.
- The first concrete adapters are GitHub Issue Bug Source, GitHub Repository Provider, and GitHub Issue Result Sink. Their logical responsibilities remain separate even when a run uses one GitHub repository and Issue.

### GitHub-first integration

- Bug Source snapshots Issue title, body, paginated comments, identity, URLs, timestamps, references, and available attachment metadata at Run start. Later remote changes never mutate the Run.
- Repository Provider resolves an explicit ref, or the default-branch HEAD, to one commit SHA once and checks out only that revision.
- GitHub credentials are ephemeral references to a fine-grained `GITHUB_TOKEN` initially. Repository metadata and contents are read-only; Issue write permission is required only when publication is enabled. A GitHub App token provider may replace this mechanism without changing adapter contracts.
- Result Sink publishes only a sanitized terminal Result Projection. It uses a stable marker containing Run identity and projection digest to update or create exactly one comment per Run, never duplicates comments, and never changes Issue lifecycle state.
- Transient remote errors and rate limits use bounded retries with recorded backoff. Missing access, inaccessible source material, or unresolved revisions are material blockers after reasonable attempts; an adapter defect is a SYSTEM_ERROR.

### Execution profile and eligibility

- The supported MVP profile is a fresh, non-privileged Linux x86_64 OCI container for every actor. It has a dedicated ephemeral workspace, read-only inputs, finite resource quotas, and no arbitrary host mounts, Docker socket, device access, root privilege, or direct production access.
- Headless browser automation is supported when provided in a controlled image and driven programmatically. Desktop, mobile, hardware, manual interaction, non-Linux operating systems, host privilege, and unapproved external network capability are UNSUPPORTED.
- Provisioning may use egress only to policy-authorized hosts and registries. Investigation and verification deny external egress except explicit local services provisioned by the Run.
- Secrets are explicit deployment capabilities with minimal scope and Run lifetime. They are never supplied as CLI values, stored in the Run Record, retained in artifacts, or published.
- SUPPORTED means a complete non-interactive, isolated, and verifiable plan exists. UNSUPPORTED means a required capability is outside the profile. INCONCLUSIVE means a material prerequisite within the profile remains unavailable after reasonable attempts. ABR inability to enforce or attest its own environment is SYSTEM_ERROR.

### Lifecycle and independent actors

- A Run starts only after invocation validation, then progresses through snapshotting, eligibility, primary investigation, primary verification when candidates exist, secondary investigation and verification when required, and one immutable Terminal Disposition.
- Terminal Disposition is exactly UNSUPPORTED, REPRODUCED, NOT_REPRODUCED, INCONCLUSIVE, or SYSTEM_ERROR. Termination reason is recorded separately from disposition.
- A primary Investigator, Verifier, and secondary Investigator have distinct sessions, workspaces, and capability tokens. They receive fresh immutable inputs and cannot read one another's prompts, transcripts, workspaces, hypotheses, self-assessments, or verdicts.
- A Candidate Package moves to independent verification without allowing the submitting Investigator to self-certify it. If no primary candidate is accepted and no material blocker exists, a fresh secondary Investigator receives the original inputs and policy without primary-investigation memory.
- REPRODUCED requires a verifier acceptance. NOT_REPRODUCED requires a functioning environment, two independent substantive investigations, no unresolved material blocker, and no accepted candidate. INCONCLUSIVE requires a named material blocker and attempted alternatives. SYSTEM_ERROR represents ABR platform failure, not Target Repository setup evidence.

### Investigation policy

- Every Run snapshots a versioned InvestigationPolicy with hard limits for wall time, model calls, tool actions, experiments, and sandbox resources. Token and monetary limits apply when provider telemetry is available; unavailable values are recorded as unknown.
- Numeric profiles are explicit configuration and are calibrated through evaluation rather than embedded as hidden defaults.
- Investigators maintain material facts, unknowns, hypotheses, experiments, expected discriminating observables, results, and stop justification. Repeated or non-discriminating activity does not count as substantive investigation.
- Human escalation is permitted only for the smallest material fact or capability that cannot be inferred or safely explored. New information always starts a fresh Run.

### Reproduction Bundle and verification

- A Reproduction Bundle is a versioned portable directory, optionally archived for transport, containing a manifest, human-readable instructions, inspectable patch and added files, and a structured observable record. Archive format or pull request publication is not canonical.
- Its manifest binds the bundle to the target commit and base digest, declares allowed changed paths and roles, setup, non-secret commands, stimulus, expected observable, stability requirements, and links to Bug Report Snapshot and Evidence Item.
- Automated tests are preferred. An executable scenario is permitted only when the manifest explains why a conventional test is not natural and it still provides programmatic setup, stimulus, and observable.
- Verification uses a fresh checkout, validates bundle integrity and patch applicability, confirms production-path integrity, runs the declared commands under the execution profile, performs policy-required reruns, and evaluates semantic correspondence to the Bug Report Snapshot.
- Verification Records explicitly accept or reject with integrity, untouched-production, provisioning, execution, stability, or semantic-mismatch evidence. A known fixed revision is strengthening evidence when available, not a requirement.

### Run Record, evidence, security, and retention

- The canonical Run Record is a versioned JSON document assembled from an append-only material event journal and finalized immutably. It contains input provenance, effective configuration, eligibility and execution evidence, trace, Candidate Packages and Verification Records, terminal decision, confidence basis, resources, and Artifact References.
- Evidence Item is attributable, time-stamped, typed, sensitivity-classified, and linked to the decision it supports. The record captures material evidence and decisions, not private chain of thought or unfiltered tool logs.
- Artifact References carry role, content digest, media type, size, sensitivity, and optional Store locator. Unknown provenance or cost remains explicit rather than inferred.
- Sanitization is mandatory before any persistence or Result Projection. A versioned SanitizationPolicy redacts prohibited structured fields, supplied secret values, and configured detector matches. Unsanitized content exists only transiently inside actor sandboxes.
- The first persistent Store profile is local SQLite plus filesystem. A configured SQLite database retains sanitized Run Records, Artifact References, trace, and retention metadata; a configured local artifact root retains immutable Reproduction Bundles and artifact bytes by digest. The ephemeral profile uses in-memory SQLite and a temporary artifact root, both discarded after the Run. A future PostgreSQL or object-store adapter satisfies the same Store interface without changing the Run Orchestrator.
- Retention is an explicit per-Run policy with at least ephemeral and persistent profiles for records, artifacts, trace, and optional audit bundles. Expired content may leave a tombstone with digest, policy, and deletion reason.

### Evaluation and performance view

- The initial benchmark suite contains 12 versioned cases: at least six reproducible, two NOT_REPRODUCED, two INCONCLUSIVE, and two UNSUPPORTED. Real public Issue/repository cases are primary; controlled fixtures complete reliable coverage for difficult categories.
- Benchmark Case contains only ABR-visible immutable input: report snapshot, target commit, permitted execution profile and capabilities, policy, and input digests.
- Evaluation Oracle is stored and accessed separately from Benchmark Case and is unavailable to Investigators, Verifiers, Result Sink, and pre-judgment Run Record processing. It contains expected eligibility or disposition when known, independent evidence, expected observables, known fixes or reference reproductions, and adjudication notes.
- Post-run evaluation produces a separate Evaluation Record. It checks input identity, judges terminal evidence against the hidden oracle, distinguishes verifier rejection from product-level false positive, and marks insufficient-oracle cases as unjudgeable rather than inventing a metric.
- An ABR-specific evaluation command produces a versioned JSON Evaluation Report and static HTML view from persisted Run Records and Evaluation Records. It reports separate dimensions and cohort filters, not a generic experiment-management platform.
- Replay creates a new Run from a Replay Descriptor that records case version, input digest, policy, environment and provisioning digests, ABR and model identifiers, available seeds, and secret-free capability references. It supports comparability but does not promise deterministic model behavior or feed prior conclusions back to investigators.

## Testing Decisions

- The principal behavioral test seam is Run Orchestrator executing a RunRequest and returning a RunResult. Tests cross this interface with controlled adapters and assert observable lifecycle, terminal disposition, Run Record, artifact, and publication behavior rather than internal implementation steps.
- No pre-existing implementation or test suite exists. The first test architecture must establish fixtures at the Run Orchestrator seam and avoid tests coupled to a concrete agent framework, storage technology, or GitHub client implementation.
- Lifecycle tests cover immutable snapshotting, default revision resolution, each eligibility classification, policy exhaustion, material blocker escalation, independent primary and secondary investigation, verifier acceptance and rejection, terminal immutability, and SYSTEM_ERROR separation.
- Reproduction tests cover automated-test preference, executable-scenario fallback, fresh verifier execution, protected production code, stability policy, semantic mismatch rejection, bundle portability, and absence of human-only success.
- Adapter contract tests cover GitHub pagination and snapshot identity, commit resolution, least-privilege authentication behavior, rate-limit retry, Result Sink idempotency, sanitization before storage or publication, SQLite persistence and retention, and artifact digests.
- Execution Environment tests cover isolation requirements, denied capabilities, controlled provisioning egress, browser availability, resource-limit recording, secret non-retention, and supported versus unsupported classification.
- Evaluation tests use intentionally separated Benchmark Case and Evaluation Oracle fixtures. They cover oracle non-disclosure, all four product classifications plus SYSTEM_ERROR reporting, false-positive distinction, replay comparability, cohort grouping, and all required metric dimensions.
- End-to-end tests use small controlled Target Repositories and container images. They assert only externally visible RunResult, Reproduction Bundle, Run Record, Evaluation Record, and sanitized Result Projection behavior.

## Out of Scope

- Automatic bug repair, repair patch generation, repair pull requests, and merging changes.
- General-purpose coding-agent, issue-triage, connector, model-routing, tracing, evaluation-lab, or development-OS platforms.
- Autonomous production debugging, direct production access, hidden persistent agent memory, resumable conversational investigations, and automatic historical revision inference.
- Automatic Issue labeling, assignment, closing, prioritization, or selection.
- Universal operating-system, browser, desktop, mobile, hardware, physical-device, manual-interaction, proprietary-infrastructure, or arbitrary-network support.
- Automatic pull-request publication of Reproduction Bundles.
- Numeric product success, latency, cost, or deterministic-replay guarantees beyond configured policy and measured reporting.

## Further Notes

- This specification synthesizes the decisions in the Wayfinder map "Wayfinder: SPEC MVP completo da PRD" and preserves the MVP acceptance criteria in the PRD.
- The project remains pre-implementation. This specification defines behavioral contracts and testing seams; implementation tickets should split the work along dependency edges without extracting generalized infrastructure before ABR demonstrates reuse.
- `CONTEXT.md` is the canonical vocabulary for Run, Candidate Package, Verification Record, Terminal Disposition, Run Record, Evidence Item, integration roles, Reproduction Bundle, Benchmark Case, and Evaluation Oracle.
