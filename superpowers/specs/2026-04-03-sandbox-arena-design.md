---
title: Sandbox arena design
description: Product and engineering design for the first sandbox evaluation arena.
---

# Sandbox arena design

## Summary

This document defines the first version of a new engineer-facing arena product for comparing sandbox platforms.

Version one focuses on a single category: `sandbox`.

The product is not a content site or a configurable admin product. It is an evaluation tool that runs real benchmark tasks, records structured results, and helps engineers compare providers through metrics, logs, and repeatable runs.

The first providers in scope are:

- `E2B`
- `Sprite`
- `BoxLite`
- `Rivet Agent OS`

## Goals

- Build one solid category before expanding to models, agent frameworks, or other tool classes.
- Keep the codebase unified around one platform model instead of scattering logic by provider.
- Run benchmarks through a shared runner contract instead of embedding provider-specific workflow in page code.
- Present results in a way that helps engineers compare stability, speed, capability coverage, and execution details.
- Support both low-level capability checks and higher-level coding workflow validation.

## Non-goals

- Building a no-code category configuration system.
- Supporting non-engineer audiences.
- Producing a single absolute leaderboard for all use cases.
- Shipping multiple categories in the first release.
- Solving external agent submission in version one.

## Product scope

The first release is a `sandbox arena` with one category and one shared product shell.

The arena evaluates sandbox providers through two layers of tasks:

1. Atomic capability tests.
2. Engineering scenario tasks.

The first engineering scenario task is `code execution and validation`.

This scenario answers a practical engineering question: can a sandbox reliably prepare an execution environment, install dependencies when needed, run commands or tests, and return enough structured output to understand what happened.

## Design principles

### One platform model

The system should use one platform vocabulary across the full product. Providers can differ in behavior and supported features, but they should not create parallel page flows, parallel result types, or provider-specific product architecture.

### Fixed categories in code

Version one is intentionally code-defined. Categories, suites, and provider registrations live in code. This keeps the product focused on engineering workflows and avoids building a heavy configuration layer too early.

### Unified execution boundary

The platform defines tasks, run lifecycle, result shape, and comparison views. Each provider implements an adapter behind a shared runner contract. The product should never require page-level logic to know how a specific provider executes work.

### Multi-dimensional comparison

The product should not force a single total ranking. Engineers need to sort and compare providers by different dimensions depending on workload. The UI should emphasize switchable views such as stability, speed, capability coverage, and cost when available.

### Explainable results

Every benchmark result should include enough context to explain success or failure. Metrics alone are not enough. Logs, artifacts, status changes, and task inputs are part of the product, not debugging extras.

## Domain model

The platform should use a small set of core entities.

### `BenchmarkTrack`

Represents an evaluation category. Version one only includes `sandbox`.

### `SandboxProvider`

Represents one entrant in the arena, such as `E2B` or `BoxLite`. A provider includes descriptive metadata, supported capability declarations, and a link to its runner adapter.

### `TaskSuite`

Represents a grouped benchmark set inside a track. The initial suites are:

- `capability`
- `code-exec`

### `TaskCase`

Represents one runnable benchmark case. Examples include `file-write`, `network-access`, or `install-and-run-tests`.

### `Run`

Represents one execution of a provider against a task case. A run tracks lifecycle state, timestamps, status transitions, and output references.

### `Metric`

Represents a structured measurement attached to a run, such as duration, exit code, setup time, install time, or success state.

### `Artifact`

Represents execution output that helps with interpretation, such as stdout, stderr, event logs, command history, or captured files.

### `ScoreCard`

Represents a normalized comparison summary for UI consumption. A scorecard groups results into platform-level comparison buckets such as quality, execution metrics, and replay or review details. It can expose category-specific fields without changing the top-level product structure.

## Result framework

All category output should map into three top-level result sections:

- `quality`
- `execution metrics`
- `review data`

For sandbox benchmarks, these sections mean:

- `quality`: whether the task completed correctly and whether the provider met the expected capability or scenario outcome.
- `execution metrics`: timing, exit behavior, retry patterns, setup overhead, and cost if it becomes available.
- `review data`: logs, command trace, task input, artifacts, and failure evidence.

Not every task case needs the same fields, but every result must fit into this shared structure.

## Task model

Version one has two task layers.

### Atomic capability tests

These are narrow checks that isolate one platform behavior at a time. Initial examples include:

- file read and write
- command execution
- dependency installation
- network access
- timeout handling
- long output handling
- concurrent run behavior
- environment initialization time

These tests explain why a provider succeeds or fails in a larger workflow.

### Engineering scenario tasks

These tasks simulate real engineering work.

Version one includes one primary scenario: `code execution and validation`.

A typical flow is:

