# CLAUDE.md - AI Assistant Guide

This document provides comprehensive guidance for AI assistants working on the Coding Agent Template codebase. It covers architecture, conventions, workflows, and critical rules.

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Development Setup](#development-setup)
- [Code Conventions](#code-conventions)
- [Critical Security Rules](#critical-security-rules)
- [Common Development Tasks](#common-development-tasks)
- [Database Management](#database-management)
- [Testing and Quality Assurance](#testing-and-quality-assurance)
- [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)

---

## Project Overview

### What This Is

A multi-tenant SaaS platform that enables users to execute AI-powered coding tasks in isolated Vercel Sandbox environments. The platform supports 6 different AI coding agents (Claude Code, OpenAI Codex, GitHub Copilot CLI, Cursor CLI, Google Gemini CLI, and opencode).

### Key Features

- **Multi-user authentication**: GitHub and Vercel OAuth
- **Isolated execution**: Each task runs in a Vercel Sandbox
- **Multiple AI agents**: Choose from 6 different coding assistants
- **Git integration**: Automatic branch creation, commits, and PRs
- **Real-time logging**: Stream agent output to UI
- **MCP server support**: Extend Claude/Codex with Model Context Protocol servers
- **File editing**: In-browser Monaco editor with git diff viewer
- **Repository viewer**: Browse commits, issues, and pull requests

### Tech Stack

- **Framework**: Next.js 15 (App Router) + React 19
- **Language**: TypeScript (strict mode)
- **Database**: PostgreSQL (Neon) + Drizzle ORM
- **Authentication**: Arctic (OAuth) + Jose (JWE sessions)
- **State**: Jotai (atomic state management)
- **UI**: shadcn/ui + Tailwind CSS 4
- **AI**: AI SDK 5 (Vercel AI Gateway integration)
- **Sandbox**: Vercel Sandbox (@vercel/sandbox)
- **Git**: Octokit (GitHub API client)
- **Code Quality**: ESLint + Prettier + TypeScript

---

## Architecture

### Directory Structure

```
/
├── app/                          # Next.js App Router
│   ├── api/                      # API routes
│   │   ├── auth/                 # Authentication endpoints
│   │   ├── tasks/                # Task management
│   │   ├── github/               # GitHub integration
│   │   ├── repos/                # Repository APIs
│   │   ├── connectors/           # MCP connectors
│   │   └── api-keys/             # API key management
│   ├── repos/[owner]/[repo]/     # Repository viewer pages
│   │   ├── commits/page.tsx
│   │   ├── issues/page.tsx
│   │   └── pull-requests/page.tsx
│   ├── tasks/[taskId]/           # Task detail page
│   ├── layout.tsx                # Root layout
│   └── page.tsx                  # Home page
├── components/                   # React components
│   ├── ui/                       # shadcn components
│   ├── auth/                     # Auth components
│   ├── connectors/               # MCP management
│   └── providers/                # Context providers
├── lib/                          # Core library code
│   ├── db/                       # Database (Drizzle)
│   │   ├── schema.ts             # Table definitions
│   │   ├── client.ts             # DB client
│   │   └── migrations/           # SQL migrations
│   ├── sandbox/                  # Vercel Sandbox integration
│   │   ├── agents/               # Agent implementations
│   │   ├── creation.ts           # Sandbox provisioning
│   │   ├── git.ts                # Git operations
│   │   └── commands.ts           # Shell commands
│   ├── auth/                     # Authentication logic
│   ├── session/                  # Session management
│   ├── github/                   # GitHub API client
│   ├── utils/                    # Utility functions
│   ├── atoms/                    # Jotai atoms
│   ├── hooks/                    # React hooks
│   └── crypto.ts                 # Encryption helpers
├── public/                       # Static assets
└── scripts/                      # Utility scripts
```

### Database Schema

**Core Tables:**

1. **`users`**: User profiles and primary OAuth account
   - `id` (PK), `provider`, `externalId`, `accessToken` (encrypted)
   - One-to-many: tasks, connectors, keys, accounts

2. **`tasks`**: User coding tasks
   - `id` (PK), `userId` (FK), `prompt`, `status`, `logs`
   - Stores task configuration, execution state, and results
   - Soft delete via `deletedAt`

3. **`taskMessages`**: Follow-up messages for tasks
   - `id` (PK), `taskId` (FK), `role`, `content`
   - Cascade delete when task is deleted

4. **`accounts`**: Linked OAuth accounts (e.g., GitHub for Vercel users)
   - `id` (PK), `userId` (FK), `provider`, `accessToken` (encrypted)

5. **`keys`**: User-provided API keys
   - `id` (PK), `userId` (FK), `provider`, `value` (encrypted)
   - One key per provider per user

6. **`connectors`**: MCP server configurations
   - `id` (PK), `userId` (FK), `name`, `type`, `baseUrl`, `env` (encrypted)

7. **`settings`**: User preferences
   - `id` (PK), `userId` (FK), `key`, `value`

### Authentication Flow

1. **OAuth Sign-In**:
   ```
   User → /api/auth/signin/{provider}
        → OAuth provider (GitHub/Vercel)
        → /api/auth/{provider}/callback
        → Create/update user in DB (tokens encrypted)
        → Create JWE session cookie
        → Redirect to /
   ```

2. **Session Storage**:
   - Sessions stored as JWE tokens in HTTP-only cookies
   - Server-side validation via `getServerSession()`
   - Client-side context via `SessionProvider`

3. **Token Management**:
   - All tokens encrypted with AES-256-GCM (`ENCRYPTION_KEY`)
   - JWE signed with `JWE_SECRET`
   - Tokens decrypted on-demand, never exposed to client

### Agent Execution Flow

1. **Task Creation** (`POST /api/tasks`):
   - Validate user session
   - Check rate limits
   - Create task in DB (status: `pending`)
   - Generate AI branch name (non-blocking via `after()`)
   - Return task ID

2. **Sandbox Start** (`POST /api/tasks/[taskId]/start-sandbox`):
   - Provision Vercel Sandbox
   - Clone repository
   - Optionally install dependencies
   - Execute agent with instruction
   - Stream logs to client via SSE/WebSocket
   - Commit and push changes
   - Update task status

3. **Follow-Up** (`POST /api/tasks/[taskId]/continue`):
   - Check if sandbox is alive
   - Resume agent session (if supported)
   - Execute follow-up instruction
   - Stream logs and update task

### Key Patterns

**Server Components (default)**:
```typescript
// app/page.tsx
export default async function Home() {
  const session = await getServerSession()
  return <HomePageContent user={session?.user} />
}
```

**Client Components**:
```typescript
'use client'

import { useAtom } from 'jotai'
import { sessionAtom } from '@/lib/atoms/session'

export function TaskChat() {
  const [session] = useAtom(sessionAtom)
  // ...
}
```

**API Routes**:
```typescript
export async function POST(req: Request) {
  const session = await getServerSession()
  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await req.json()
  const validatedData = insertTaskSchema.parse(body)

  const [task] = await db.insert(tasks).values(validatedData).returning()
  return NextResponse.json({ task })
}
```

---

## Development Setup

### Prerequisites

- **Node.js**: 20.x
- **pnpm**: 9.x
- **PostgreSQL**: Any version (Neon recommended)

### Environment Variables

Create `.env.local` with:

```bash
# Database
POSTGRES_URL=postgresql://...

# Vercel Sandbox
SANDBOX_VERCEL_TOKEN=...
SANDBOX_VERCEL_TEAM_ID=...
SANDBOX_VERCEL_PROJECT_ID=...

# Encryption
JWE_SECRET=$(openssl rand -base64 32)
ENCRYPTION_KEY=$(openssl rand -hex 32)

# Auth Providers (at least one required)
NEXT_PUBLIC_AUTH_PROVIDERS=github  # or "vercel" or "github,vercel"

# GitHub OAuth (if using GitHub auth)
NEXT_PUBLIC_GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...

# Vercel OAuth (if using Vercel auth)
NEXT_PUBLIC_VERCEL_CLIENT_ID=...
VERCEL_CLIENT_SECRET=...

# Optional: Global API keys (users can override)
ANTHROPIC_API_KEY=...
OPENAI_API_KEY=...
CURSOR_API_KEY=...
GEMINI_API_KEY=...
AI_GATEWAY_API_KEY=...

# Optional: Configuration
MAX_SANDBOX_DURATION=300  # 5 hours default
MAX_MESSAGES_PER_DAY=5
```

### Installation

```bash
# Clone and install
git clone https://github.com/vercel-labs/coding-agent-template.git
cd coding-agent-template
pnpm install

# Set up database
pnpm db:generate  # Generate migrations
pnpm db:push      # Apply to database

# Start development (DO NOT do this as an AI assistant)
# pnpm dev
```

### Database Commands

```bash
pnpm db:generate   # Generate new migration
pnpm db:push       # Push schema changes
pnpm db:studio     # Open Drizzle Studio
```

---

## Code Conventions

### File Naming

- **Components**: `kebab-case.tsx` (e.g., `task-chat.tsx`)
- **Utilities**: `kebab-case.ts` (e.g., `rate-limit.ts`)
- **API routes**: `route.ts` (Next.js convention)
- **Pages**: `page.tsx`, `layout.tsx`
- **Types**: Inline or in same file (no separate `.types.ts`)

### Import Patterns

```typescript
// ✅ Absolute imports with @ alias
import { db } from '@/lib/db/client'
import { TaskChat } from '@/components/task-chat'
import { cn } from '@/lib/utils'

// ✅ Type imports
import type { Task } from '@/lib/db/schema'

// ✅ Relative imports for local files
import { helper } from './utils'

// ❌ Don't mix relative and absolute
import { db } from '../../lib/db/client'
```

### Export Patterns

```typescript
// ✅ Named exports (preferred)
export function createSandbox() { ... }
export const MAX_DURATION = 300

// ✅ Default exports for pages/components
export default function TaskPage() { ... }

// ✅ Barrel exports
export { executeClaudeInSandbox } from './claude'
export type { AgentExecutionResult } from './types'
```

### TypeScript Conventions

- **Strict mode**: Always enabled (`"strict": true`)
- **No `any`**: Use proper types or `unknown`
- **Zod for validation**: Especially for API inputs
- **Type imports**: Use `import type` for types

```typescript
// ✅ Good
import type { Task } from '@/lib/db/schema'
const task: Task = await fetchTask()

// ✅ Good - Zod validation
const validatedData = insertTaskSchema.parse(body)

// ❌ Bad
const task: any = await fetchTask()
```

### Async/Await Pattern

```typescript
// ✅ Always use try-catch
export async function GET(req: Request) {
  try {
    const data = await fetchData()
    return NextResponse.json({ data })
  } catch (error) {
    console.error('Error:', error)
    return NextResponse.json(
      { error: 'Failed to fetch data' },
      { status: 500 }
    )
  }
}
```

### Next.js 15 Patterns

**Async params** (required in Next.js 15):
```typescript
// ✅ Correct
async function Page({ params }: { params: { taskId: string } }) {
  const { taskId } = await params  // Must await!
  // ...
}

// ❌ Wrong
function Page({ params }: { params: { taskId: string } }) {
  const { taskId } = params  // Will error in Next.js 15
}
```

**After hook** (non-blocking operations):
```typescript
import { after } from 'next/server'

// Generate branch name after response
after(async () => {
  const branchName = await generateBranchName(taskId)
  await updateTask(taskId, { branchName })
})

return NextResponse.json({ taskId })  // Returns immediately
```

---

## Critical Security Rules

### ⚠️ NEVER LOG DYNAMIC VALUES

**This is the #1 security rule. Logs are displayed in the UI and can leak sensitive data.**

```typescript
// ❌ NEVER do this
await logger.info(`Task created: ${taskId}`)
await logger.error(`Failed to process ${filename}`)
console.log(`User ${userId} logged in`)

// ✅ ALWAYS do this
await logger.info('Task created')
await logger.error('Failed to process file')
console.log('User logged in')
```

**Why?** Logs are returned in API responses and displayed to users. Dynamic values can expose:
- User IDs and personal information
- File paths and repository URLs
- API keys and tokens
- Error details with sensitive context

**Applies to:**
- `logger.info()`, `logger.error()`, `logger.success()`, `logger.command()`
- `console.log()`, `console.error()`, `console.warn()`
- Any logging that reaches the user

### Credential Protection

**Never expose these to client:**
- `SANDBOX_VERCEL_TOKEN`, `SANDBOX_VERCEL_TEAM_ID`, `SANDBOX_VERCEL_PROJECT_ID`
- `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `CURSOR_API_KEY`
- `GITHUB_TOKEN`, `GH_TOKEN`
- `JWE_SECRET`, `ENCRYPTION_KEY`
- User access tokens

**Only expose these** (via `NEXT_PUBLIC_` prefix):
- `NEXT_PUBLIC_AUTH_PROVIDERS`
- `NEXT_PUBLIC_GITHUB_CLIENT_ID`
- `NEXT_PUBLIC_VERCEL_CLIENT_ID`

### Encryption

All sensitive data is encrypted at rest:

```typescript
import { encrypt, decrypt } from '@/lib/crypto'

// Encrypt before storing
const encryptedToken = encrypt(accessToken)
await db.insert(users).values({ accessToken: encryptedToken })

// Decrypt when retrieving
const decryptedToken = decrypt(user.accessToken)
```

Uses **AES-256-GCM** with `ENCRYPTION_KEY` environment variable.

### Redaction (Backup Only)

The `redactSensitiveInfo()` function auto-redacts known patterns, but **don't rely on it**. The primary defense is to never log dynamic values.

---

## Common Development Tasks

### Adding a New API Route

1. Create route file: `app/api/[route]/route.ts`
2. Add session validation:
   ```typescript
   const session = await getServerSession()
   if (!session?.user?.id) {
     return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
   }
   ```
3. Add Zod validation for inputs
4. Implement logic
5. Return JSON response

### Adding a New Database Table

1. Add table to `lib/db/schema.ts`:
   ```typescript
   export const myTable = pgTable('my_table', {
     id: text('id').primaryKey(),
     userId: text('user_id').notNull().references(() => users.id),
     createdAt: timestamp('created_at').defaultNow().notNull(),
   })
   ```
2. Add Zod schemas:
   ```typescript
   export const insertMyTableSchema = z.object({ ... })
   export const selectMyTableSchema = z.object({ ... })
   ```
3. Generate migration: `pnpm db:generate`
4. Apply migration: `pnpm db:push`

### Adding a New Agent

1. Create agent file: `lib/sandbox/agents/my-agent.ts`
2. Implement execution function:
   ```typescript
   export async function executeMyAgentInSandbox(
     sandbox: Sandbox,
     instruction: string,
     logger: TaskLogger,
     apiKey?: string
   ): Promise<AgentExecutionResult>
   ```
3. Add to `lib/sandbox/agents/index.ts`
4. Update agent type in `lib/db/schema.ts`
5. Add UI option in `components/task-form.tsx`

### Adding a New Component

1. Create file: `components/my-component.tsx`
2. If client-side, add `'use client'` directive
3. Import from UI library:
   ```typescript
   import { Button } from '@/components/ui/button'
   ```
4. Export as default or named export
5. Use absolute imports when importing

### Modifying Database Schema

1. Edit `lib/db/schema.ts`
2. Generate migration: `pnpm db:generate`
3. Review migration in `lib/db/migrations/`
4. Apply to local DB: `pnpm db:push`
5. Test thoroughly before deploying

---

## Database Management

### Drizzle ORM Patterns

**Select**:
```typescript
const userTasks = await db
  .select()
  .from(tasks)
  .where(and(
    eq(tasks.userId, userId),
    isNull(tasks.deletedAt)  // Respect soft deletes
  ))
  .orderBy(desc(tasks.createdAt))
```

**Insert**:
```typescript
const [newTask] = await db
  .insert(tasks)
  .values(validatedData)
  .returning()
```

**Update**:
```typescript
await db
  .update(tasks)
  .set({ status: 'completed', completedAt: new Date() })
  .where(eq(tasks.id, taskId))
```

**Delete (soft)**:
```typescript
await db
  .update(tasks)
  .set({ deletedAt: new Date() })
  .where(eq(tasks.id, taskId))
```

### Migrations

**Generate**:
```bash
pnpm db:generate
```

**Push to database**:
```bash
pnpm db:push
```

**Rollback**: Manually revert in SQL

### Common Queries

**Get user with tasks**:
```typescript
const user = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: {
    tasks: {
      where: isNull(tasks.deletedAt),
      orderBy: desc(tasks.createdAt),
    },
  },
})
```

---

## Testing and Quality Assurance

### Code Quality Commands

**ALWAYS run after editing `.ts` or `.tsx` files:**

```bash
pnpm format        # Auto-format with Prettier
pnpm type-check    # TypeScript validation
pnpm lint          # ESLint checking
pnpm build         # Test production build
```

### Pre-Commit Checks

The project uses Husky for pre-commit hooks (if enabled):

- ✅ Prettier formatting
- ✅ TypeScript type checking
- ✅ ESLint linting

### CI/CD Checks

GitHub Actions runs on all PRs (`.github/workflows/pr-checks.yml`):

1. Install dependencies
2. Run lint (`pnpm lint`)
3. Run format check (`pnpm format:check`)
4. Run build (`pnpm build`)

**All checks must pass before merging.**

### Manual Testing

- Test authentication flow (sign in, sign out)
- Test task creation and execution
- Test follow-up messages
- Test file editing and diff viewer
- Test PR creation
- Test MCP connector management

### ⚠️ NEVER Run Dev Servers

**DO NOT run `pnpm dev`, `npm start`, etc.**

**Why?**
- Dev servers run indefinitely
- Conflict with other running instances
- Block terminal sessions
- Make conversations hang

**Instead:**
- Use `pnpm build` to verify production build
- Use `pnpm type-check` to verify types
- Let the user run dev server themselves

---

## Deployment

### Vercel Deployment

**One-Click Deploy**:
1. Click "Deploy with Vercel" button in README
2. Configure environment variables
3. Neon PostgreSQL auto-provisioned
4. Set up OAuth apps
5. Deploy

**Manual Deployment**:
```bash
vercel
```

### Environment Variables

**Required**:
- `POSTGRES_URL` (auto-set on Vercel)
- `SANDBOX_VERCEL_TOKEN`
- `SANDBOX_VERCEL_TEAM_ID`
- `SANDBOX_VERCEL_PROJECT_ID`
- `JWE_SECRET`
- `ENCRYPTION_KEY`
- `NEXT_PUBLIC_AUTH_PROVIDERS`
- At least one OAuth provider (GitHub or Vercel)

**Optional**:
- Global API keys (ANTHROPIC, OPENAI, etc.)
- `MAX_SANDBOX_DURATION`
- `MAX_MESSAGES_PER_DAY`

### Database Migrations

**Production migration**:
```bash
# Using Vercel CLI
vercel env pull .env.production
pnpm db:push
```

Or use `scripts/migrate-production.ts`.

### Post-Deployment

1. Verify OAuth callbacks work
2. Test task creation
3. Verify sandbox provisioning
4. Check logs for errors
5. Monitor database connections

---

## Troubleshooting

### Common Issues

**"Unauthorized" errors**:
- Check session cookie is set
- Verify JWE_SECRET matches
- Check user exists in database

**Sandbox creation fails**:
- Verify SANDBOX_VERCEL_* env vars
- Check Vercel token has correct permissions
- Ensure team/project IDs are correct

**Agent execution fails**:
- Check API keys are set (user or global)
- Verify agent CLI is installed in sandbox
- Check sandbox logs for errors

**Database connection errors**:
- Verify POSTGRES_URL is correct
- Check database is accessible
- Ensure SSL is configured (for Neon)

**Build errors**:
- Run `pnpm type-check` to find type errors
- Run `pnpm lint` to find linting errors
- Check all imports are correct

### Debugging

**Server-side logs**:
```typescript
console.error('Debug info:', { variable })  // Shows in server logs, not user UI
```

**Client-side logs**:
```typescript
console.log('Debug:', data)  // Shows in browser console
```

**Database queries**:
```typescript
// Enable Drizzle logging
import { drizzle } from 'drizzle-orm/postgres-js'
const db = drizzle(client, { logger: true })
```

### Getting Help

- Check `README.md` for setup instructions
- Check `AGENTS.md` for agent-specific rules
- Review GitHub issues for similar problems
- Check Vercel Sandbox docs: https://vercel.com/docs/vercel-sandbox

---

## Additional Resources

- **README.md**: User-facing documentation
- **AGENTS.md**: Critical rules for AI agents (logging, security, dev servers)
- **Vercel Sandbox Docs**: https://vercel.com/docs/vercel-sandbox
- **Drizzle ORM Docs**: https://orm.drizzle.team/
- **Next.js Docs**: https://nextjs.org/docs
- **AI SDK Docs**: https://sdk.vercel.ai/docs

---

## Key Takeaways for AI Assistants

1. **Security First**: Never log dynamic values. Ever.
2. **Type Safety**: Use TypeScript strictly. Validate with Zod.
3. **Quality Checks**: Always run `pnpm format`, `pnpm type-check`, `pnpm lint` after changes.
4. **Never Run Dev Servers**: Use `pnpm build` instead.
5. **Encryption**: All tokens/keys encrypted at rest.
6. **User Isolation**: All queries must filter by `userId`.
7. **Soft Deletes**: Always check `deletedAt IS NULL`.
8. **Async Params**: Await params in Next.js 15.
9. **Absolute Imports**: Use `@/` prefix for all imports.
10. **Error Handling**: Always use try-catch in async functions.

---

**Last Updated**: 2026-01-06
**Version**: 2.0.0
