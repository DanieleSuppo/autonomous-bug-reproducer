# Autonomous Bug Reproducer

ABR investigates a reported bug against a fixed repository revision and produces inspectable evidence of a reproduction outcome.

## Language

**ABR Host**:
The runtime that executes ABR itself, separate from the technology stack of the repository being investigated.
_Avoid_: target runtime, repository runtime

**Target Repository**:
The repository and resolved revision whose reported behavior ABR investigates and attempts to reproduce.
_Avoid_: ABR repository, host repository

**Repository Execution Contract**:
Versioned repository evidence from which ABR can derive a non-interactive, isolated, and verifiable execution plan.
_Avoid_: language profile, framework whitelist

**Run**:
One bounded reproduction attempt over immutable input snapshots and an explicit investigation policy.
_Avoid_: resumable investigation, agent session

**Candidate Package**:
Revision-bound reproduction material and its claimed observable, submitted for independent verification.
_Avoid_: verified reproduction, final result

**Verification Record**:
The independently produced evidence and decision that accepts or rejects a Candidate Package.
_Avoid_: investigator judgment

**Terminal Disposition**:
The single final classification of a Run: `UNSUPPORTED`, `REPRODUCED`, `NOT_REPRODUCED`, `INCONCLUSIVE`, or `SYSTEM_ERROR`.
_Avoid_: outcome when referring specifically to eligibility or system failure

**Run Record**:
The versioned, self-contained, sanitized record of a terminal Run and its material provenance, evidence, decisions, and resource use.
_Avoid_: agent memory, result comment

**Evidence Item**:
An attributable observation, artifact reference, or deterministic check material to a Run decision.
_Avoid_: unfiltered tool log, private reasoning

**Bug Source**:
The external system from which ABR snapshots the bug report input for a Run.
_Avoid_: repository provider, result sink

**Repository Provider**:
The external system from which ABR resolves and obtains the Target Repository revision.
_Avoid_: bug source, source code host

**Result Sink**:
The external system to which ABR optionally publishes a sanitized result projection after a Run terminates.
_Avoid_: Run/Artifact Store, bug source

**Result Projection**:
A deliberately limited, sanitized external presentation derived from a Run Record.
_Avoid_: canonical record, full trace

**Reproduction Bundle**:
The portable, revision-bound directory of reproduction material, manifest, execution instructions, and declared observable submitted in a Candidate Package.
_Avoid_: repair patch, pull request, opaque archive

**Benchmark Case**:
A versioned, immutable set of input visible to ABR for evaluating one Run against a controlled target.
_Avoid_: evaluation oracle, hidden ground truth

**Evaluation Oracle**:
Ground truth and adjudication evidence held separately from a Benchmark Case and inaccessible to ABR actors.
_Avoid_: bug report context, verifier input
