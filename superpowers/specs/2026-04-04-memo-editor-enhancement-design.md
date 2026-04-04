# Memo Editor Enhancement — Design Spec

## Overview

Replace the current minimal `RichEditor` + custom toolbar implementation in the memos app with `@haklex/rich-kit-shiro` (ShiroEditor), a pre-configured Lexical-based rich editor bundle. This adds floating toolbar, slash menu, block handles, and rich content types (images, code blocks, KaTeX math, alerts/callouts, links) while maintaining backward compatibility with existing memos.

## Decisions Made

| Decision | Choice |
|----------|--------|
| **Goal** | Full treatment: rich content types + premium UX |
| **Interaction model** | Floating toolbar + slash menu + block handles (Notion-style) |
| **Content types** | Core set: images, code blocks, links, KaTeX, alerts/callouts |
| **Visual style** | Minimal & clean — whitespace-focused, subtle borders, soft shadows |
| **Implementation** | Use `@haklex/rich-kit-shiro` bundle |

## Architecture

### Component Changes

#### `memo-editor.tsx` — Full Rewrite

Remove:
- Custom `ToolbarButton` component
- `ToolbarState` interface and `syncToolbarState` logic
- `formatText()`, `toggleList()`, `setBlockType()` action functions
- `getSelectionBlockType()`, `transformSelectedBlocks()` helpers
- All Lexical command imports (`FORMAT_TEXT_COMMAND`, `INSERT_ORDERED_LIST_COMMAND`, etc.)
- The top toolbar div with all button groups

Add:
- `ShiroEditor` from `@haklex/rich-kit-shiro`
- `imageUpload` handler (paste/drag-drop → base64 or URL)
- Minimal wrapper for styling and submit footer

Props preserved:
- `onSubmit: (draft) => Promise<void>`
- `initialContent?: SerializedEditorState | null`
- `placeholder?: string`
- `autoFocus?: boolean`
- `compact?: boolean`

#### `memo-content.tsx` — One-Line Swap

```tsx
// Before
import { RichRenderer } from "@haklex/rich-static-renderer";
<RichRenderer value={content} variant="note" theme={theme} />

// After
import { ShiroRenderer } from "@haklex/rich-kit-shiro";
<ShiroRenderer value={content} variant="note" theme={theme} />
```

#### `memo-draft.ts` — No Changes

`extractTextContent` from `@haklex/rich-editor/static` continues to work with ShiroEditor's serialized state format. No migration needed.

#### `package.json` — Add Dependency

Add `@haklex/rich-kit-shiro` at version `0.0.93` (matching existing Haklex packages).

### Data Flow (Unchanged)

```
ShiroEditor onChange → setContent → deriveMemoDraft(content)
  ├── plainText: extractTextContent(content)
  ├── excerpt: first 140 chars
  └── tags: #hashtags extracted from plainText

onSubmit → API call → memo saved
ShiroRenderer → renders saved content in memo-card
```

## Visual Styling

### CSS Override Strategy

Three layers:
1. ShiroEditor's built-in Vanilla Extract CSS (base)
2. CSS variables via wrapper class (colors, spacing, fonts)
3. Targeted CSS overrides for specific elements

### Style Tokens

| Token | Value |
|-------|-------|
| Editor background | Transparent (inherits from parent card) |
| Border | 1px solid `var(--border-color)` with 12px radius |
| Padding | 24px |
| Font | `system-ui, -apple-system, sans-serif` |
| Font size | 15px |
| Line height | 1.6 |
| Floating toolbar | Dark bg (#1a1a1a), 8px radius, soft shadow |
| Slash menu | White bg, 10px radius, elevated shadow |
| Block handles | Hidden by default, visible on hover (#a3a3a3) |

### Block-Specific Styling

| Block | Treatment |
|-------|-----------|
| Code blocks | Dark bg (#1e1e1e), rounded, language label, copy button |
| Alerts/Callouts | Light colored bg, 3px left accent border, icon + title |
| KaTeX | Centered, light bg container, generous padding |
| Images | Rounded corners, subtle border, caption, lightbox |
| Links | Inline: accent color underline. Cards: rounded preview |

### Image Upload

- **Phase 1**: Paste and drag-drop with base64 data URLs (no backend changes)
- **Phase 2** (future): Wire to upload API endpoint if one becomes available

## Error Handling

| Scenario | Handling |
|----------|----------|
| Image upload fails | Inline error in block, "Retry" button |
| KaTeX parse error | Render error message with raw LaTeX for editing |
| Invalid saved content | Fall back to empty state, log warning |
| Large memo (>100KB) | Character count warning in footer |
| Dark mode toggle | Automatic via CSS variables |

## Backward Compatibility

ShiroEditor uses the same Lexical serialized state format as the current RichEditor. Existing memos render correctly through ShiroRenderer without migration. New block types simply won't be present in old content.

## Performance

- **Bundle size**: ~200-300KB gzipped (all plugins + renderers)
- **Lazy loading**: Heavy nodes (KaTeX, Mermaid, code highlighting) lazy-loaded internally
- **Static rendering**: ShiroRenderer is tree-shaken — memo cards only load needed renderers
- **Images**: Blurhash placeholders for large images (built into Haklex)

## File Changes Summary

| File | Change |
|------|--------|
| `src/components/memo/memo-editor.tsx` | Rewrite: replace RichEditor + custom toolbar with ShiroEditor |
| `src/components/memo/memo-content.tsx` | Swap RichRenderer → ShiroRenderer |
| `src/lib/memo-draft.ts` | No changes |
| `package.json` | Add `@haklex/rich-kit-shiro` dependency |
| `src/styles/editor-overrides.css` (new) | CSS overrides for minimal aesthetic |

## Testing

- Update existing memo-editor tests to work with ShiroEditor
- Verify backward compatibility: existing memos render correctly
- Test image paste/drag-drop flow
- Test slash menu and floating toolbar interactions
- Verify dark mode compatibility
