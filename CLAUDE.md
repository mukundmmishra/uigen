# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # install deps, generate Prisma client, run migrations
npm run dev            # start dev server (Turbopack)
npm run build          # production build
npm test               # run all tests (watch mode)
npx vitest run         # run tests once
npx vitest run src/lib/__tests__/file-system.test.ts  # single test file
npm run db:reset       # reset database (destructive — confirm first)
npx prisma generate    # re-generate client after schema changes
npx prisma migrate dev --name <migration-name>        # new migration
```

## Environment Variables

| Variable | Required | Notes |
|----------|----------|-------|
| `ANTHROPIC_API_KEY` | No | Without it, `MockLanguageModel` is used (hardcoded demo responses) |
| `JWT_SECRET` | No | Falls back to `"development-secret-key"` in dev |

Database: SQLite at `prisma/dev.db`. Prisma client generated to `src/generated/prisma/`.

## Coding Standards

- TypeScript strict mode — no `any`, prefer explicit return types on exported functions
- Use `@/` path alias for all imports (never relative `../../../`)
- React components: named exports only, no default exports
- Server-only modules must import `server-only` at the top
- Vitest test files that call server-only code require `// @vitest-environment node` at line 1

## Rules

- **NEVER** run `npm audit fix --force` without explicit user approval
- **NEVER** run `db:reset` without confirming with the user — it drops all data
- **Do NOT** add `h-full` to the inner wrapper `<div>` inside Radix `ScrollArea` — it breaks height propagation; render empty states outside `ScrollArea` instead
- **Do NOT** use Unix-style `NODE_OPTIONS='...'` in npm scripts — use `cross-env` for Windows compatibility

## Workflow

- After editing `prisma/schema.prisma`, run `npx prisma generate` then create a migration
- After pulling changes that include schema updates, run `npm run setup`
- `node-compat.cjs` is loaded via `NODE_OPTIONS` in all scripts — required for Node 25+ SSR compatibility (deletes `globalThis.localStorage`/`sessionStorage`)

## Architecture

UIGen is an AI-powered React component generator. Users describe components in chat; the AI writes code into a client-side virtual file system; results preview live in a sandboxed iframe.

### Request / Data Flow

1. `MessageInput` → `ChatContext` sends `POST /api/chat` with `{ messages, files: vfs.serialize(), projectId }`
2. `/api/chat/route.ts` reconstructs the VFS, calls `streamText()` with `claude-haiku-4-5` and two tools
3. AI calls tools (`str_replace_editor`, `file_manager`) to mutate files; each streams back to the client
4. `FileSystemContext.handleToolCall()` applies mutations to the VFS and increments `refreshTrigger`
5. `PreviewFrame` transpiles VFS files with `@babel/standalone`, builds an import map (`blob:` URLs for local, `esm.sh` for third-party), sets result as `srcdoc` on a sandboxed iframe
6. On finish, if authenticated, messages + VFS state persist to the `Project` row in SQLite

### Key Abstractions

| Abstraction | File | Purpose |
|---|---|---|
| `VirtualFileSystem` | `src/lib/file-system.ts` | In-memory `Map<string, FileNode>`; `serialize()` / `deserializeFromNodes()` for persistence |
| `FileSystemContext` | `src/lib/contexts/file-system-context.tsx` | Owns VFS, exposes `handleToolCall()` |
| `ChatContext` | `src/lib/contexts/chat-context.tsx` | Wraps `useChat`, injects VFS into requests, routes tool responses |
| `str_replace_editor` | `src/lib/tools/str-replace.ts` | AI tool: `view`, `create`, `str_replace`, `insert` |
| `file_manager` | `src/lib/tools/file-manager.ts` | AI tool: `rename`, `delete` |
| Preview Pipeline | `src/lib/transform/jsx-transformer.ts` | Babel JSX transform + import map; entry point `/App.jsx` |
| Auth | `src/lib/auth.ts` | JWT HS256, 7-day expiry, `httpOnly` cookie `auth-token` |
| `MockLanguageModel` | `src/lib/provider.ts` | Scripted demos when `ANTHROPIC_API_KEY` absent |

### Pages

- `/` — redirects authenticated users to most recent project (or creates one); shows empty state for anonymous
- `/[projectId]` — loads project, renders `MainContent` split-pane (chat left, preview/code right)

## Testing

Vitest + jsdom + Testing Library. Test files live in `__tests__/` alongside source.

```bash
npx vitest run --reporter=verbose  # verbose output
```
