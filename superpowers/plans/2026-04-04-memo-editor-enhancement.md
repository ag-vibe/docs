# Memo Editor Enhancement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current custom memo editor toolbar with a minimal Shiro-based rich editor that supports floating formatting, slash commands, block handles, and the first-pass rich blocks used in memos.

**Architecture:** Keep the existing memo data flow intact and swap the editor and renderer surfaces to `@haklex/rich-kit-shiro`. Preserve `deriveMemoDraft()` as the serialization boundary, add focused CSS overrides in the existing global stylesheet, and cover the integration with memo component tests before and after the migration.

**Tech Stack:** React 19, TanStack Query, HeroUI, Lexical, Haklex/Shiro, Testing Library, Vite+

---

### Task 1: Lock the current memo editor contract with tests

**Files:**
- Create: `memos/src/components/memo/memo-editor.test.tsx`
- Modify: `memos/src/components/memo/memo-editor.tsx`
- Test: `memos/src/components/memo/memo-editor.test.tsx`

- [ ] **Step 1: Write the failing test**

```tsx
import { fireEvent, render, screen, waitFor } from "@testing-library/react";
import { describe, expect, it, vi } from "vite-plus/test";

import { MemoEditor } from "./memo-editor";

vi.mock("@haklex/rich-kit-shiro", () => ({
  ShiroEditor: ({ onChange, placeholder }: { onChange: (value: unknown) => void; placeholder?: string }) => (
    <div>
      <textarea
        aria-label="Memo editor"
        placeholder={placeholder}
        onChange={(event) => {
          onChange({
            root: {
              type: "root",
              version: 1,
              direction: null,
              format: "",
              indent: 0,
              children: [
                {
                  type: "paragraph",
                  version: 1,
                  direction: null,
                  format: "",
                  indent: 0,
                  textFormat: 0,
                  textStyle: "",
                  children: [
                    {
                      type: "text",
                      version: 1,
                      text: event.currentTarget.value,
                      detail: 0,
                      format: 0,
                      mode: "normal",
                      style: "",
                    },
                  ],
                },
              ],
            },
          });
        }}
      />
    </div>
  ),
}));

describe("MemoEditor", () => {
  it("submits a derived draft from rich editor content", async () => {
    const onSubmit = vi.fn().mockResolvedValue(undefined);

    render(<MemoEditor onSubmit={onSubmit} placeholder="Write memo" />);

    fireEvent.change(screen.getByLabelText("Memo editor"), {
      target: { value: "Ship rich editor #work" },
    });

    fireEvent.click(screen.getByTestId("memo-submit"));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith(
        expect.objectContaining({
          plainText: "Ship rich editor #work",
          excerpt: "Ship rich editor #work",
          tags: ["work"],
        }),
      );
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vp test run src/components/memo/memo-editor.test.tsx`
Expected: FAIL because `memo-editor.tsx` still imports `RichEditor` and does not use `ShiroEditor`.

- [ ] **Step 3: Write minimal implementation**

```tsx
import { ShiroEditor } from "@haklex/rich-kit-shiro";

<ShiroEditor
  initialValue={content}
  placeholder={placeholder}
  onChange={(value) => setContent(value)}
/>
```

Replace the direct `RichEditor` usage while preserving submit behavior and the existing footer button.

- [ ] **Step 4: Run test to verify it passes**

Run: `vp test run src/components/memo/memo-editor.test.tsx`
Expected: PASS

### Task 2: Migrate renderer and editor styles

**Files:**
- Modify: `memos/src/components/memo/memo-content.tsx`
- Modify: `memos/src/styles.css`
- Test: `memos/src/components/memo/memos-page.test.tsx`

- [ ] **Step 1: Write the failing renderer test**

Add a renderer mock and assertion to `memos-page.test.tsx` that proves memo content renders through the Shiro surface:

```tsx
const shiroRendererMock = vi.fn(({ value }: { value: unknown }) => (
  <div data-testid="shiro-renderer">{JSON.stringify(value)}</div>
));

vi.mock("@haklex/rich-kit-shiro", async () => {
  const actual = await vi.importActual<typeof import("@haklex/rich-kit-shiro")>("@haklex/rich-kit-shiro");
  return {
    ...actual,
    ShiroRenderer: shiroRendererMock,
  };
});

expect(await screen.findAllByTestId("shiro-renderer")).toHaveLength(2);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vp test run src/components/memo/memos-page.test.tsx`
Expected: FAIL because `memo-content.tsx` still renders `RichRenderer`.

- [ ] **Step 3: Write minimal implementation**

```tsx
import { ShiroRenderer } from "@haklex/rich-kit-shiro";

<ShiroRenderer value={content} variant="note" theme={theme} />
```

Update `src/styles.css` to swap the Haklex style import to the kit stylesheet and add the memo editor override shell in the same file:

