# Autonomous Bug Reproducer (ABR) — Product Requirements Document

**Status:** Product definition complete; ready for architecture/specification

**Product stage:** Pre-implementation

**Primary source of truth:** current repository README, refined by product discovery decisions captured in this PRD

---

## 1. Product / Problem Statement

Software bug reports are often incomplete, ambiguous, or disconnected from the exact executable conditions that trigger the reported behavior. Before a bug can be repaired confidently, a developer usually needs to understand the repository, determine how to build and test it, identify relevant code paths, infer missing reproduction conditions, run experiments, reject misleading failures, and reduce the result to something that can be executed again.

ABR — Autonomous Bug Reproducer — automates that investigation.

Given a software repository and a bug report, ABR attempts to produce an independently executable and verifiable reproduction of the reported bug against a specific repository revision. The preferred output is a failing automated test; when a conventional test is not an appropriate representation, ABR may instead produce an executable and verifiable reproduction scenario.

ABR is intentionally focused on **reproduction, not repair**.

The product must be measurable, inspectable, and evaluation-first. Its purpose is not to demonstrate an impressive but opaque coding agent. It must support repeatable benchmarking, failure-mode analysis, cost and latency inspection, and later comparison of models, prompts, policies, and investigation strategies.

The core product loop is:

> **one bug + one repository → investigation → reproduction attempt → verifiable outcome**

---

## 2. Target User / Use Cases

### Primary user

A software developer or repository maintainer who has a concrete bug report and wants a verified reproduction before attempting or reviewing a repair.

### Secondary user

An ABR maintainer, AI/software-engineering researcher, or evaluator who wants to run ABR across controlled benchmark cases and understand how reproduction performance changes across models, policies, versions, costs, and failure modes.

### Core use cases

1. A developer explicitly invokes ABR on a GitHub Issue and repository and receives a verified failing test or executable scenario.
2. ABR investigates a valid case but cannot reproduce the behavior within its declared constraints and reports `NOT_REPRODUCED` with inspectable evidence from two independent investigations.
3. ABR discovers that a case cannot be conclusively investigated because a material prerequisite or piece of information is unavailable and reports `INCONCLUSIVE`, optionally requesting the smallest necessary piece of information through the configured result integration.
4. ABR determines before investigation that the case requires execution capabilities outside the supported MVP execution envelope and reports it as unsupported rather than counting it as a failed reproduction.
5. ABR is executed repeatedly on benchmark cases while retaining structured run records and artifacts for evaluation, audit, and performance analysis.

---

## 3. Product Thesis

Bug reproduction is a narrow, externally verifiable vertical for autonomous software-engineering agents.

Instead of asking an agent to fix an issue and judging a patch subjectively, ABR asks a stricter question:

> **Can the agent make the reported bug fail on demand, for the expected reason, against the declared repository revision?**

A generated failing test is not sufficient by itself. The observed failure must correspond semantically to the reported behavior and must not have been introduced by ABR's own modifications to production behavior.

ABR should therefore be judged as an engineering system through reproducible outcomes, evidence, false-positive analysis, failure-mode analysis, and resource usage — not through cherry-picked demos.

---

## 4. Goals

The MVP must:

- turn a bug report and repository revision into a bounded autonomous reproduction investigation;
- discover normal build/test workflows when they are reasonably derivable from repository-local information;
- execute experiments in an isolated, programmatically controllable environment;
- prefer a failing automated test when that is a natural representation of the bug;
- support an independently executable and verifiable scenario when a conventional test is not appropriate;
- distinguish successful reproduction from unrelated, artificial, or setup-induced failures;
- require explicit independent verification before declaring `REPRODUCED`;
- require a second independent investigation before declaring `NOT_REPRODUCED` when the first investigator exhausts its budget without a blocker;
- use human requests for missing information only as a last resort;
- produce structured evidence and provenance for every terminal run;
- support configurable persistence of run records and artifacts without relying on hidden persistent agent state;
- provide GitHub Issues as the first bug source and result sink, while preserving generic logical boundaries for future integrations;
- support repeatable benchmark execution with evaluation ground truth kept separate from the information visible to ABR;
- expose ABR-specific aggregate performance and resource-usage views from persisted run records.

---

## 5. Non-Goals

The MVP does **not** aim to:

