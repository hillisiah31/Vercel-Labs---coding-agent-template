# CLAUDE.md - AI Assistant Documentation

This document provides comprehensive guidance for AI assistants (like Claude Code) working on this codebase. It covers architecture, conventions, workflows, and critical rules to follow.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Codebase Structure](#codebase-structure)
4. [Database Schema](#database-schema)
5. [Development Workflows](#development-workflows)
6. [Security Guidelines](#security-guidelines)
7. [Code Quality Standards](#code-quality-standards)
8. [API Conventions](#api-conventions)
9. [Common Tasks](#common-tasks)
10. [Testing and Validation](#testing-and-validation)

---

## Project Overview

**Coding Agent Template** is a Next.js application that enables AI-powered coding agents to execute tasks on GitHub repositories using Vercel Sandbox. It supports multiple AI agents (Claude Code, OpenAI Codex, GitHub Copilot, Cursor, Google Gemini, and OpenCode) and provides a multi-user platform for managing coding tasks.

### Key Features

- **Multi-Agent Support**: Users can choose from 6 different AI coding agents
- **User Authentication**: OAuth with GitHub and/or Vercel
- **Secure Sandboxes**: Isolated execution environments via Vercel Sandbox
- **Task Management**: Create, track, and manage coding tasks with real-time logs
- **Git Integration**: Automatic branch creation, commits, and PR management
- **MCP Servers**: Connect Model Context Protocol servers (Claude only)
- **Multi-User**: Each user has isolated tasks, API keys, and GitHub connections

### Tech Stack

- **Framework**: Next.js 15 (App Router)
- **UI**: React 19, Tailwind CSS 4, shadcn/ui
- **Database**: PostgreSQL with Drizzle ORM
- **AI**: AI SDK 5, Vercel AI Gateway
- **Sandbox**: Vercel Sandbox
- **Auth**: OAuth (GitHub/Vercel) with Arctic
- **Encryption**: jose (JWE)

---

## Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client (Browser)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Task Creator │  │ Task Viewer  │  │ Profile/API Keys      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Next.js App Router                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐   │
│  │ Pages      │  │ API Routes │  │ Server Actions         │   │
│  │ (app/)     │  │ (app/api/) │  │ (lib/actions/)         │   │
│  └────────────┘  └────────────┘  └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Business Logic Layer                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐   │
│  │ Auth       │  │ Database   │  │ Sandbox Management     │   │
│  │ (lib/auth) │  │ (lib/db)   │  │ (lib/sandbox)          │   │
│  └────────────┘  └────────────┘  └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│  PostgreSQL  │    │  Vercel Sandbox  │    │  GitHub API  │
│  (Database)  │    │  (Execution)     │    │  (Git Ops)   │
└──────────────┘    └──────────────────┘    └──────────────┘
```

### Request Flow

1. **User Authentication**: User signs in via OAuth (GitHub or Vercel)
2. **Session Management**: Session token stored in encrypted cookie (JWE)
3. **Task Creation**: User creates task → API route validates → Database insert
4. **Sandbox Execution**:
   - Create sandbox with repository
   - Install dependencies (optional)
   - Run selected agent (Claude, Codex, etc.)
   - Stream logs to client via WebSocket
5. **Git Operations**: Agent commits changes → Pushes to branch → Creates PR
6. **Cleanup**: Sandbox destroyed (unless Keep Alive is enabled)

---

## Codebase Structure

### Directory Layout

```
coding-agent-template/
├── app/                          # Next.js App Router
│   ├── api/                      # API routes
│   │   ├── auth/                 # Authentication endpoints
│   │   │   ├── callback/         # OAuth callbacks
│   │   │   ├── github/           # GitHub-specific auth
│   │   │   ├── signin/           # Sign-in endpoints
│   │   │   └── signout/          # Sign-out endpoint
│   │   ├── api-keys/             # User API key management
│   │   ├── connectors/           # MCP connector management
│   │   ├── github/               # GitHub API proxies
│   │   ├── repos/                # Repository data (commits, PRs, issues)
│   │   ├── sandboxes/            # Sandbox management
│   │   ├── tasks/                # Task CRUD and execution
│   │   └── vercel/               # Vercel API proxies
│   ├── repos/                    # Repository viewer pages
│   │   └── [owner]/[repo]/       # Dynamic repo routes
│   │       ├── commits/          # Commits page
│   │       ├── issues/           # Issues page
│   │       └── pull-requests/    # PRs page
│   ├── tasks/                    # Task pages
│   │   └── [taskId]/             # Task detail page
│   ├── layout.tsx                # Root layout
│   ├── page.tsx                  # Home page
│   └── globals.css               # Global styles
│
├── components/                   # React components
│   ├── auth/                     # Auth-related components
│   ├── connectors/               # MCP connector UI
│   ├── icons/                    # Icon components
│   ├── logos/                    # Agent logos
│   ├── providers/                # Context providers
│   ├── ui/                       # shadcn/ui components
│   ├── app-layout.tsx            # Main app layout
│   ├── task-details.tsx          # Task detail view
│   ├── logs-pane.tsx             # Log viewer
│   ├── repo-*.tsx                # Repository components
│   └── ...
│
├── lib/                          # Shared utilities and logic
│   ├── actions/                  # Server actions
│   ├── api-keys/                 # API key management
│   ├── atoms/                    # Jotai state atoms
│   ├── auth/                     # Authentication logic
│   ├── db/                       # Database layer
│   │   ├── client.ts             # Drizzle client
│   │   ├── schema.ts             # Database schema
│   │   ├── users.ts              # User operations
│   │   └── settings.ts           # Settings operations
│   ├── github/                   # GitHub integration
│   ├── hooks/                    # React hooks
│   ├── jwe/                      # Encryption utilities
│   ├── sandbox/                  # Sandbox management
│   │   ├── agents/               # Agent implementations
│   │   │   ├── claude.ts         # Claude Code agent
│   │   │   ├── codex.ts          # OpenAI Codex agent
│   │   │   ├── copilot.ts        # GitHub Copilot agent
│   │   │   ├── cursor.ts         # Cursor agent
│   │   │   ├── gemini.ts         # Google Gemini agent
│   │   │   └── opencode.ts       # OpenCode agent
│   │   ├── commands.ts           # Sandbox command helpers
│   │   ├── config.ts             # Sandbox configuration
│   │   ├── creation.ts           # Sandbox creation logic
│   │   ├── git.ts                # Git operations
│   │   ├── types.ts              # Type definitions
│   │   └── ...
│   ├── session/                  # Session management
│   ├── utils/                    # Utility functions
│   └── vercel-client/            # Vercel API client
│
├── public/                       # Static assets
├── scripts/                      # Utility scripts
├── .github/                      # GitHub Actions
├── .husky/                       # Git hooks
├── drizzle.config.ts             # Drizzle ORM config
├── next.config.ts                # Next.js config
├── package.json                  # Dependencies
├── tsconfig.json                 # TypeScript config
├── tailwind.config.ts            # Tailwind config
├── README.md                     # User documentation
├── AGENTS.md                     # Agent guidelines
└── CLAUDE.md                     # This file
```

### Key Directories Explained

#### `app/api/`

All API routes follow Next.js App Router conventions. Each `route.ts` file exports HTTP handlers (`GET`, `POST`, `PATCH`, `DELETE`).

**Important patterns**:
- All routes require authentication (check session)
- Use Zod for request validation
- Return standardized error responses
- Filter data by `userId` for security

#### `lib/sandbox/agents/`

Each agent file implements the agent-specific execution logic:
- Install agent CLI
- Configure agent (API keys, settings)
- Execute agent with user prompt
- Stream logs back to client
- Handle errors and cleanup

#### `lib/db/`

Database layer using Drizzle ORM:
- `schema.ts`: Table definitions and Zod schemas
- `client.ts`: Database connection
- `users.ts`: User CRUD operations
- `settings.ts`: User settings operations

#### `components/`

React components organized by feature:
- `ui/`: shadcn/ui primitives (button, dialog, input, etc.)
- `auth/`: Authentication UI (sign-in, user menu, etc.)
- `connectors/`: MCP connector management UI
- Root-level components: Feature-specific (tasks, repos, logs)

---

## Database Schema

### Tables Overview

| Table | Purpose | Key Relationships |
|-------|---------|-------------------|
| `users` | User profiles and primary OAuth account | None (root table) |
| `accounts` | Additional linked accounts (e.g., GitHub) | `userId` → `users.id` |
| `keys` | User API keys (Anthropic, OpenAI, etc.) | `userId` → `users.id` |
| `tasks` | Coding tasks created by users | `userId` → `users.id` |
| `task_messages` | Messages for task conversations | `taskId` → `tasks.id` |
| `connectors` | MCP server connections | `userId` → `users.id` |
| `settings` | User-specific settings | `userId` → `users.id` |

### Important Tables

#### `users`

Primary user table. Each user has one primary OAuth provider (GitHub or Vercel).

```typescript
{
  id: string (primary key)
  provider: 'github' | 'vercel'
  externalId: string (OAuth provider's user ID)
  accessToken: string (encrypted)
  refreshToken?: string (encrypted)
  scope?: string
  username: string
  email?: string
  name?: string
  avatarUrl?: string
  createdAt: Date
  updatedAt: Date
  lastLoginAt: Date
}
```

**Unique constraint**: `(provider, externalId)` - prevents duplicate accounts

#### `tasks`

Stores all coding tasks.

```typescript
{
  id: string (primary key)
  userId: string (foreign key)
  prompt: string
  title?: string
  repoUrl?: string
  selectedAgent: 'claude' | 'codex' | 'copilot' | 'cursor' | 'gemini' | 'opencode'
  selectedModel?: string
  installDependencies: boolean
  maxDuration: number (minutes)
  keepAlive: boolean
  status: 'pending' | 'processing' | 'completed' | 'error' | 'stopped'
  progress: number (0-100)
  logs: LogEntry[]
  error?: string
  branchName?: string
  sandboxId?: string
  agentSessionId?: string
  sandboxUrl?: string
  previewUrl?: string
  prUrl?: string
  prNumber?: number
  prStatus?: 'open' | 'closed' | 'merged'
  prMergeCommitSha?: string
  mcpServerIds?: string[]
  createdAt: Date
  updatedAt: Date
  completedAt?: Date
  deletedAt?: Date (soft delete)
}
```

#### `keys`

User API keys (encrypted at rest).

```typescript
{
  id: string (primary key)
  userId: string (foreign key)
  provider: 'anthropic' | 'openai' | 'cursor' | 'gemini' | 'aigateway'
  value: string (encrypted)
  createdAt: Date
  updatedAt: Date
}
```

**Unique constraint**: `(userId, provider)` - one key per provider per user

#### `accounts`

Additional OAuth accounts linked to users (e.g., Vercel users connecting GitHub).

```typescript
{
  id: string (primary key)
  userId: string (foreign key)
  provider: 'github' (currently only GitHub)
  externalUserId: string
  accessToken: string (encrypted)
  refreshToken?: string (encrypted)
  expiresAt?: Date
  scope?: string
  username: string
  createdAt: Date
  updatedAt: Date
}
```

**Unique constraint**: `(userId, provider)` - one account per provider per user

### Data Access Patterns

**Always filter by `userId`**:
```typescript
// GOOD ✅
const tasks = await db
  .select()
  .from(tasks)
  .where(and(eq(tasks.userId, userId), eq(tasks.id, taskId)))

// BAD ❌ - Security vulnerability!
const task = await db
  .select()
  .from(tasks)
  .where(eq(tasks.id, taskId))
```

**Use transactions for related operations**:
```typescript
await db.transaction(async (tx) => {
  const user = await tx.insert(users).values(userData).returning()
  await tx.insert(keys).values({ userId: user[0].id, ...keyData })
})
```

---

## Development Workflows

### Starting Work

1. **Read the task requirements** carefully
2. **Check existing files** before making changes
3. **Understand the context** by reading related code
4. **Plan your approach** - don't jump straight to coding

### Making Changes

1. **Read files first**: Always use `Read` tool before editing
2. **Edit existing files**: Prefer `Edit` over `Write` for existing files
3. **Follow conventions**: Match the style of surrounding code
4. **Be minimal**: Only change what's necessary
5. **Validate types**: Ensure TypeScript is happy

### Code Formatting & Quality

**CRITICAL**: Always run these commands after making changes to TypeScript/TSX files:

```bash
pnpm format        # Auto-format code with Prettier
pnpm type-check    # Verify TypeScript types
pnpm lint          # Check ESLint rules
```

**If errors occur**:
- **Type errors**: Fix type annotations, imports, or mismatches
- **Lint errors**: Follow ESLint suggestions
- **Never skip or ignore errors** - fix them before completing the task

### Prettier Configuration

The project uses specific Prettier settings (defined in `package.json`):

```json
{
  "semi": false,
  "singleQuote": true,
  "printWidth": 120,
  "trailingComma": "all"
}
```

All code must follow these formatting rules.

### Git Workflow

**Branch naming**: All branches must follow this pattern:
```
claude/<descriptive-name>-<session-id>
```

Example: `claude/add-user-auth-A1b2C3`

**Committing**:
1. Stage relevant files with `git add`
2. Create commit with descriptive message
3. Use heredoc for multi-line messages:

```bash
git commit -m "$(cat <<'EOF'
Add user authentication feature

- Implement OAuth with GitHub
- Add session management
- Create user profile page
EOF
)"
```

**Pushing**:
```bash
git push -u origin claude/<branch-name>-<session-id>
```

**Important**: If push fails with 403, verify branch name matches required pattern. Retry network failures up to 4 times with exponential backoff (2s, 4s, 8s, 16s).

### Never Run Dev Servers

**DO NOT run development servers** (`pnpm dev`, `npm run dev`, `next dev`, etc.) as they:
- Run indefinitely and block the session
- Cause port conflicts with existing instances
- Make the conversation hang for the user

**Instead**:
- Use `pnpm build` to verify production build
- Use `pnpm type-check` for type validation
- Use `pnpm lint` for code quality
- Let the user run dev server themselves if needed

---

## Security Guidelines

### Critical: Static Logging Only

**ALL log statements MUST use static strings only. NEVER include dynamic values.**

This is the most important security rule in the codebase.

#### Why This Rule Exists

- Logs are displayed directly in the UI
- Dynamic values can expose sensitive information (user IDs, tokens, file paths)
- This applies to ALL log levels (info, error, success, command)

#### Examples

**BAD ❌ - Never do this**:
```typescript
await logger.info(`Task created: ${taskId}`)
await logger.error(`Failed to process ${filename}`)
console.log(`User ${userId} logged in`)
console.error(`Error for ${provider}:`, error)
```

**GOOD ✅ - Always do this**:
```typescript
await logger.info('Task created')
await logger.error('Failed to process file')
console.log('User logged in')
console.error('Error occurred:', error)
```

#### Sensitive Data to Never Log

- Vercel credentials (SANDBOX_VERCEL_TOKEN, SANDBOX_VERCEL_TEAM_ID, SANDBOX_VERCEL_PROJECT_ID)
- User IDs and personal information
- File paths and repository URLs
- Branch names and commit messages
- API keys and access tokens
- Error details containing sensitive context

### Encryption

All sensitive data is encrypted at rest using per-user encryption:

**Encrypting data**:
```typescript
import { encrypt } from '@/lib/jwe/encrypt'

const encryptedValue = await encrypt(sensitiveData)
```

**Decrypting data**:
```typescript
import { decrypt } from '@/lib/jwe/decrypt'

const decryptedValue = await decrypt(encryptedValue)
```

**Requires environment variable**: `ENCRYPTION_KEY` (32-byte hex string)

### Authentication

All API routes must verify the session:

```typescript
import { getSession } from '@/lib/session/server'

export async function GET(request: Request) {
  const session = await getSession()
  if (!session?.user?.id) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const userId = session.user.id
  // ... rest of handler
}
```

### Authorization

Always filter queries by `userId` to prevent unauthorized access:

```typescript
// Fetch user's tasks only
const userTasks = await db
  .select()
  .from(tasks)
  .where(eq(tasks.userId, userId))

// Fetch specific task (verify ownership)
const task = await db
  .select()
  .from(tasks)
  .where(and(
    eq(tasks.id, taskId),
    eq(tasks.userId, userId) // 👈 Critical!
  ))
```

---

## Code Quality Standards

### TypeScript

- **Enable strict mode**: `tsconfig.json` has `strict: true`
- **Avoid `any`**: Use proper types or `unknown`
- **Use Zod for validation**: All API inputs must be validated
- **Export types from schema**: Use Drizzle-inferred types

Example:
```typescript
import { insertTaskSchema, type Task } from '@/lib/db/schema'

// Validate input
const validatedData = insertTaskSchema.parse(requestBody)

// Use inferred types
function processTask(task: Task) {
  // ...
}
```

### React Components

- **Use TypeScript**: All components should have proper types
- **Client components**: Use `'use client'` directive when needed
- **Server components**: Default - no directive needed
- **Prop types**: Always define prop interfaces

```typescript
interface TaskCardProps {
  task: Task
  onSelect?: (taskId: string) => void
}

export function TaskCard({ task, onSelect }: TaskCardProps) {
  // ...
}
```

### Error Handling

- **Always handle errors**: Never let errors crash the app
- **Use try-catch**: Wrap risky operations
- **Return user-friendly messages**: Don't expose internals
- **Log server-side details**: Use `console.error` for debugging

```typescript
try {
  const result = await riskyOperation()
  return Response.json({ success: true, data: result })
} catch (error) {
  console.error('Detailed error for debugging:', error)
  return Response.json(
    { error: 'Operation failed' }, // Generic message
    { status: 500 }
  )
}
```

### File Organization

- **One component per file**: Except for related small components
- **Group related files**: Use directories for features
- **Barrel exports**: Use `index.ts` for cleaner imports
- **Colocation**: Keep related files together

---

## API Conventions

### Request/Response Format

**Standard success response**:
```json
{
  "success": true,
  "data": { ... }
}
```

**Standard error response**:
```json
{
  "error": "Error message",
  "details": "Optional details"
}
```

### Status Codes

- `200`: Success
- `201`: Created
- `400`: Bad request (validation error)
- `401`: Unauthorized (not authenticated)
- `403`: Forbidden (authenticated but not authorized)
- `404`: Not found
- `500`: Internal server error

### Validation Pattern

All API routes should validate inputs with Zod:

```typescript
import { z } from 'zod'

const requestSchema = z.object({
  prompt: z.string().min(1),
  repoUrl: z.string().url().optional(),
})

export async function POST(request: Request) {
  try {
    const body = await request.json()
    const validated = requestSchema.parse(body)

    // ... use validated data
  } catch (error) {
    if (error instanceof z.ZodError) {
      return Response.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      )
    }
    throw error
  }
}
```

---

## Common Tasks

### Adding a New API Endpoint

1. Create file in `app/api/[feature]/route.ts`
2. Export HTTP handler (`GET`, `POST`, etc.)
3. Verify session and extract `userId`
4. Validate request body with Zod
5. Perform database operations (filter by `userId`)
6. Return standardized response

Example:
```typescript
import { getSession } from '@/lib/session/server'
import { db } from '@/lib/db/client'
import { tasks } from '@/lib/db/schema'
import { eq } from 'drizzle-orm'

export async function GET(request: Request) {
  const session = await getSession()
  if (!session?.user?.id) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const userTasks = await db
    .select()
    .from(tasks)
    .where(eq(tasks.userId, session.user.id))

  return Response.json({ success: true, data: userTasks })
}
```

### Adding a New Database Table

1. Define table in `lib/db/schema.ts`
2. Add Zod validation schemas
3. Export TypeScript types
4. Generate migration: `pnpm db:generate`
5. Apply migration: `pnpm db:push`
6. Update related queries

### Adding a New Agent

1. Create file in `lib/sandbox/agents/[agent-name].ts`
2. Implement `executeAgent` function with signature:
   ```typescript
   export async function executeAgent(
     sandbox: Sandbox,
     prompt: string,
     logger: TaskLogger,
     options: AgentOptions
   ): Promise<AgentExecutionResult>
   ```
3. Add agent to `lib/sandbox/agents/index.ts`
4. Update `selectedAgent` enum in `lib/db/schema.ts`
5. Add logo to `components/logos/[agent-name].tsx`
6. Update UI to include new agent option

### Adding a New Component

1. Create file in `components/[feature-name].tsx`
2. Define prop types interface
3. Implement component with proper TypeScript
4. Use client directive if needed: `'use client'`
5. Import and use in parent component
6. Run `pnpm format` to format code

---

## Testing and Validation

### Pre-Commit Checklist

Before committing changes, ensure:

- [ ] No template literals with `${}` in log statements
- [ ] All logger calls use static strings
- [ ] No sensitive data in error messages
- [ ] Ran `pnpm format` - code is formatted
- [ ] Ran `pnpm type-check` - no TypeScript errors
- [ ] Ran `pnpm lint` - no linting errors
- [ ] Ran `pnpm build` - production build succeeds
- [ ] Tested in UI (if applicable) - no data leakage

### Manual Testing

For UI changes:
1. Start dev server locally (user runs this, not AI)
2. Sign in with OAuth
3. Test the feature end-to-end
4. Check browser console for errors
5. Verify logs don't expose sensitive data

For API changes:
1. Use `curl` or Postman to test endpoints
2. Verify authentication requirements
3. Check response format matches conventions
4. Test error cases (invalid input, unauthorized, etc.)

### Common Issues

**TypeScript errors**:
- Check import paths are correct
- Verify types match expected values
- Use `as const` for literal types where needed

**Linting errors**:
- Missing dependencies in `useEffect`
- Unused variables (remove or prefix with `_`)
- Missing `key` prop in list rendering

**Build errors**:
- Environment variables not available at build time
- Server components using client-only APIs
- Missing dependencies in `package.json`

---

## Additional Resources

- **README.md**: User-facing documentation and setup guide
- **AGENTS.md**: Specific guidelines for AI agents (security, logging, code quality)
- **Database Schema**: `lib/db/schema.ts` - Source of truth for data structure
- **API Routes**: `app/api/` - All backend endpoints
- **Components**: `components/` - All UI components

---

## Key Takeaways for AI Assistants

1. **Security First**: Never log dynamic values, always encrypt sensitive data, always filter by `userId`
2. **Code Quality**: Always run format, type-check, and lint before finishing
3. **Read First**: Always read files before editing them
4. **Be Minimal**: Only change what's necessary, don't over-engineer
5. **Follow Conventions**: Match existing patterns and styles
6. **Never Run Dev Servers**: Use build/type-check/lint instead
7. **Validate Everything**: Use Zod for all user inputs
8. **Think User-Scoped**: All data access must consider user ownership

---

**Last Updated**: December 2024 (v2.0.0)