```css
@import "@haklex/rich-kit-shiro/style.css";

.memo-editor-shell {
  border: 1px solid oklch(var(--foreground) / 0.12);
  border-radius: 1rem;
  background: oklch(var(--background));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `vp test run src/components/memo/memos-page.test.tsx`
Expected: PASS

### Task 3: Replace the custom toolbar with the Shiro editor shell

**Files:**
- Modify: `memos/src/components/memo/memo-editor.tsx`
- Modify: `memos/src/styles.css`
- Test: `memos/src/components/memo/memo-editor.test.tsx`

- [ ] **Step 1: Write the failing UI tests**

Extend `memo-editor.test.tsx` with the minimal shell expectations:

```tsx
it("renders a clean editor shell without the legacy toolbar", () => {
  render(<MemoEditor onSubmit={vi.fn().mockResolvedValue(undefined)} />);

  expect(screen.queryByTitle("Bold")).toBeNull();
  expect(screen.getByText("0 chars")).toBeTruthy();
  expect(screen.getByText("Send")).toBeTruthy();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vp test run src/components/memo/memo-editor.test.tsx`
Expected: FAIL because the current toolbar buttons are still rendered.

- [ ] **Step 3: Write minimal implementation**

Remove the legacy toolbar code and keep a focused shell around `ShiroEditor`:

```tsx
<div className="memo-editor-shell">
  <ShiroEditor
    variant="note"
    initialValue={content}
    placeholder={placeholder}
    autoFocus={autoFocus}
    onChange={(value) => setContent(value)}
    className="memo-editor-surface"
    editorClassName={compact ? "memo-editor-content memo-editor-content--compact" : "memo-editor-content"}
  />
  <div className="memo-editor-footer">...</div>
</div>
```

Add matching CSS selectors for the shell, content area, hover/focus states, and footer spacing inside `src/styles.css`.

- [ ] **Step 4: Run test to verify it passes**

Run: `vp test run src/components/memo/memo-editor.test.tsx`
Expected: PASS

### Task 4: Add image upload behavior and rich block affordances

**Files:**
- Modify: `memos/src/components/memo/memo-editor.tsx`
- Test: `memos/src/components/memo/memo-editor.test.tsx`

- [ ] **Step 1: Write the failing behavior test**

Extend the Shiro mock and add a test that proves the editor receives an image upload handler:

```tsx
const shiroEditorMock = vi.fn(
  ({ imageUpload, onChange }: { imageUpload?: (file: File) => Promise<string>; onChange: (value: unknown) => void }) => (
    <div>
      <button
        type="button"
        onClick={async () => {
          if (imageUpload) {
            await imageUpload(new File(["hello"], "memo.png", { type: "image/png" }));
          }
          onChange(makeEditorState("Image memo"));
        }}
      >
        Simulate change
      </button>
    </div>
  ),
);

it("provides an image upload handler to the rich editor", async () => {
  render(<MemoEditor onSubmit={vi.fn().mockResolvedValue(undefined)} />);

  fireEvent.click(screen.getByText("Simulate change"));

  await waitFor(() => {
    expect(shiroEditorMock).toHaveBeenCalledWith(
      expect.objectContaining({ imageUpload: expect.any(Function) }),
      undefined,
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vp test run src/components/memo/memo-editor.test.tsx`
Expected: FAIL because `MemoEditor` does not yet pass `imageUpload`.

- [ ] **Step 3: Write minimal implementation**

Add a small `createObjectUrlForImage(file)` helper in `memo-editor.tsx` and pass it into `ShiroEditor`:

```tsx
async function handleImageUpload(file: File) {
  return URL.createObjectURL(file);
}

<ShiroEditor imageUpload={handleImageUpload} />
```

Do not add persistence or backend upload logic in this pass.

- [ ] **Step 4: Run test to verify it passes**

Run: `vp test run src/components/memo/memo-editor.test.tsx`
Expected: PASS

### Task 5: Install the new package and run project verification

**Files:**
- Modify: `memos/package.json`
- Modify: `memos/pnpm-lock.yaml`
- Test: `memos/src/components/memo/memo-editor.test.tsx`
- Test: `memos/src/components/memo/memos-page.test.tsx`

- [ ] **Step 1: Add the dependency**

Run: `vp add @haklex/rich-kit-shiro@0.0.93`
Expected: `package.json` and lockfile updated with the new dependency.

- [ ] **Step 2: Run focused tests**

Run: `vp test run src/components/memo/memo-editor.test.tsx src/components/memo/memos-page.test.tsx`
Expected: PASS

- [ ] **Step 3: Run project checks**

Run: `vp check`
Expected: PASS with formatting, linting, and type checks clean.

- [ ] **Step 4: Run the full test suite**

Run: `vp test run`
Expected: PASS
