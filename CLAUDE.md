# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator. Users describe UI components in a chat interface, and Claude generates/edits files in a virtual file system that renders live in a preview pane. Built with Next.js 15 App Router, React 19, TypeScript, Tailwind CSS v4, Prisma/SQLite, and the Anthropic AI SDK.

## Commands

```bash
npm run setup       # First-time setup: install deps + prisma generate + migrate
npm run dev         # Start dev server with Turbopack on :3000
npm run build       # Production build
npm start           # Run production server
npm run lint        # ESLint via Next.js
npm test            # Run Vitest tests (jsdom environment)
npm run db:reset    # Drop and re-run SQLite migrations
```

Run a single test file:
```bash
npx vitest run src/path/to/__tests__/file.test.ts
```

## Architecture

### Request Flow

1. User sends a message → `src/app/api/chat/route.ts` streams a response from Claude (`claude-haiku-4-5`)
2. Claude issues tool calls (`str_replace_editor`, `file_manager`) to modify the virtual file system
3. `FileSystemContext` (`src/lib/contexts/FileSystemContext.tsx`) processes those tool calls and updates in-memory state
4. The preview iframe (`src/components/preview/`) picks up file changes and re-renders using a Babel JSX transformer (`src/lib/transform/`)

### Key Abstractions

- **Virtual File System** (`src/lib/file-system.ts`): Pure in-memory FS — no disk writes. Holds all generated component files.
- **ChatContext** (`src/lib/contexts/ChatContext.tsx`): Wraps `@ai-sdk/react` `useChat`. Manages streaming chat state and relays tool-call results to FileSystemContext.
- **AI Provider** (`src/lib/provider.ts`): Returns an Anthropic model or a mock provider (when `ANTHROPIC_API_KEY` is unset). The mock returns canned responses.
- **AI Tools** (`src/lib/tools/`): `str_replace_editor` (view/create/str_replace/insert) and `file_manager` (rename/delete). Tool schemas are passed to the API route and their execution lives in FileSystemContext.
- **System Prompt** (`src/lib/prompts/`): Sent with `cacheControl: { type: "ephemeral" }` to enable prompt caching.

### Auth & Persistence

- Unauthenticated users work anonymously (browser `localStorage`); no DB writes.
- Authenticated users: JWT sessions (via `src/lib/auth.ts`, bcrypt), projects persisted via Prisma (`User`, `Project` models in `prisma/schema.prisma`).
- `src/middleware.ts` guards `/api/projects` and `/api/filesystem` routes.
- Server actions in `src/actions/` handle auth and project CRUD.

### Path Alias

`@/*` resolves to `src/*` (configured in `tsconfig.json`).

## Environment Variables

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Enables real Claude responses; omit to use the mock provider |
| `JWT_SECRET` | Secret for signing JWT session tokens |
| `DATABASE_URL` | SQLite path (defaults to `file:./dev.db`) |

## Testing

Tests live in `__tests__/` subfolders co-located with source. Uses Vitest + jsdom + `@testing-library/react`. Coverage focuses on `ChatInterface`, `MessageList`, `FileTree`, `file-system`, and `jsx-transformer`.
