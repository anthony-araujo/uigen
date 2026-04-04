# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup
npm run dev
npm run build
npm test
npx vitest run src/lib/__tests__/file-system.test.ts
npm run lint
npm run db:reset
```

## Environment

Copy `.env` and add your Anthropic API key:
```
ANTHROPIC_API_KEY=your-api-key-here
```

Without an API key, the app uses a `MockLanguageModel` (defined in `src/lib/provider.ts`) that returns static pre-scripted component generation flows instead of calling Claude.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude generates working code displayed in a live preview and editable via Monaco Editor.

### Request flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route reconstructs a `VirtualFileSystem` from the serialized `files` payload, then calls `streamText` (Vercel AI SDK) with two tools: `str_replace_editor` and `file_manager`
3. Claude uses those tools to create/modify files in the in-memory `VirtualFileSystem`
4. The streaming response is piped back to the client via `result.toDataStreamResponse()`
5. On finish, if the user is authenticated and a `projectId` exists, messages and file system state are persisted to SQLite via Prisma

### Virtual File System (`src/lib/file-system.ts`)

`VirtualFileSystem` is an in-memory tree of `FileNode` objects. No files are ever written to disk. It serializes to/from plain objects (JSON-safe) for persistence and transport. Claude interacts with it exclusively through the two tools:

- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`) — view and edit file contents via string replacement
- **`file_manager`** (`src/lib/tools/file-manager.ts`) — create, delete, and move files/directories

### Preview (`src/components/preview/PreviewFrame.tsx`)

Components are compiled in-browser using `@babel/standalone` and rendered inside an iframe. The iframe receives serialized file system contents and hot-reloads on every change.

### AI provider (`src/lib/provider.ts`)

`getLanguageModel()` returns either:
- `claude-haiku-4-5` via `@ai-sdk/anthropic` (when `ANTHROPIC_API_KEY` is set)
- `MockLanguageModel` — a hand-written class that simulates multi-step tool-call sequences for local dev/testing without an API key

### Authentication

Session-based auth using JWTs (jose). Passwords hashed with bcrypt. `src/middleware.ts` protects routes. Users can operate anonymously (no sign-up required); projects are only persisted to the DB for authenticated users.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models:
- `User` — email + hashed password
- `Project` — stores `messages` (JSON conversation history) and `data` (serialized `VirtualFileSystem`)

### State management

Two React contexts:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the client-side `VirtualFileSystem` instance
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — owns conversation messages and streaming state

The main layout (`src/app/main-content.tsx`) is a resizable split panel: chat on the left, preview/code editor on the right.
