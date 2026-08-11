# ABR — Autonomous Bug Reproducer

> Turn a bug report into a reproducible failing test.

ABR is an autonomous AI agent that takes a software repository and a bug report, investigates the codebase, forms and tests hypotheses, and attempts to produce the smallest executable test or scenario that reliably reproduces the reported bug.

The project is intentionally focused on **reproduction, not repair**. Its job is to turn an ambiguous issue into a concrete, inspectable failure that a developer—or another agent—can work from.

## Why ABR

Bug reports are often incomplete, imprecise, or disconnected from the exact code path that causes the failure. Before a bug can be fixed confidently, someone usually has to:

- understand the repository and its test strategy;
- identify the relevant code paths;
- infer missing reproduction conditions;
- build one or more hypotheses;
- run experiments;
- separate real failures from setup problems or false positives;
- reduce the result to a stable test or minimal scenario.

ABR treats that investigation as a measurable agentic workflow.

Instead of asking an AI system to "fix this issue" and judging the resulting patch subjectively, ABR has a narrower and more verifiable goal:

**Can the agent make the bug fail on demand?**

That makes bug reproduction a useful vertical problem for evaluating autonomous software-engineering agents.

## What ABR does

Given:

1. a source repository;
2. a bug report or issue;
3. the instructions required to build and test the project;

ABR attempts to:

1. inspect the repository and understand its structure;
2. collect evidence relevant to the reported behavior;
3. generate explicit hypotheses about the failure;
4. design reproduction attempts;
5. execute them in an isolated environment;
6. update or discard hypotheses based on observed results;
7. produce a minimal failing automated test whenever possible;
8. fall back to a reproducible executable scenario when a normal test is not appropriate;
9. report failure transparently when reproduction cannot be established.

A successful run should leave behind something another engineer can execute independently and observe failing for the expected reason.

## Core principle

ABR distinguishes three very different outcomes:

### Reproduced

The reported behavior has been reproduced through a deterministic or sufficiently stable test/scenario, with evidence that the observed failure corresponds to the issue.

### Not reproduced

The agent completed meaningful investigation and experiments but could not establish the reported failure within its constraints.

### Inconclusive

The run could not answer the reproduction question because of environment, dependency, data, permission, infrastructure, or other blocking conditions.

ABR should never turn an inability to reproduce into a fabricated root cause.

## Output of a run

A run is more than a generated test file. ABR records enough information to make the result inspectable:

- reproduction status;
- generated or modified files;
- exact commands required to reproduce the result;
- relevant observations and evidence;
- hypotheses considered during investigation;
- failed reproduction attempts;
- environment and setup information;
- agent/tool execution trace;
- cost and latency data where available;
- confidence in the final reproduction result.

The goal is that a developer can understand both **what ABR concluded** and **why**.

## What ABR does not do

The initial scope is deliberately narrow.

ABR does **not**:

- automatically fix the bug;
- open a pull request containing a repair;
- claim a root cause unless the evidence supports one;
- treat a compilation or environment failure as successful bug reproduction;
- optimize for the largest number of generated tests;
- hide unsuccessful attempts or agent failures.

Bug fixing may become a downstream workflow later, but reproduction remains a separate capability with its own evaluation criteria.

## Agent workflow

Conceptually, an ABR run follows this loop:

```text
Issue + Repository
        |
        v
Repository exploration
        |
        v
Evidence collection
        |
        v
Hypothesis generation
        |
        v
Reproduction experiment
        |
        +------ failure / new evidence ------+
        |                                    |
        +<----- hypothesis refinement <------+
        |
        v
Minimal failing test / scenario
        |
        v
Verification + report + trace
```

The workflow is iterative rather than a single prompt-response interaction. Hypotheses are expected to change as the agent observes the repository and executes experiments.

## Evaluation-first development

ABR is designed to be evaluated as an engineering system, not demonstrated through cherry-picked examples.

The evaluation suite will use issues with known outcomes and separate several failure dimensions rather than collapsing everything into one success rate.

Initial metrics include:

- repository setup success rate;
- bug reproduction rate;
- valid failing-test rate;
- false-positive reproduction rate;
- number of reproduction attempts;
- tool/model cost;
- end-to-end latency;
- failure-mode distribution.

Where technically possible, runs should be replayable so that changes to models, prompts, tools and policies can be compared against the same cases.

No benchmark results are published yet. Metrics will be added only after the evaluation dataset and execution protocol are defined and run.

## Reproduction quality

Generating a failing test is not sufficient on its own.

A reproduction is useful only when the failure is causally relevant to the original report. A test that fails because the environment is misconfigured, an unrelated assertion was introduced, or the agent intentionally broke the application is a false positive.

ABR therefore evaluates at least two separate questions:

1. **Does the produced test/scenario fail reliably?**
2. **Does it fail because of the behavior described by the issue?**

This distinction is central to the project.

## Observability

Agent behavior must be inspectable.

ABR records structured execution information for repository exploration, tool calls, hypotheses, experiments, retries, failures and resource usage. This instrumentation is initially part of ABR itself rather than a standalone framework.

If the tracing model proves reusable across other agents, it may later be extracted into a dedicated **AI Flight Recorder** component.

## Evaluation infrastructure

Similarly, benchmark execution, replay, regression testing and model/policy comparisons begin inside this repository.

If those capabilities become independently useful, they can evolve into a separate **Agent Evaluation Lab** rather than being designed as a generic framework upfront.

The guiding rule is simple:

> Extract infrastructure from proven needs; do not build the platform before the vertical works.

## Roadmap

The first development milestone focuses only on reliable bug reproduction:

- formalize the input/output contract;
- define the first benchmark dataset;
- establish reproduction and false-positive criteria;
- implement repository exploration and experiment execution;
- produce minimal failing tests/scenarios;
- capture structured traces and run metrics;
- evaluate the system across repeatable cases.

Later iterations may explore model/policy comparison, richer tracing, adaptive model routing and integration into broader agentic software-development workflows.

## Project status

ABR is currently at the **specification and benchmark-design stage**.

The repository intentionally starts with this README before implementation. The next artifacts will define the machine-readable input/output contract, evaluation protocol, initial architecture and benchmark cases.

## Name

**ABR** stands for **Autonomous Bug Reproducer**.