- fix bugs automatically;
- generate or merge repair pull requests;
- become a general-purpose coding agent;
- claim a root cause unless evidence independently supports one;
- make root-cause analysis a success requirement;
- monitor all new issues autonomously;
- decide which issues deserve investigation;
- manage the broader GitHub issue lifecycle through assignment, labeling, closing, prioritization, or triage;
- make automatic PR creation part of reproduction success;
- guarantee universal support for every language, framework, browser, desktop environment, mobile device, physical device, hardware dependency, or external infrastructure class;
- autonomously debug production systems or require direct production access;
- accept human-only instructions as sufficient evidence for `REPRODUCED`;
- automatically infer the historical repository version on which a bug was originally observed;
- implement resumable conversational investigations that suspend and later continue with new human input;
- automatically feed previous run conclusions into future investigators;
- build a general connector framework;
- build a general-purpose AI Flight Recorder, Evaluation Lab, model router, or Agentic Development OS in the MVP.

---

## 6. MVP Scope

### 6.1 Invocation

MVP runs are explicitly initiated. ABR does not autonomously select or monitor issues.

The concrete invocation mechanism — CLI, GitHub Action, API, UI, or another trigger — is a specification decision, not a product requirement.

### 6.2 First-class integrations

The MVP is **GitHub-first but not GitHub-bound**.

It must implement GitHub as the first concrete provider for three logically separate capabilities:

- **Bug Source:** GitHub Issues;
- **Repository Provider:** GitHub repositories;
- **Result Sink:** GitHub Issue comments.

These are product-level responsibility boundaries, not mandated software interface names. Future sources/providers/sinks must be possible without redefining ABR's core reproduction semantics.

Result publication is configurable and may be disabled.

### 6.3 Execution envelope

The MVP targets repositories whose relevant behavior can be exercised in an isolated, programmatically controllable software environment.

The MVP may use repository-provided test, service, browser, or integration infrastructure when it is available or reasonably provisionable.

Universal support for arbitrary browsers, mobile devices, physical hardware, production systems, proprietary external infrastructure, or manual-only interaction is outside the MVP commitment.

Cases requiring unsupported execution capabilities must be classified before or at the boundary of investigation as **unsupported**, rather than being counted as failed reproduction attempts.

---

## 7. User Stories / Core Scenarios

### 7.1 Successful automated-test reproduction

As a developer, I can invoke ABR on a GitHub Issue and repository so that ABR investigates the issue, creates a failing test and any necessary fixtures/support files, verifies the test independently against the original production code, and returns `REPRODUCED` with evidence and execution instructions.

### 7.2 Successful executable-scenario reproduction

As a developer, I can receive an executable reproduction scenario when the reported behavior cannot reasonably be represented in the repository's normal test infrastructure, provided ABR actually executes and independently verifies that scenario.

### 7.3 Meaningful non-reproduction

As a developer, if the primary investigator exhausts its declared investigation constraints without reproducing the bug, a second independent investigator must attempt the same problem from a fresh investigation perspective before ABR may return `NOT_REPRODUCED`.

### 7.4 Blocking missing information

As a developer, if ABR cannot proceed because a materially necessary fact or prerequisite is unavailable, ABR should first exhaust reasonable inference and bounded experimentation. Only if the missing information cannot be derived or safely explored may ABR return `INCONCLUSIVE` and, if configured, publish a precise information request to the originating GitHub Issue.

### 7.5 Unsupported environment

As an evaluator, I can distinguish cases ABR failed to investigate from cases outside the declared execution envelope, so unsupported cases do not contaminate reproduction performance metrics.

### 7.6 Evaluation and audit

As an ABR maintainer or evaluator, I can inspect persisted run records and artifacts, aggregate performance across runs, compare versions/configurations, inspect costs and failure modes, and reconstruct the evidence behind an outcome without relying on hidden agent memory.

---

## 8. Inputs

Each run operates on an immutable snapshot of externally supplied inputs.

### 8.1 Required logical inputs

A run requires:

- a repository target;
- a bug report;
- an execution context sufficient to attempt the case.

### 8.2 Repository target

A user may provide an immutable revision or a mutable reference such as a branch or tag.

Every run must resolve the target once, at run start, to an **exact immutable repository revision** and record that resolved revision.

For interactive GitHub usage, when no revision is supplied, ABR defaults to the current HEAD of the repository's default branch and must report the exact resolved revision.

For benchmark cases, an immutable revision is mandatory. A benchmark case without one is invalid.

ABR does not automatically infer the historical revision likely associated with the original issue report.

### 8.3 Bug report snapshot

The bug report available at run start is snapshotted for the run. For GitHub Issues, this may include the issue title, description, comments, relevant attachments/references, source identity, and other available context.

