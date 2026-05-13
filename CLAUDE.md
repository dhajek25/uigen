# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project layout

The git root is `uigen/` (the repo root), but the entire Next.js application lives one level deeper in `uigen/uigen/`. **Run all commands from `uigen/uigen/`.**

```
uigen/           ← git root (CLAUDE.md lives here)
└── uigen/       ← Next.js app root (package.json, prisma/, src/, …)
```

The Prisma schema is at `uigen/prisma/schema.prisma` (relative to the git root).

## Commands

All commands must be run from the `uigen/uigen/` directory.

```bash
npm run dev          # Dev server with Turbopack
npm run build        # Production build
npm run start        # Production server
npm run lint         # ESLint
npm run test         # Vitest (all tests)
npm run setup        # First-time setup: install + prisma generate + migrate
npm run db:reset     # Reset and re-seed the database
```

Run a single test file:
```bash
npx vitest run src/components/chat/ChatInterface.test.tsx
```

## Architecture

UIGen is a Next.js 15 full-stack app where users chat with Claude to generate React components that render in a live preview panel.

### Request flow

1. User types a prompt → `ChatContext` sends it to `POST /api/chat`
2. `api/chat/route.ts` streams a response via the Vercel AI SDK, calling Claude with two tools: `str_replace_editor` (edit files) and `file_manager` (create/delete/list)
3. Tool calls mutate the **virtual file system** (`FileSystemContext` / `src/lib/file-system.ts`) — an in-memory store, no disk writes
4. The preview panel watches `FileSystemContext`, transpiles JSX with `@babel/standalone`, and injects it into an `<iframe>`

### Key directories

| Path | What lives here |
|------|-----------------|
| `src/app/` | Next.js pages and the `/api/chat` route |
| `src/app/main-content.tsx` | Root split-view layout (35% chat / 65% editor+preview) |
| `src/actions/` | Server actions — auth (sign-up/in/out) and project CRUD |
| `src/components/chat/` | Chat UI: message list, input, markdown renderer |
| `src/components/editor/` | Monaco editor + file tree |
| `src/components/preview/` | iframe-based live preview |
| `src/components/ui/` | shadcn/ui primitives |
| `src/lib/file-system.ts` | In-memory virtual FS — the shared state Claude's tools mutate |
| `src/lib/tools/` | AI tool definitions (`str_replace_editor`, `file_manager`) |
| `src/lib/transform/` | Babel JSX-to-JS transform for the preview |
| `src/lib/contexts/` | `FileSystemContext` and `ChatContext` React providers |
| `src/lib/prompts/` | System prompt that shapes Claude's component-generation behavior |
| `src/lib/provider.ts` | AI provider setup (Anthropic `claude-haiku-4-5` with mock fallback) |
| `prisma/` | SQLite schema — `User` and `Project` (stores messages + VFS snapshot as JSON) |

### Auth & sessions

JWT sessions via `jose`, stored in HTTP-only cookies. Auth logic is in `src/actions/auth.ts` and protected via middleware. Passwords are bcrypt-hashed.

### State management

- `FileSystemContext` — canonical in-memory file store; editor and preview both read from it
- `ChatContext` — chat message history and streaming state
- Project persistence: when saving, the full VFS and message list are serialized to JSON and written to the `Project` DB row

### Path alias

`@/*` resolves to `./src/*` (configured in `tsconfig.json` and `vitest.config.mts`).

### Testing

Vitest + jsdom + `@testing-library/react`. Test files sit alongside source files (`*.test.tsx`). The VFS (`file-system.ts`) is tested with pure unit tests; UI components use React Testing Library.
