# PDF Attachment Flow Analysis

## Current Attachment Flow

### 1. Entry Points

**CLI (`packages/coding-agent/src/cli/file-processor.ts:32-87`):**
- Handles `@file` arguments
- Only recognizes images by extension (`.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`)
- **PDFs fall into the "text file" case** and are read with `readFileSync(absolutePath, "utf-8")` - which would corrupt binary PDFs or fail

**Web UI (`packages/web-ui/src/utils/attachment-utils.ts:29-201`):**
- `loadAttachment()` detects file type by mime type or extension
- For PDFs → calls `processPdf()` using `pdfjs-dist` to extract text
- Returns `Attachment` with populated `extractedText` field

### 2. Attachment Type (`packages/agent/src/types.ts:14-23`)

```typescript
interface Attachment {
  type: "image" | "document";
  content: string;         // base64 encoded binary
  extractedText?: string;  // For documents (extracted client-side)
}
```

### 3. Message Transformation (`packages/agent/src/agent.ts:10-51`)

The `defaultMessageTransformer()` converts attachments to content blocks:

| Attachment Type | Has extractedText? | Result |
|----------------|-------------------|--------|
| `image` | n/a | → `ImageContent` (binary passed to provider) |
| `document` | ✅ Yes | → `TextContent` with `[Document: filename]\n{extractedText}` |
| `document` | ❌ No | → **Silently dropped** |

### 4. Provider Layer (`packages/ai/src/providers/anthropic.ts:407-549`)

- `convertMessages()` only handles `TextContent` and `ImageContent`
- No `DocumentContent` type exists
- Images filtered if `!model.input.includes("image")`

### 5. Type System (`packages/ai/src/types.ts`)

- `UserMessage.content: string | (TextContent | ImageContent)[]`
- `Model.input: ("text" | "image")[]` - no "document" capability

---

## Key Finding

**PDFs are ALWAYS converted to text client-side.** The binary PDF is stored in `attachment.content` but never sent to the LLM. Only the `extractedText` (from pdfjs-dist extraction) reaches the model as a plain text block.

This means:
1. PDFs work in web-ui (has pdfjs-dist for extraction)
2. PDFs partially broken in CLI (no extraction, treated as text files)
3. Native PDF capabilities of Claude/Gemini are unused
4. Visual elements in PDFs (charts, images, layouts) are lost

---

## Author's Decision: No Native PDF Support

Per [GitHub issue #204](https://github.com/badlogic/pi-mono/issues/204), the library author has decided **against** adding native PDF processing to the server/provider layer.

**Key arguments:**
- PDF extraction is "a huge task in itself" and distracts from the core mission of effectively using LLMs
- Users need control over the conversion pipeline since "PDFs are hard"
- Different extraction tools (pdftotext, markitdown, etc.) produce varying quality results
- Native LLM PDF capabilities have their own scalability limitations
- "You want to be in control of the full PDF → text pipeline"

**Result:** Native server-side PDF processing will not be added. The current client-side text extraction approach remains.

---

## Files Involved

- `packages/coding-agent/src/cli/file-processor.ts` - CLI @file handling
- `packages/coding-agent/src/main.ts` - CLI entry point
- `packages/agent/src/types.ts` - Attachment type definition
- `packages/agent/src/agent.ts` - defaultMessageTransformer
- `packages/web-ui/src/utils/attachment-utils.ts` - loadAttachment, processPdf
- `packages/ai/src/types.ts` - TextContent, ImageContent, UserMessage, Model
- `packages/ai/src/providers/anthropic.ts` - convertMessages, convertContentBlocks
- `packages/ai/src/providers/google.ts` - Google provider (similar structure)
