# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered, browser-based React component generator. Users describe UI in natural language; the AI (Claude) calls tools to write files into an in-memory virtual file system; those files are transpiled by Babel in the browser and rendered live in an `<iframe>`. Nothing is written to disk at runtime.

## Setup & Commands

```bash
npm run setup       # First-time: install deps + prisma generate + migrate
npm run dev         # Dev server at http://localhost:3000 (uses Turbopack)
npm run build       # Production build
npm test            # Run all tests with Vitest
npx vitest run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run lint        # ESLint via next lint
npm run db:reset    # Reset SQLite DB (prisma migrate reset --force)
```

Requires `ANTHROPIC_API_KEY` in `.env` for real AI responses. Without it, `lib/provider.ts` returns a `MockLanguageModel` that generates static Counter/Form/Card component code.

## Architecture

### Request Flow

1. User submits a prompt → `ChatContext` (`src/lib/contexts/chat-context.tsx`) calls `POST /api/chat` with message history + serialized virtual FS
2. `src/app/api/chat/route.ts` deserializes the FS, calls `streamText` (Vercel AI SDK) with Claude (`claude-haiku-4-5`) and two tools: `str_replace_editor` and `file_manager`
3. Tool call chunks stream back; `onToolCall` in `ChatContext` → `FileSystemContext.handleToolCall` applies file operations (create/edit/rename/delete) to the in-memory `VirtualFileSystem`
4. `PreviewFrame` watches a `refreshTrigger`. On change, it runs all FS files through `createImportMap` (Babel + blob URLs + esm.sh for third-party packages) and sets the resulting HTML as `<iframe srcdoc>`
5. On finish, if authenticated with a `projectId`, messages and FS state are saved to SQLite via Prisma

### Key Abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`): In-memory file tree. Implements text-editor commands (`viewFile`, `createFileWithParents`, `replaceInFile`, `insertInFile`) used by the AI tools. Serialize/deserialize for DB persistence and API transport.
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): Wraps `VirtualFileSystem` in React state. `handleToolCall` is the bridge from AI streaming → FS mutations → UI refresh.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK `useChat`, passes serialized FS in request body, delegates tool calls to `FileSystemContext`.
- **`jsx-transformer.ts`** (`src/lib/transform/`): Uses `@babel/standalone` to transpile JSX/TSX to blob URLs in the browser. Resolves `@/` path aliases, maps third-party imports to `esm.sh`. Called by `PreviewFrame` to generate the iframe `srcdoc`.
- **AI tools** (`src/lib/tools/`): `str_replace_editor` (view/create/str_replace/insert) and `file_manager` (rename/delete). Defined as Vercel AI SDK tool schemas; their execution in `route.ts` calls `VirtualFileSystem` methods.
- **Generation system prompt** (`src/lib/prompts/generation.tsx`): Instructs the AI to always start with `/App.jsx` and use `@/` import aliases.

### Auth

JWT-based (jose, HS256, 7-day httpOnly cookies). Passwords bcrypt-hashed. Middleware only protects `/api/projects` and `/api/filesystem`—not `/api/chat`. Anonymous users get session-tracked work via `sessionStorage` (`lib/anon-work-tracker.ts`).

### Database

Prisma + SQLite. Schema is defined at `src/generated/prisma/schema.prisma`. Two models: `User` and `Project`. `Project.data` stores the serialized virtual FS; `Project.messages` stores AI conversation history (both as JSON strings). Prisma client is generated into `src/generated/prisma`.

## Tech Stack

- Next.js 15 (App Router, Server Actions), React 19, TypeScript
- Tailwind CSS v4, shadcn/ui (Radix UI primitives)
- Vercel AI SDK (`ai` + `@ai-sdk/anthropic`), `@babel/standalone`
- Prisma 6 + SQLite, Monaco Editor
- Vitest + React Testing Library (jsdom)

## Node Compatibility

`node-compat.cjs` is required at startup via `NODE_OPTIONS='--require ./node-compat.cjs'` (all npm scripts include this). It deletes `globalThis.localStorage/sessionStorage` to fix a Node 25 SSR regression. Do not remove it from scripts.
