---
title: Sandbox arena v2 design
description: Revised first-version design for arena with a lightweight backend, JSON persistence, and BoxLite live integration.
---

# Sandbox arena v2 design

## Summary

This document revises the first-version design of `arena` after the initial frontend shell was built.

The product remains an engineer-facing sandbox evaluation tool, but the first release is narrowed and clarified in three important ways:

- the default provider set changes to `OrbStack machine`, `BoxLite`, and `Sprite`
- `arena` now includes a lightweight built-in backend
- the first live integration is `BoxLite`, while the other providers remain registered but not yet fully connected

The goal of this revision is to keep the product honest and usable while preventing the frontend from becoming a mixed layer of fixture logic, page shaping, and execution orchestration.

## Goals

- Keep the first release focused on one category: `sandbox`
- Show a tighter first provider set that matches the current evaluation focus
- Move execution and persistence responsibilities out of the frontend
- Keep the frontend organized around display and interaction, not orchestration
- Deliver one real provider integration through `BoxLite`
- Preserve a shared platform model so more providers can be added without rewriting product structure

## Non-goals

- Introducing authentication or permissions in version one
- Building a configurable admin backend
- Supporting every provider with live execution on day one
- Adding a database for the first release
- Rebuilding the frontend from scratch

## Revised product scope

The first release keeps one track, `sandbox`, but changes the first-release provider set to:

- `OrbStack machine`
- `BoxLite`
- `Sprite`

These providers do not all have the same implementation state.

- `BoxLite` is the first live provider
- `OrbStack machine` is registered for the product shell and comparison surfaces, but not fully live in version one
- `Sprite` is registered for the product shell and comparison surfaces, but not fully live in version one

This distinction must be visible in the product. The UI should not imply that every listed provider already supports the same depth of execution.

## Why this revision exists

The initial `arena` implementation proved the shared model and UI shape, but it also revealed an architectural pressure point.

The frontend already contains route loaders, fixture-backed view helpers, and page-specific composition logic. That is still manageable today, but if real execution is added without a backend boundary, the frontend will start owning execution state, provider coordination, and persistence concerns. That would make the code harder to extend and harder to reason about.

The revised first version fixes that by introducing a lightweight backend inside the same `arena` project.

## Design principles

### One project, two clear runtime roles

`arena` stays a single project, but it should have a clearer internal split:

- frontend: interaction, route rendering, comparison views
- backend: execution, persistence, run lifecycle, provider-backed APIs

This keeps deployment and local development simple without mixing responsibilities.

### Honest provider status

The UI should represent provider readiness truthfully. A registered provider is not automatically a live provider. The product should distinguish between:

- registered provider metadata
- fixture-backed comparison presence
- live execution support

### Thin frontend data shaping

The frontend should stop acting as the source of benchmark truth. It can still adapt backend responses for page use, but it should no longer own the primary run state model or persistence story.

### Lightweight persistence first

Version one should persist benchmark state in local JSON files. This is intentionally simple. The design should make later migration possible, but it should not pre-emptively introduce database complexity.

## System boundaries

### Frontend responsibilities

- render provider, leaderboard, compare, and run-detail surfaces
- trigger benchmark runs through API calls
- poll or refresh run status when needed
- render provider readiness state such as `live` or `planned`
- adapt backend responses into page-friendly display data

### Backend responsibilities

- expose provider registry and provider status
- create and manage benchmark runs
- execute live benchmark runs through provider adapters
- persist runs, metrics, and artifacts to local JSON files
- expose run list, run detail, leaderboard, and compare APIs

The backend becomes the source of truth for run state.

## Provider model

The provider model should gain one explicit readiness field.

Suggested states:

- `live`
- `planned`

Version one uses these states as follows:

- `BoxLite`: `live`
- `OrbStack machine`: `planned`
- `Sprite`: `planned`

This allows the product to show the intended provider lineup without faking equal implementation depth.

## Task model

The overall task model remains the same as the first design:

- atomic capability tests
- engineering scenario tasks

The primary engineering scenario still remains `code execution and validation`.

For the revised first version, `BoxLite` is only required to support the minimum live path for this scenario. The product does not need to support every possible benchmark mode immediately.

## Backend shape

The lightweight backend should live inside `arena` and use the existing server capabilities of the app stack instead of creating a second standalone service.

The backend should provide at least these API surfaces:

- `GET /api/providers`
- `GET /api/runs`
- `GET /api/runs/:id`
- `POST /api/runs`
- `GET /api/leaderboard`
- `GET /api/compare`

These routes should return normalized application data, not raw provider-specific response shapes.

## Persistence model

Version one should use local JSON persistence.

Suggested layout:

- `arena/.local/runs/index.json`
- `arena/.local/runs/<run-id>.json`
- `arena/.local/artifacts/<run-id>/stdout.log`
- `arena/.local/artifacts/<run-id>/stderr.log`
- `arena/.local/artifacts/<run-id>/commands.log`

