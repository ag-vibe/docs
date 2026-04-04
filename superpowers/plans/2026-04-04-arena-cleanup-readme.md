# Arena Cleanup And README Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove dead runner and compatibility code left behind by the arena migration, then rewrite `README.md` with a practical get started guide that matches the implemented backend-backed app.

**Architecture:** The cleanup keeps the current backend-backed arena flow intact and only removes stale code that still reflects the retired provider set or obsolete frontend ownership boundaries. Documentation is rewritten around the actual folder ownership and runtime behavior in the repo so new engineers can start, reset local data, and verify the app without reading source first.

**Tech Stack:** TypeScript, TanStack Router, Vite+, ofetch, Vitest via `vp test run`

---

## File Map

- Delete: `src/runners/providers/e2b.ts` - stale runner adapter for removed provider
- Delete: `src/runners/providers/rivet.ts` - stale runner adapter for removed provider
- Modify: `src/runners/orchestrator.ts` - ensure imports and registry only reference active providers
- Modify: `src/runners/orchestrator.test.ts` - ensure tests assert current provider trio only
- Modify: `src/lib/arena-data.ts` - remove any remaining unused compatibility exports if present after cleanup
- Modify: `src/components/compare-matrix.tsx` - remove any stale compatibility-derived typing if still present
- Modify: `src/components/arena-ui.test.tsx` - keep UI tests independent from deleted compatibility surfaces
- Modify: `README.md` - add `Get started` flow and refresh architecture/runtime docs

### Task 1: Remove Dead Runner Adapters

**Files:**
- Delete: `src/runners/providers/e2b.ts`
- Delete: `src/runners/providers/rivet.ts`
- Modify: `src/runners/orchestrator.ts`
- Test: `src/runners/orchestrator.test.ts`

- [ ] **Step 1: Confirm the current orchestrator test covers only active providers**

Read `src/runners/orchestrator.test.ts` and verify it references only:

```ts
["orbstack-machine", "sprite", "boxlite"]
```

and imports:

```ts
import { orbstackMachineRunner } from "@/runners/providers/orbstack-machine";
```

- [ ] **Step 2: Delete the stale runner files**

Delete these files exactly:

```text
src/runners/providers/e2b.ts
src/runners/providers/rivet.ts
```

- [ ] **Step 3: Remove any dead imports from the orchestrator**

Ensure `src/runners/orchestrator.ts` imports only active adapters:

```ts
import { boxliteRunner } from "@/runners/providers/boxlite";
import { orbstackMachineRunner } from "@/runners/providers/orbstack-machine";
import { spriteRunner } from "@/runners/providers/sprite";

const runnerRegistry = new Map<string, SandboxRunnerAdapter>([
  [orbstackMachineRunner.providerId, orbstackMachineRunner],
  [spriteRunner.providerId, spriteRunner],
  [boxliteRunner.providerId, boxliteRunner],
]);
```

- [ ] **Step 4: Run the focused orchestrator test**

Run: `vp test run src/runners/orchestrator.test.ts`

Expected: PASS with all orchestrator tests green.

### Task 2: Remove Any Remaining Dead Compatibility Surface

**Files:**
- Modify: `src/lib/arena-data.ts`
- Modify: `src/components/compare-matrix.tsx`
- Modify: `src/components/arena-ui.test.tsx`
- Test: `src/lib/arena-data.test.ts`
- Test: `src/components/arena-ui.test.tsx`

- [ ] **Step 1: Check `src/lib/arena-data.ts` for helper-only exports**

The file should contain only reusable shaping helpers such as:

```ts
export function buildCompareRows(providers: SandboxProvider[], runs: BenchmarkRun[]) {
  return [
    {
      label: "Focus",
      values: providers.map((provider) => provider.focus),
    },
    {
      label: "Capabilities",
      values: providers.map((provider) => provider.capabilities.join(", ")),
    },
    {
      label: "Latest summary",
      values: providers.map(
        (provider) => runs.find((run) => run.providerId === provider.id)?.summary ?? "No runs",
      ),
    },
  ];
}
```