1. Prepare or load a small project fixture.
2. Run setup commands.
3. Install dependencies when required.
4. Execute a validation command such as tests or a build script.
5. Capture structured outputs, logs, timing, and final state.

This scenario keeps the first release grounded in real coding-agent workloads without requiring full code modification tasks on day one.

## Execution architecture

The platform should separate benchmark orchestration from provider execution.

### Platform responsibilities

- Register tracks, suites, cases, and providers.
- Define the task contract and expected result schema.
- Schedule runs.
- Persist run state, metrics, and artifacts.
- Build comparison views and run detail pages from normalized output.

### Runner responsibilities

- Accept one normalized benchmark task.
- Translate it into provider-specific execution steps.
- Stream or report lifecycle events.
- Return normalized metrics and artifacts.
- Surface provider-specific errors without breaking the platform schema.

### Adapter responsibilities

Each sandbox provider gets its own adapter implementation behind the shared runner contract. Adapters are the only layer that should know provider-specific SDK calls, authentication requirements, environment boot logic, or capability quirks.

This boundary keeps the platform extensible. Adding a new provider should mainly require adapter work and metadata registration, not product-level rewrites.

## Product surfaces

The UI should use one shared page skeleton for the sandbox track.

### Arena home

Shows the sandbox track overview, current providers, recent runs, and links into leaderboards and comparisons.

### Leaderboards

Shows switchable ranking views instead of one universal winner.

Initial ranking dimensions should include:

- stability
- speed
- capability coverage
- cost when available
- review completeness if useful for debugging quality

### Compare view

Lets engineers compare two to four providers side by side.

The compare view should emphasize:

- supported capability matrix
- recent benchmark outcomes
- key timing and reliability metrics
- scenario task results
- links into raw run detail

### Run detail

Shows a single execution in a replay-friendly format.

The page should include:

- task case and suite
- provider
- run status timeline
- metrics
- stdout and stderr
- execution logs
- relevant artifacts
- failure summary when present

## Code organization

The first implementation should favor platform-first boundaries over provider-first sprawl.

Suggested structure:

- `src/domain/arena`: shared product model and registration types
- `src/domain/benchmark`: task, run, metric, artifact, and score types
- `src/features/providers`: provider registry and provider-facing UI helpers
- `src/features/sandbox`: sandbox track definitions, suite metadata, and comparison config
- `src/runners`: runner contract, orchestration entry points, and execution lifecycle helpers
- `src/runners/providers/e2b`
- `src/runners/providers/sprite`
- `src/runners/providers/boxlite`
- `src/runners/providers/rivet`
- `src/routes`: route entry points
- `src/components`: presentational components shared across views

The intent is simple:

- platform model stays centralized
- sandbox track logic stays together
- provider differences stay inside adapters
- routes and components stay provider-agnostic

## Data flow

The expected run flow is:

1. An engineer selects a suite, task case, and one or more providers.
2. The platform creates benchmark run records.
3. The orchestration layer dispatches each run through the shared runner contract.
4. The selected provider adapter performs execution.
5. The adapter reports lifecycle events, metrics, and artifacts.
6. The platform persists normalized results.
7. Leaderboards, compare pages, and run detail pages render from the stored result model.

This model supports both direct UI-triggered runs and future scheduled or agent-triggered runs without changing the product vocabulary.

## Error handling

Version one should treat failure as a first-class result.

The platform should distinguish between:

- task failure: the benchmark ran correctly, but the provider failed the task
- runner failure: the adapter or execution pipeline failed
- platform failure: the arena could not schedule, persist, or read the run correctly

Each failure mode should remain visible in run detail and should not collapse into a generic error state.

## Testing strategy

The implementation should verify the platform at three levels.

### Domain tests

Validate scorecard shaping, run normalization, and leaderboard sorting.

### Runner contract tests

Validate that provider adapters return the required run lifecycle fields, metrics, and artifact references.

### UI integration tests

Validate that leaderboards, compare views, and run detail pages render the normalized result model correctly.

Version one does not need exhaustive live-provider coverage in every automated test. It does need a stable way to test the shared runner contract and result shaping logic.

## Future expansion

This design intentionally leaves room for later additions without changing the first version architecture.

Likely next steps after version one:

- more sandbox scenario tasks such as code modification or environment bootstrapping
- scheduled benchmark runs
- historical trend views
- external agent-triggered submissions
- new top-level tracks such as models or agent frameworks

Those additions should reuse the same platform primitives where possible. New work should extend the track and runner ecosystem, not create separate product silos.

## Open decisions deferred from version one

These topics are intentionally not locked in by this design document:

- the exact persistence layer for run data
- whether runs are executed fully in-process or through a background job system
- the final visual design language of the frontend
- how provider credentials are stored in development and production

These decisions belong in the implementation plan and can be resolved with the real project constraints in view.