New external information that materially changes the bug report does not silently mutate an active run. It becomes input to a new fresh run.

### 8.4 Setup and execution information

ABR is expected to discover normal setup, build, and test procedures when they are reasonably derivable from repository-local sources such as documentation, scripts, build configuration, test configuration, or CI configuration.

External prerequisites that cannot reasonably be inferred must be supplied by the execution environment or configuration.

Missing material prerequisites may lead to `INCONCLUSIVE`; they must not be fabricated or silently assumed.

### 8.5 Secrets and sensitive inputs

Secrets and sensitive data may be supplied explicitly by the execution environment when required for an eligible reproduction case.

They are execution capabilities, not persistent ABR memory.

ABR must not intentionally copy credentials into persisted artifacts or external result publications.

Direct autonomous interaction with production systems is outside the MVP execution envelope. Authorized production-derived logs, dumps, or evidence may be supplied as inputs.

---

## 9. Agent Responsibilities

ABR is responsible for the following product behaviors, independent of the eventual agent/runtime architecture.

### 9.1 Repository exploration

ABR must inspect the repository sufficiently to understand its relevant structure, test strategy, execution workflow, and likely code paths.

### 9.2 Evidence collection

ABR must collect evidence that informs the reproduction investigation and must distinguish observed facts from hypotheses or missing information.

### 9.3 Hypothesis-driven investigation

ABR must generate and revise explicit material hypotheses about the reported failure, design bounded experiments, and update or discard hypotheses based on observed results.

### 9.4 Reasonable inference before escalation

ABR should test reasonable interpretations and common-sense conditions when they can be explored safely and cheaply.

It must not request clarification merely because the issue omits a detail that can reasonably be inferred from repository evidence or tested empirically.

Human feedback is an escalation of last resort.

### 9.5 Reproduction construction

ABR should first attempt to represent the bug as an automated test when that is natural and practical.

When a conventional test is not appropriate, ABR may construct an executable scenario that programmatically establishes setup, applies the relevant stimulus, and exposes a verifiable observable.

### 9.6 Temporary investigation changes vs final proof

During investigation ABR may modify its isolated workspace as needed for diagnostics or experimentation.

However, a final `REPRODUCED` proof must execute against the **original production/application code of the declared target revision**.

The final proof may add tests, fixtures, support scripts, reproduction-specific configuration, mocks/stubs, or other support material necessary to exercise that code, but it must not modify production behavior in a way that creates the failure being claimed.

### 9.7 Independent verification

An investigator's reproduction candidate cannot directly become `REPRODUCED` solely because the same investigator believes it is correct.

Every reproduction candidate must pass an explicit **independent verification step** that challenges at least:

- whether the reproduction executes against the declared untouched production revision;
- whether it behaves reliably enough to be a useful reproduction;
- whether the observed failure corresponds semantically to the reported behavior;
- whether the failure is artificial, incidental, setup-induced, or otherwise unrelated.

The PRD does not prescribe whether independence is implemented through separate agents, contexts, models, deterministic checks, or combinations of these mechanisms.

### 9.8 Second investigation before `NOT_REPRODUCED`

A single unsuccessful investigation is insufficient for `NOT_REPRODUCED`.

If the primary investigator exhausts its investigation constraints without obtaining a valid reproduction and without encountering a legitimate blocker, ABR must launch a second independent investigation from a fresh investigation perspective.

`NOT_REPRODUCED` may be returned only when both investigations have completed meaningful attempts within their declared constraints without reproducing the reported behavior.

The second investigator should not merely continue the first investigator's reasoning path. The exact technical isolation mechanism is deferred.

---

## 10. Outputs and Artifacts

### 10.1 Structured Run Record

Every terminal run must produce a structured Run Record, regardless of whether that record is ultimately persisted.

The Run Record must contain enough information to understand and independently assess the outcome without hidden ABR state.

At minimum it must represent:

- repository identity and exact resolved revision;
- bug report snapshot/reference and input provenance;
- effective execution configuration and declared constraints;
- ABR version and relevant investigator/verifier/model/policy identifiers where available;
- eligibility result;
- material hypotheses and experiments;
- material observations/evidence;
- reproduction candidates and verification decisions when applicable;
- second-investigator activity when applicable;
- terminal outcome and termination reason;
- confidence assessment and its basis;
- resource usage, elapsed time, and model/tool cost where determinable;
- references to reproduction artifacts and execution commands where applicable.

Unavailable provenance or cost fields must be represented as unavailable/unknown rather than invented or silently defaulted.