and:

```ts
export function buildRunDetail(run: BenchmarkRun) {
  const provider = sandboxProviders.find((entry) => entry.id === run.providerId);
  const suite = sandboxSuites.find((entry) => entry.id === run.suiteId);
  const taskCase = suite?.taskCases.find((entry) => entry.id === run.taskCaseId);

  if (!provider || !suite || !taskCase) {
    return null;
  }

  return {
    run,
    provider,
    suite,
    taskCase,
    scoreCard: buildScoreCard(run),
  };
}
```

If any route-owning exports remain, remove them in this task.

- [ ] **Step 2: Check component typing for stale compatibility coupling**

Ensure `src/components/compare-matrix.tsx` consumes the feature-layer type directly:

```ts
import type { CompareView } from "@/features/arena/compare-data";

export function CompareMatrix({ view }: { view: CompareView }) {
```

- [ ] **Step 3: Keep UI tests independent of deleted compatibility functions**

Ensure `src/components/arena-ui.test.tsx` builds local test data from current helpers instead of importing deleted route-view builders. The test should construct values like:

```ts
const compareView: CompareView = {
  providers: sandboxProviders.filter((provider) => ["boxlite", "sprite"].includes(provider.id)),
  runs: sandboxRunFixtures.filter((run) => ["boxlite", "sprite"].includes(run.providerId)),
  rows: buildCompareRows(
    sandboxProviders.filter((provider) => ["boxlite", "sprite"].includes(provider.id)),
    sandboxRunFixtures.filter((run) => ["boxlite", "sprite"].includes(run.providerId)),
  ),
};
```

- [ ] **Step 4: Run the focused helper and UI tests**

Run: `vp test run src/lib/arena-data.test.ts src/components/arena-ui.test.tsx`

Expected: PASS with all helper/UI tests green.

### Task 3: Rewrite README With Get Started Flow

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace the current top-level structure with the new outline**

Rewrite `README.md` to use this section order:

```md
# arena

<short summary>

## Get started
## What this app does
## First-release scope
## Local persistence
## Backend routes
## Project structure
## Verification
```

- [ ] **Step 2: Add a practical `Get started` section**

Include this command block:

```bash
vp install
vp dev --port 3000
```

and include plain-language notes that:

- persisted runs are stored in `.local/`
- deleting `.local/` resets locally created runs

- [ ] **Step 3: Update the product/architecture language to match the implemented app**

Describe the current runtime with wording equivalent to:

```md
- Shared benchmark types and fixtures live in `src/domain/benchmark`.
- Backend arena services, persistence, and provider adapters live in `src/server/arena`.
- Frontend route-facing feature loaders live in `src/features/arena`.
- UI routes live in `src/routes`.
- The runner contract and orchestrator live in `src/runners`.
```

Do not describe the app as primarily fixture-backed at the route layer.

- [ ] **Step 4: Keep backend route docs accurate and concise**

Document these routes exactly:

```md
- `GET /api/providers`
- `GET /api/runs`
- `POST /api/runs`
- `GET /api/runs/:runId`
- `GET /api/leaderboard?dimension=<dimension>`
- `GET /api/compare?provider=<id>&provider=<id>`
```

- [ ] **Step 5: Add a verification section**

Include this exact command:

```bash
vp test run && vp check && vp build
```

### Task 4: Final Verification

**Files:**
- Verify repo state only

- [ ] **Step 1: Run the full verification suite**

Run: `vp test run && vp check && vp build`

Expected: all tests pass, no lint/type errors, and production build succeeds.

- [ ] **Step 2: Review diff for scope correctness**

Run: `git diff -- README.md src/runners src/lib/arena-data.ts src/components/compare-matrix.tsx src/components/arena-ui.test.tsx`

Expected: only dead-code cleanup and README/get-started documentation changes remain in scope.
