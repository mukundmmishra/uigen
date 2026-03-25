# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies, generate Prisma client, run migrations
npm run setup

# Start dev server (Turbopack)
npm run dev

# Build for production
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Reset database (destructive)
npm run db:reset

# Re-generate Prisma client after schema changes
npx prisma generate

# Run a new migration
npx prisma migrate dev --name <migration-name>
```

## Environment Variables

| Variable | Required | Notes |
|----------|----------|-------|
| `ANTHROPIC_API_KEY` | No | Without it, a `MockLanguageModel` is used (hardcoded demo responses) |
| `JWT_SECRET` | No | Falls back to `"development-secret-key"` in dev |

Database is SQLite at `prisma/dev.db`. Prisma client is generated to `src/generated/prisma/`.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat; the AI writes code into a client-side virtual file system; results are previewed live in a sandboxed iframe.

### Request / Data Flow

1. User types in `MessageInput` → `ChatContext` (wraps Vercel AI SDK `useChat`) sends `POST /api/chat` with `{ messages, files: vfs.serialize(), projectId }`
2. `/api/chat/route.ts` reconstructs the VFS, calls `streamText()` with `claude-haiku-4-5` and two tools
3. AI calls tools (`str_replace_editor`, `file_manager`) to mutate files; each tool call streams back to the client
4. `FileSystemContext.handleToolCall()` applies mutations to the client-side VFS and increments `refreshTrigger`
5. `PreviewFrame` detects the trigger, transpiles all VFS files with `@babel/standalone`, builds an import map (local files → `blob:` URLs, third-party → `esm.sh`), and sets the result as `srcdoc` on a sandboxed iframe
6. On finish, if authenticated, messages + VFS state are persisted to the `Project` row in SQLite

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — In-memory file tree as a `Map<string, FileNode>`. Never writes to disk for generated components. `serialize()` / `deserializeFromNodes()` convert to/from JSON for persistence and API transport.

**`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — React context that owns the VFS instance and exposes `handleToolCall()`. Initialized from `project.data` on page load.

**`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — Wraps `useChat`, injects VFS state into every request, and routes tool call responses to `FileSystemContext`.

**AI Tools** — Two tools registered server-side with `streamText`:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`): `view`, `create`, `str_replace`, `insert`
- `file_manager` (`src/lib/tools/file-manager.ts`): `rename`, `delete`

**Preview Pipeline** (`src/lib/transform/jsx-transformer.ts`) — Babel-in-browser JSX transform + import map. Entry point is `/App.jsx`. CSS files become `<style>` blocks. Tailwind is loaded from CDN.

**Auth** (`src/lib/auth.ts`) — Custom JWT (`HS256`, 7-day expiry) stored in an `httpOnly` cookie `auth-token`. No third-party auth library. Middleware protects `/api/projects` and `/api/filesystem`; `/api/chat` checks auth only for the DB-persist step.

**Anonymous Work Tracking** (`src/lib/anon-work-tracker.ts`) — Buffers unauthenticated messages + VFS in `sessionStorage`. On sign-in/sign-up the buffer is saved as a new project and the user is redirected to it.

**`MockLanguageModel`** (`src/lib/provider.ts`) — Activated when `ANTHROPIC_API_KEY` is absent. Produces scripted Counter/Form/Card demos across 4 steps; useful for UI development without API costs.

### Pages

- `/` — Redirects authenticated users to their most recent project (or creates one); shows empty `MainContent` for anonymous users
- `/[projectId]` — Loads a project (auth required) and renders `MainContent`

Both routes render the same `MainContent` split-pane layout: chat panel (left, resizable) + preview/code tabs (right).

### Node Compatibility Shim

`node-compat.cjs` is required via `NODE_OPTIONS` in all scripts. It deletes `globalThis.localStorage` and `globalThis.sessionStorage` on the server to prevent SSR crashes under Node.js 25+.

## Testing

Tests use Vitest + jsdom + Testing Library. Test files live alongside source in `__tests__/` subdirectories.

```bash
npx vitest run           # run once
npx vitest               # watch mode
npx vitest run --reporter=verbose
```
