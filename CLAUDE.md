# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install deps, generate Prisma client, run migrations
npm run dev            # Development server with Turbopack
npm run build          # Production build
npm run lint           # ESLint check
npm run test           # Run all tests with Vitest
npx vitest run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run db:reset       # Reset database (destructive)
```

Set `ANTHROPIC_API_KEY` in `.env` to enable real AI generation; without it the app returns static mock responses.

## Architecture

UIGen is a Next.js 15 (App Router) full-stack app that lets users generate React components via chat and preview them live.

### Three-panel layout
- **Left:** Chat interface (`src/components/chat/`) — uses Vercel AI SDK's `useChat` hook, streaming from `/api/chat`
- **Right top:** Live preview iframe (`src/components/preview/`) — renders generated code via `src/lib/transform/jsx-transformer.ts`, which builds an importmap-based HTML document
- **Right bottom:** File tree + Monaco editor (`src/components/editor/`)

### Data flow
1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route calls the Anthropic model (or mock) with two tools: `str_replace_editor` (create/modify files) and `file_manager` (rename/delete)
3. Tool calls mutate the virtual file system, which is streamed back as data parts
4. On the client, `FileSystemContext` (`src/lib/contexts/`) reconstructs the in-memory `VirtualFileSystem` (`src/lib/file-system.ts`) and updates React state
5. The preview component re-renders the iframe whenever files change

### Key abstractions
- **`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory file system; no disk writes. Serializes to/from JSON for database persistence.
- **`provider.ts`** (`src/lib/provider.ts`) — returns the Anthropic model when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that returns hardcoded example components.
- **`jsx-transformer.ts`** (`src/lib/transform/`) — converts virtual FS files into a self-contained iframe HTML document using ES module importmaps for React/ReactDOM.
- **`ChatContext` / `FileSystemContext`** (`src/lib/contexts/`) — client-side state; chat wraps `useChat`, file system tracks current files and selected file.

### Persistence
- Prisma + SQLite. Schema defined in `prisma/schema.prisma` — always reference it when working with DB operations.
- Models: `User` (accounts) and `Project` (stores serialized messages + file system JSON per user).
- Anonymous users can work without signing in; projects are only persisted for authenticated users.
- Server actions in `src/actions/` handle all DB operations.

### Routing
- `/` (`src/app/page.tsx`) — home; redirects authenticated users to their first project
- `/[projectId]` (`src/app/[projectId]/page.tsx`) — main workspace
- `/api/chat` — streaming chat endpoint

### Testing
Tests live alongside source in `__tests__/` subdirectories. Uses Vitest with jsdom and React Testing Library. Path alias `@/*` → `src/*` works in tests via `vitest.config.mts`.