### 10.2 Reproduction Artifact

The PRD intentionally does not prescribe whether a reproduction artifact is a patch, directory, archive, test file set, script bundle, or another technical format.

A successful reproduction artifact must be:

- **revision-bound:** tied unambiguously to the exact target revision;
- **portable:** transferable outside the ABR execution context;
- **independently executable:** usable by a verifier or developer without hidden run state;
- **self-describing:** sufficient to identify setup, execution command(s), and expected observable;
- **non-invasive:** it must not alter production behavior to manufacture the claimed bug;
- **inspectable:** its changes and purpose must be reviewable;
- **verifiable:** it must contain or reference enough material for independent re-execution.

### 10.3 Reproduction quality and minimality

ABR should reduce a successful reproduction to the conditions and artifacts reasonably necessary to manifest the reported behavior.

Minimality is a quality objective, not a proof of globally minimal size. ABR must not sacrifice validity, stability, clarity, or verifiability in pursuit of fewer lines or files.

### 10.4 Persistence vs publication

ABR separates canonical run/artifact persistence from external result publication.

A configurable **Run/Artifact Store capability** may retain Run Records, reproduction artifacts, evidence, traces, and optional audit data.

The product does not mandate the storage technology.

Deployments may use ephemeral or persistent storage policies.

A **Result Sink** publishes selected projections of the run outcome. For the MVP, GitHub Issue comments are the first Result Sink implementation.

GitHub comments and future PRs are not the canonical persistence mechanism for the full Run Record.

### 10.5 Pull requests

ABR may later support publishing reproduction artifacts as a pull request, but automatic PR creation is not required by the MVP and is not part of the definition of successful reproduction.

A reproduction artifact may be valid even when it is not contribution-ready or appropriate for permanent inclusion in the repository's test suite.

---

## 11. Outcome Semantics

Outcome semantics are product requirements, not agent self-assessments.

### 11.1 Eligibility: `UNSUPPORTED`

`UNSUPPORTED` is not a reproduction outcome. It is an eligibility result indicating that the case requires execution capabilities outside the declared supported execution envelope.

Unsupported cases must be measurable separately and must not be counted as reproduction failures.

### 11.2 `REPRODUCED`

ABR may return `REPRODUCED` only when:

- the target repository revision is fixed and known;
- the final reproduction executes against the untouched production/application code of that revision;
- the reproduction is independently executable and sufficiently stable;
- concrete observables support that the reproduced behavior corresponds semantically to the reported bug;
- an independent verification step accepts the reproduction;
- the observed failure is not merely a build/setup/environment failure, unrelated assertion failure, intentionally introduced failure, or other incidental failure.

A generic failing test is insufficient.

### 11.3 `NOT_REPRODUCED`

`NOT_REPRODUCED` means:

> ABR established a working investigation environment, had a sufficiently meaningful reproduction target, performed two independent and substantive investigations within declared constraints, and did not observe a valid reproduction of the reported behavior.

It does **not** mean that the bug does not exist.

The outcome is always relative to the run's declared investigation constraints and supported environment.

### 11.4 `INCONCLUSIVE`

`INCONCLUSIVE` means ABR could not reach a trustworthy reproduction conclusion because a material blocker prevented a valid investigation or interpretation.

Examples include unavailable required data, dependency, permission, external service, environment capability, or information that cannot reasonably be derived or explored.

`INCONCLUSIVE` must not become a generic escape hatch for difficult cases.

ABR must first attempt reasonable repository discovery, inference, and bounded experiments.

When the blocker is missing human/source information and write-back is enabled, ABR may publish a precise information request. The current run terminates as `INCONCLUSIVE`; newly supplied information becomes input to a future fresh run.

### 11.5 Internal/system failure

Failures of ABR itself that prevent a reliable product outcome must remain distinguishable from `NOT_REPRODUCED` and from a valid reproduction result. The exact system-error taxonomy is deferred to specification.

---

## 12. Reproduction Validity

A valid reproduction establishes a relationship between:

1. the conditions represented by the bug report;
2. the untouched target production code;
3. the reproduction stimulus/setup;
4. a concrete observable behavior;
5. evidence that the observable corresponds to the reported failure.

Exact textual matching of an exception message, stack trace, or other output is not required unless the issue itself makes that exact observable material.

Semantic correspondence must be supported by concrete observables, which may include, depending on the bug class:

