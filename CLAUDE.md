# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude generates JSX/TSX files into a virtual in-memory file system; the results are previewed live in an iframe via Babel + importmap + blob URLs — no files ever written to disk.

## Commands

```sh
npm run setup          # First-time: install deps + prisma generate + migrate
npm run dev            # Dev server at localhost:3000 (Turbopack)
npm run build          # Production build
npm run test           # Run all Vitest tests
npm run lint           # ESLint
npm run db:reset       # Reset the SQLite database (destructive)
```

To run a single test file:
```sh
npx vitest run src/lib/__tests__/file-system.test.ts
```

Prisma operations (after schema changes):
```sh
npx prisma migrate dev    # Create + apply migration
npx prisma generate       # Regenerate client (output: src/generated/prisma/)
```

## Architecture

### Request Flow

1. User types a message → `ChatInterface` calls `useChat` (Vercel AI SDK)
2. POST to `/api/chat` with `{ messages, files, projectId }` — `files` is the serialized `VirtualFileSystem`
3. Server reconstructs `VirtualFileSystem`, calls `streamText` with two tools:
   - `str_replace_editor` — create/str_replace/insert operations on files
   - `file_manager` — rename/delete operations
4. AI tool calls stream back to the client; `ChatContext` dispatches each tool call to `FileSystemContext.handleToolCall()`
5. `FileSystemContext` mutates the client-side `VirtualFileSystem` and increments `refreshTrigger`
6. `PreviewFrame` reacts to `refreshTrigger`, calls `createImportMap()` (Babel transforms all files to blob URLs), injects an importmap into an iframe, and renders `/App.jsx` as the entry point

### Key Files

| File | Purpose |
|------|---------|
| `src/lib/file-system.ts` | `VirtualFileSystem` class — all FS operations, serialize/deserialize |
| `src/lib/transform/jsx-transformer.ts` | Babel transforms JSX→JS; builds import map + blob URLs for iframe preview |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping `VirtualFileSystem`; handles AI tool call dispatch |
| `src/lib/contexts/chat-context.tsx` | Wraps Vercel AI SDK `useChat`; sends serialized FS state with each message |
| `src/app/api/chat/route.ts` | AI streaming endpoint; persists to DB on finish if user is authenticated |
| `src/lib/prompts/generation.tsx` | System prompt — instructs Claude to always create `/App.jsx` as entrypoint, use `@/` import alias |
| `src/lib/tools/str-replace.ts` | `buildStrReplaceTool()` — Vercel AI SDK tool wrapping VirtualFileSystem text edit ops |
| `src/lib/tools/file-manager.ts` | `buildFileManagerTool()` — Vercel AI SDK tool for rename/delete |
| `src/lib/auth.ts` | JWT session via `jose`; uses `auth-token` httpOnly cookie |
| `src/lib/prisma.ts` | Prisma singleton client |
| `src/middleware.ts` | Route protection — redirects unauthenticated users away from `/[projectId]` |

### Data Models (SQLite via Prisma)

- **User**: `id`, `email`, `password` (bcrypt), `projects[]`
- **Project**: `id`, `name`, `userId?`, `messages` (JSON string), `data` (JSON string of serialized VirtualFileSystem)

Anonymous users can use the app without a session; projects only persist for authenticated users.

### Preview Mechanism

`createImportMap()` in `jsx-transformer.ts`:
- Transforms each `.jsx/.tsx/.js/.ts` file with Babel (standalone, in-browser)
- Creates blob URLs for each file and builds an ES import map
- Third-party imports (non-relative, non-`@/`) are routed to `esm.sh`
- Missing local imports get placeholder stub modules (empty React components)
- CSS files are collected and injected as `<style>` tags
- The iframe uses `srcdoc` with `allow-scripts allow-same-origin allow-forms`

### Auth

JWT sessions with `jose`. The `JWT_SECRET` env var is required for production; falls back to `"development-secret-key"` locally. Sessions expire in 7 days. Server actions import from `src/lib/auth.ts` which is marked `server-only`.

### Without an API Key

If `ANTHROPIC_API_KEY` is unset, `getLanguageModel()` returns a mock provider that returns static responses. `maxSteps` is reduced to 4 in this mode.

## Testing

Tests live in `__tests__/` subdirectories next to the code they test. Uses Vitest + jsdom + Testing Library. The `vitest.config.mts` includes `vite-tsconfig-paths` so `@/` aliases resolve in tests.