The repository should ignore `.local/`.

The index file should support listing and sorting runs without reading every artifact file.

Each run file should contain:

- run metadata
- provider id
- suite id
- task case id
- lifecycle timestamps
- normalized metrics
- references to artifact files
- final status and summary

## Execution flow

The revised first-version run flow should be:

1. Frontend requests a run through `POST /api/runs`
2. Backend validates provider and task input
3. Backend creates an initial run record in JSON storage
4. Backend dispatches execution through the live provider adapter
5. Adapter returns normalized metrics and artifacts
6. Backend writes the completed run and artifact references to storage
7. Frontend reads list and detail APIs to display updated results

The first version does not need a distributed queue or worker system. It only needs a clear execution boundary and durable local state.

## BoxLite live integration

`BoxLite` is the first provider that must execute real work.

The live integration only needs to support the minimum path for version one:

- accept the normalized `code-exec` task input
- execute the task through BoxLite
- capture timing, status, and result summary
- collect enough artifact output to support run detail and debugging
- map BoxLite output into the shared `BenchmarkRun` shape

The live adapter should stay isolated behind the backend execution layer. Provider-specific details must not leak into frontend route or component code.

## Leaderboard and compare behavior

Leaderboards and compare views should read backend data instead of frontend fixture constants.

The product should still support multiple ranking dimensions, but those rankings now come from stored run data.

Because only `BoxLite` is live in version one, the UI should make one of these states explicit for every provider row:

- live result available
- planned or fixture-only
- no run data yet

This keeps the comparison honest and avoids implying nonexistent parity.

## Frontend structure changes

The current frontend does not need a full rewrite, but it should be tightened.

Recommended structure:

- `src/routes/`: route entries and loader wiring only
- `src/features/arena/`: page-level sections and API-facing adapters for frontend display
- `src/components/`: presentational UI components
- `src/domain/benchmark/`: shared model and type definitions
- `src/server/arena/`: provider registry, run service, JSON repository, adapter execution

The main improvement is this:

- routes should not be long data kitchens
- `arena-data` should stop being the source of truth for run state
- backend services should own execution and persistence

## Component quality goals

The current component layer is not yet badly tangled, but the risk is growing in the page-data boundary rather than in primitive components.

For this revision, the frontend cleanup should focus on:

- keeping components presentational where possible
- keeping page sections separate from base display components
- removing provider-specific execution assumptions from UI components
- moving source-of-truth data access away from route-local or fixture-local composition

This is a medium cleanup, not a redesign.

## Error handling

The backend must treat these failure classes separately:

- invalid request: bad provider or task input
- persistence error: failed to read or write JSON state
- execution error: provider adapter failed to execute
- task failure: execution completed, but benchmark outcome failed

Run detail should preserve these distinctions.

## Testing strategy

The revised first version should verify four areas:

### Frontend rendering

Route and component tests should verify that:

- provider status is rendered correctly
- compare and leaderboard pages handle live and planned providers
- run detail renders backend-backed data cleanly

### Backend JSON repository

Tests should verify that:

- run index writes succeed
- individual run reads and writes succeed
- artifact references stay consistent

### Execution service

Tests should verify that:

- `POST /api/runs` creates a run record
- live adapter results become normalized run output
- backend failure states are represented correctly

### BoxLite adapter contract

Tests should verify that BoxLite live integration maps execution output into the shared model without leaking provider-specific response shapes.

## Code organization

The previous design recommended a fully frontend-centered structure. The revised design changes that to make room for the built-in backend.

Suggested organization now:

- `src/domain/benchmark`: shared task, run, metric, and score types
- `src/features/providers`: frontend-facing provider display helpers if needed
- `src/features/arena`: frontend sections and API-facing view shaping
- `src/components`: presentational components
- `src/routes`: frontend route entries
- `src/server/arena/providers`: provider registry and provider metadata for backend use
- `src/server/arena/repository`: JSON persistence layer
- `src/server/arena/service`: run creation, list, detail, leaderboard, and compare services
- `src/server/arena/adapters/boxlite`: live BoxLite integration
- `src/routes/api/*` or equivalent server route location: API handlers

This keeps execution and persistence out of UI code while preserving the shared benchmark domain.

## Deferred items

These are intentionally deferred from this revised first version:

- live integration for `OrbStack machine`
- live integration for `Sprite`
- authentication
- background job infrastructure
- database storage
- multi-user concurrency or tenant separation

## Summary decision

The revised first version of `arena` should be understood as:

- a sandbox arena with three default providers
- one live provider, `BoxLite`
- a built-in lightweight backend
- JSON-backed local persistence
- a frontend that is tightened, not rewritten

This is the smallest architecture that keeps the product honest, keeps the codebase from drifting into a frontend-only tangle, and creates a clean path for future provider expansion.