- exception or crash behavior;
- process exit status;
- HTTP status/response;
- output value or state;
- persisted-state inspection;
- UI state available through programmatic automation;
- timeout, deadlock, or concurrency observable;
- another deterministic or sufficiently stable signal appropriate to the case.

A bug report that is so ambiguous that ABR cannot establish what successful reproduction would mean may lead to `INCONCLUSIVE`, but only after reasonable interpretations and bounded tests have been attempted.

---

## 13. Evidence and Inspectability Requirements

### 13.1 Fresh execution, persistent evidence

Each run starts from an explicit fresh execution context and must not depend on hidden persistent agent memory from previous runs.

ABR may persist structured evidence, artifacts, and run records across time for evaluation, audit, comparison, and dashboarding.

Persisted information is not implicitly reintroduced into future investigators. Prior-run-informed investigation may be introduced later only as an explicit, observable policy.

### 13.2 Minimum structured trace

Every run must produce a compact structured decision/execution trace sufficient for evaluation, reproducibility, and failure analysis.

It should capture material:

- hypotheses;
- experiments;
- tool/runtime events that affected the decision;
- observations;
- verifier decisions;
- retries or blockers;
- termination decisions;
- resource usage and provenance.

ABR does not require storage or exposure of private model chain-of-thought or exhaustive low-level logs of every irrelevant action.

### 13.3 Full audit retention profile

A deployment may optionally retain a richer audit bundle containing additional raw execution information, intermediate artifacts, detailed stdout/stderr, model request/response metadata where permitted, environment metadata, and more complete execution traces.

The exact audit-bundle format and retention technology are deferred.

### 13.4 Sensitive-data boundary

Before persistent storage or external publication, ABR must apply an appropriate sanitization/redaction boundary so credentials and other sensitive information are not intentionally retained or exposed.

External Result Sinks should expose only the information necessary to communicate the outcome and actionable evidence.

### 13.5 Regulatory posture

ABR should support audit-oriented provenance and retention patterns that are useful in stricter governance environments, but the MVP does not claim legal or AI-regulatory compliance merely because it records traces and artifacts.

---

## 14. Confidence

Every terminal reproduction outcome should include an evidence-backed confidence assessment describing the strength of the outcome claim.

Confidence is advisory metadata.

It must not:

- replace required evidence;
- replace independent verification;
- override outcome eligibility rules;
- turn an unverified candidate into `REPRODUCED`;
- be interpreted as the probability that the bug exists or does not exist.

For `NOT_REPRODUCED`, confidence describes the strength/adequacy of the investigation within declared constraints, not confidence in the absence of the bug.

For `INCONCLUSIVE`, confidence may describe how strongly the evidence supports the identified blocker.

The MVP PRD does not mandate a numeric scale, categorical scale, formula, threshold, or calibration method.

---

## 15. Investigation Budget and Termination

Every ABR run must operate under explicit, finite, inspectable investigation constraints.

The PRD does not prescribe whether those constraints are defined by time, cost, model/tool calls, experiment count, resource limits, or a combination.

A run must not investigate indefinitely.

The Run Record must identify the effective constraints and the reason investigation terminated.

An investigator may stop before consuming every nominal resource if it can justify that reasonable investigation paths have been exhausted within the policy, but "the agent ran out of ideas" is not by itself sufficient evidence for `NOT_REPRODUCED`.

The exact budget dimensions, defaults, and enforcement mechanisms must be determined empirically during specification and evaluation.

---

## 16. GitHub Result Publication

GitHub Issues are the first MVP Result Sink.

When enabled, ABR may publish concise terminal outcome information or a necessary information request.

A result publication should identify at least the resolved target revision and the terminal outcome, and should provide enough concise evidence/actionability to be useful without dumping the complete internal trace.

For example, publication may include:

- outcome;
- target revision;
- concise reproduction/investigation summary;
- execution command or artifact reference when appropriate;
- confidence summary;
- precise missing-information request for `INCONCLUSIVE` when human input is genuinely necessary.

ABR does not need to label, assign, close, reopen, or otherwise manage the issue lifecycle in the MVP.

---

## 17. Evaluation Principles

Evaluation is a first-class product requirement.

### 17.1 Independent oracle

A benchmark case is credible only when sufficient independent ground truth exists to judge ABR's semantic outcome.

The benchmark must separate:

- **case input visible to ABR**, from
- **evaluation oracle/ground truth hidden from ABR**.

Case input may contain:

- bug report snapshot;
- repository;
- exact target revision;
- permitted execution context/configuration.

Ground truth may include, when available:

