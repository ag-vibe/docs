---
title: Arena cleanup and README onboarding design
description: Remove dead compatibility code and stale runner files, then rewrite README.md with a stronger get started flow.
---

# Arena cleanup and README onboarding design

## Goal

Make `arena` internally consistent after the backend-backed first-release migration by removing dead code that still reflects the retired provider set and by rewriting `README.md` so a new engineer can get started without reading the implementation plan.

## Scope

This work covers two linked cleanup areas:

- code cleanup for obviously dead or obsolete implementation paths
- documentation cleanup for `README.md`, including a clearer onboarding section

This does not change user-facing product behavior beyond documentation and removal of unused code.

## Cleanup design

### Dead code removal

The codebase now uses the provider trio `orbstack-machine`, `boxlite`, and `sprite`. Any runner adapter or compatibility surface that still exists only for the removed `e2b` and `rivet` provider set should be deleted if it has no remaining references.

The expected primary cleanup targets are:

- `src/runners/providers/e2b.ts`
- `src/runners/providers/rivet.ts`

Before deletion, all references will be searched. If any remaining usages exist, they will either be updated to the current provider set or removed if they are stale tests or compatibility-only paths.

### Compatibility boundary cleanup

The previous migration already moved route ownership to API-backed feature modules. This pass should finish the cleanup by removing any remaining unused compatibility code that still implies the old fixture-owned route architecture.

The desired end state is:

- route loaders depend on `src/features/arena/*`
- components depend on typed feature outputs or small helper functions
- `src/lib/arena-data.ts` remains only as a helper module for reusable shaping logic that still has real consumers

If a helper or export is no longer imported anywhere, it should be removed in this pass.

## README design

`README.md` should stop reading like a brief project note and instead act as a practical engineer entry point.

### Target structure

The README should be reorganized into these sections:

1. `# arena`
2. short project summary
3. `## Get started`
4. `## What this app does`
5. `## First-release scope`
6. `## Local persistence`
7. `## Backend routes`
8. `## Project structure`
9. `## Verification`

### Get started content

The `Get started` section should be task-oriented and concrete. It should include:

- dependency install with `vp install`
- local dev server with `vp dev --port 3000`
- note that the app persists run data under `.local/`
- note that deleting `.local/` resets locally persisted runs

### Architecture and structure wording

The README should describe the app as it exists now:

- domain model in `src/domain/benchmark`
- backend services, persistence, and adapters in `src/server/arena`
- frontend feature adapters in `src/features/arena`
- UI routes in `src/routes`
- runner contract/orchestrator only if still relevant after dead-code cleanup

Old wording that implies the frontend is primarily fixture-backed should be removed.

## Verification

After cleanup and documentation updates, verification should run at full project level:

```bash
vp test run && vp check && vp build
```

## Risks and mitigations

- Risk: deleting stale runner files could break hidden imports.
  Mitigation: grep for references before deletion and rerun full verification.

- Risk: README drift if the architecture section is copied from the old plan instead of the current code.
  Mitigation: base the README on the implemented folder ownership now in the repo.

## Success criteria

- no references remain to removed `e2b` or `rivet` runner adapters
- no obviously dead compatibility exports remain in the migrated frontend data path
- `README.md` contains a clear `Get started` section and accurate architecture/runtime guidance
- `vp test run && vp check && vp build` passes