- whether the case is genuinely reproducible at that revision;
- the relevant expected observable;
- a known fixed revision or fix patch;
- a known reproduction/reference test;
- adjudication notes for semantic ambiguity.

Not every case must have every form of oracle, but each benchmark case must have enough independent evidence to judge ABR reliably.

Known fixes, reference reproductions, and other oracle data must not be exposed to investigators or verifier unless they were legitimately part of the original bug source.

### 17.2 Positive, negative, ambiguous, and blocked cases

The benchmark must not contain only easy, known-reproducible bugs.

It must eventually exercise behavior where the correct result may be:

- `REPRODUCED`;
- `NOT_REPRODUCED`;
- `INCONCLUSIVE`;
- unsupported eligibility.

It should include cases that expose false-positive risk, unnecessary escalation, setup limitations, and difficult-but-reproducible bugs.

The concrete dataset composition is deferred.

### 17.3 Dimensions measured separately

ABR must not collapse evaluation into a single success rate.

The evaluation design must be able to measure separately at least:

- eligibility / supported-case coverage;
- repository setup success;
- ability to perform a substantive investigation;
- valid reproduction rate;
- verifier candidate acceptance/rejection behavior;
- false-positive reproduction rate;
- missed/false-negative reproduction cases where ground truth supports that judgment;
- `INCONCLUSIVE` reasons;
- human-information escalation frequency and quality;
- second-investigator rescue rate;
- reproduction attempts / investigation effort;
- latency;
- model/tool/resource cost where available;
- failure-mode distribution.

No numeric performance targets are asserted in this PRD.

### 17.4 False positives

A reproduction candidate rejected by the verifier is part of normal investigation and is **not** a final false positive.

A product-level false positive occurs when ABR declares `REPRODUCED`, but external benchmark ground truth later shows that the reproduction was invalid because, for example, the failure was:

- artificially introduced by ABR;
- incidental or caused by setup;
- semantically mismatched with the reported bug.

The evaluation layer — not the investigator — is responsible for measuring false positives against ground truth.

Where possible, known buggy and fixed revisions can provide a strong oracle: a reproduction that fails on the buggy revision and passes on the known fixed revision is stronger evidence than a failing test alone. This is not required to be the only oracle mechanism.

### 17.5 Replay and comparability

Runs should be replayable where technically possible so that ABR versions, models, prompts, policies, and strategies can later be compared on the same cases.

Replay does not require complete deterministic replication of every stochastic model behavior, but the run configuration, inputs, target revision, and material execution metadata must be sufficiently recorded to support meaningful comparison.

---

## 18. ABR-Specific Performance View

Persisted Run Records must be aggregatable into an inspectable ABR-specific evaluation/performance view.

The MVP must make it possible to inspect trends or breakdowns across relevant dimensions such as:

- terminal outcome distribution;
- unsupported eligibility;
- setup/execution failures;
- verifier rejection behavior;
- false positives where benchmark ground truth is available;
- second-investigator rescue behavior;
- investigation effort/budget consumption;
- latency;
- cost/resource usage;
- ABR version;
- model/provider identity where available;
- investigation/verifier policy or configuration;
- repository or benchmark cohort when meaningful.

The MVP does not require a specific dashboard technology or a generalized experiment-management system.

ABR must be able to measure itself; a future Evaluation Lab may generalize this capability across multiple agents, benchmark suites, experiments, regression comparisons, and replay orchestration.

---

## 19. Failure Modes

ABR must make failure causes inspectable rather than collapsing them into a single unsuccessful state.

Important product-level failure classes include:

- unsupported execution environment;
- repository/setup failure;
- missing required external prerequisite;
- missing material information that cannot reasonably be inferred or tested;
- access/permission blocker;
- infrastructure/dependency blocker;
- reproduction candidate rejected by verifier;
- investigation budget exhausted by both independent investigators;
- internal ABR/runtime failure;
- false-positive `REPRODUCED` discovered by external evaluation;
- sensitive-data sanitization/publication failure.

The detailed machine-readable taxonomy is deferred to specification and should evolve from observed failure cases rather than being overdesigned upfront.

---

## 20. Constraints

### 20.1 Vertical before infrastructure

ABR must solve bug reproduction before extracting generalized agent runtime, evaluation, tracing, routing, or connector infrastructure.

Shared infrastructure should emerge only from demonstrated reuse needs.

### 20.2 No hidden persistent agent state

A run must be understandable from explicit inputs/configuration and its recorded evidence. Correctness must not depend on undocumented memory accumulated across previous runs.

### 20.3 Reproduction before repair

No MVP feature should weaken the conceptual separation between proving the bug and fixing it.

### 20.4 Inspectability over opaque autonomy

An impressive but uninspectable success is insufficient. Material decisions, evidence, execution constraints, and outcome reasoning must be observable.

### 20.5 Safe persistence/publication

Secrets and sensitive inputs may be used only when explicitly supplied/authorized. Persistence and publication must minimize sensitive-data exposure and must not intentionally retain credentials.

### 20.6 No production-debugging dependency

ABR must be able to satisfy its MVP product contract without autonomous direct access to live production systems.

---

## 21. Assumptions

The PRD currently assumes:

- the first practical integration target is GitHub;
- a meaningful subset of bug reports can be investigated in isolated, automatable execution environments;
- repository-local build/test/CI information is often sufficient for ABR to establish normal execution workflows;
- exact reproduction validity can often be judged through concrete observables and independent verification even when the issue is imperfectly worded;
- persisted Run Records will provide enough data to derive useful evaluation and observability views without being automatically fed back into subsequent investigators;
- benchmark construction can obtain sufficient independent ground truth for at least an initial useful dataset.

These assumptions must be revisited if implementation or benchmark work disproves them.

---

## 22. Explicitly Deferred Decisions

The following decisions are intentionally **not** product requirements and should be resolved during architecture/specification or empirical evaluation:

- agent framework/runtime;
- number and process topology of agents;
- whether investigators and verifier use the same or different model families;
- technical mechanism that ensures investigator/verifier independence;
- first language/framework ecosystem(s) supported by the implementation;
- container/sandbox/execution technology;
- browser automation technology;
- concrete representation of reproduction artifacts;
- patch/archive/directory format and layout;
- structured Run Record schema and serialization format;
- Run/Artifact Store implementation (filesystem, SQLite, relational DB, object storage, combinations, etc.);
- concrete connector/interface names and APIs;
- CLI vs GitHub Action vs API vs UI invocation surface;
- investigation budget dimensions and defaults;
- confidence scale and calibration method;
- detailed trace schema;
- audit-bundle format;
- default retention policy;
- secret/sensitive-data detection and redaction implementation;
- performance-view/dashboard technology;
- benchmark dataset composition;
- benchmark sample size;
- numeric success, cost, or latency targets;
- deterministic replay guarantees;
- automatic PR publication;
- historical-revision inference;
- prior-run-informed investigation policies.

These decisions may materially affect implementation but do not need to be resolved to understand which product ABR is intended to build.

---

## 23. Future Directions / Out of Scope

Natural future directions include:

### Agent Evaluation Lab

Extract benchmark execution, controlled experiment management, replay, regression detection, and model/prompt/policy comparison if those capabilities prove independently reusable.

### AI Flight Recorder

Extract the tracing/provenance model if ABR's structured execution records prove reusable across other agents.

### LLM / Agent Router

Use collected evidence about task type, success, cost, and latency to drive routing only after sufficient empirical data exists.

### Reproduction lifecycle and historical comparison

Support explicit comparison of multiple runs of the same logical bug across repository revisions, without requiring hidden agent memory.

### Prior-run-informed investigation

Allow an explicit policy in which a run is given selected evidence from previous runs, while keeping this distinguishable from fresh benchmark execution.

### Reproduction-to-regression-test workflow

Transform a valid reproduction artifact into a contribution-ready regression test and optionally publish it as a pull request.

### Automated triggers

Allow labels, CI failures, issue events, or other orchestration policies to trigger ABR automatically.

### Richer execution envelopes

Add broader browser, mobile, desktop, distributed-system, or specialized-environment support based on demonstrated need.

### Broader Agentic Development OS

Bug repair and wider software-development orchestration may consume ABR's reproduction output later, but reproduction remains a separately evaluable capability.

---

## 24. Acceptance Criteria for the MVP

The MVP is product-complete when all of the following are true.

1. **Explicit invocation:** A user can explicitly start ABR on a GitHub Issue and GitHub repository.
2. **Immutable bug input:** ABR captures the externally supplied bug-report input as an immutable run snapshot.
3. **Immutable code target:** ABR resolves every run to an exact repository revision and records it.
4. **Reasonable default revision:** In interactive GitHub usage, an omitted revision resolves to the default branch HEAD at run start; benchmark cases require an explicit immutable revision.
5. **Eligibility:** ABR can distinguish cases inside the supported execution envelope from unsupported cases.
6. **Repository discovery:** ABR can inspect repository-local information to determine normal setup/build/test procedures when reasonably available.
7. **Prerequisite honesty:** Missing external prerequisites are surfaced rather than fabricated or silently guessed.
8. **Bounded investigation:** Every run operates under finite, inspectable investigation constraints.
9. **Hypothesis-driven experimentation:** ABR records material hypotheses, experiments, and observations rather than behaving as an opaque one-shot generator.
10. **Automated-test preference:** ABR can produce a failing automated test when that is a natural form of reproduction.
11. **Executable-scenario fallback:** ABR can produce an independently executable and verifiable scenario when a conventional test is not appropriate within the supported execution envelope.
12. **No human-only success:** Human-only reproduction instructions are insufficient by themselves for `REPRODUCED`.
13. **Untouched production proof:** A final reproduction executes against the production/application code of the exact target revision without behavior-changing production modifications introduced by ABR.
14. **Independent verification:** A reproduction candidate must pass an independent verification step before `REPRODUCED` can be returned.
15. **Semantic validity:** Verification checks that the observed failure corresponds to the reported behavior, not merely that something failed.
16. **Second independent investigation:** If the primary investigator exhausts its budget without reproduction or a legitimate blocker, ABR runs a second independent investigation before `NOT_REPRODUCED`.
17. **Correct `NOT_REPRODUCED` semantics:** `NOT_REPRODUCED` is presented as an investigation result within declared constraints, never as proof that the bug does not exist.
18. **Controlled `INCONCLUSIVE`:** `INCONCLUSIVE` identifies a material blocker and is not used merely because the case is difficult.
19. **Minimal human escalation:** ABR requests additional human/source information only when the missing information is material, cannot reasonably be derived, and cannot reasonably be explored through bounded experiments.
20. **Fresh run after new information:** New external information creates a new run snapshot rather than mutating/resuming the old investigation context.
21. **Structured Run Record:** Every terminal run emits a self-contained structured Run Record with provenance, constraints, evidence, outcome, termination reason, resource metadata, and artifact references as applicable.
22. **Portable reproduction artifact:** Every `REPRODUCED` outcome produces a revision-bound, portable, independently executable, self-describing, inspectable, and verifiable reproduction artifact.
23. **Configurable persistence:** Run records and artifacts can be retained through a configurable Run/Artifact Store capability without requiring a specific storage technology.
24. **Fresh execution / persistent evidence:** Persisted prior runs do not implicitly become investigator memory in later runs.
25. **Trace discipline:** Material decision/evidence trace is always available; exhaustive raw audit retention is optional.
26. **Sensitive-data protection:** Persistent records and external publications pass through a sanitization/redaction boundary and do not intentionally expose credentials.
27. **Optional GitHub write-back:** The MVP can publish concise terminal outcomes and necessary information requests to the originating GitHub Issue when enabled.
28. **No issue-management dependency:** Successful reproduction does not require labels, issue state changes, assignments, or pull requests.
29. **Evidence-backed confidence:** Terminal outcomes include a confidence assessment that does not replace evidence or verification.
30. **Benchmark separation:** Benchmark case input and hidden evaluation ground truth are separated so investigators/verifier cannot see the oracle.
31. **Negative-case evaluation:** The evaluation protocol can include cases whose correct behavior is not `REPRODUCED`.
32. **False-positive measurement:** External evaluation can distinguish verifier-rejected candidates from final false-positive `REPRODUCED` outcomes.
33. **Separate metrics:** Evaluation can report setup, reproduction validity, false positives/misses, escalation, second-investigator behavior, resource usage, latency, and failure modes separately.
34. **ABR-specific aggregate view:** Persisted Run Records can be aggregated into an inspectable performance/evaluation view across the principal outcome, cost, latency, model/policy, and failure-mode dimensions.
35. **No repair scope creep:** The MVP can satisfy all acceptance criteria without implementing bug repair, repair PR generation, autonomous production debugging, general-purpose issue triage, or a generic agent platform.

---

## 25. Product Boundary Summary

ABR is successful when it can take a concrete bug report and a specific repository state, autonomously perform a bounded investigation, and leave behind a result that another engineer or evaluator can inspect and verify.

Its execution model is deliberately:

> **fresh execution, persistent evidence**

Its success standard is deliberately stronger than "generated a failing test":

> **the reported behavior must be reproducibly observable for the expected reason against the declared code revision.**

Its first product increment remains deliberately narrow:

> **reproduction, not repair.**
