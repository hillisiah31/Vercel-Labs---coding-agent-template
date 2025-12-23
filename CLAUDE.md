# CLAUDE.md - AI Assistant Developer Guide

> **Last Updated**: December 2025
> **Version**: 2.0.0

This document provides comprehensive guidance for AI assistants (like Claude) working on the Coding Agent Template codebase. It explains the architecture, conventions, workflows, and critical rules to follow.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Tech Stack](#tech-stack)
3. [Architecture Overview](#architecture-overview)
4. [Directory Structure](#directory-structure)
5. [Database Schema](#database-schema)
6. [Key Features & Workflows](#key-features--workflows)
7. [Development Guidelines](#development-guidelines)
8. [API Routes Reference](#api-routes-reference)
9. [Common Development Tasks](#common-development-tasks)
10. [Security Considerations](#security-considerations)
11. [Testing & Quality Assurance](#testing--quality-assurance)

---

## Project Overview

**Coding Agent Template** is a multi-agent AI coding platform that allows users to execute automated coding tasks using various AI agents (Claude Code, OpenAI Codex, GitHub Copilot, Cursor, Google Gemini, and opencode) within secure Vercel Sandbox environments.

### Key Capabilities

- **Multi-Agent Support**: Six different AI coding agents with model selection
- **User Authentication**: OAuth-based auth with GitHub and/or Vercel
- **Secure Sandboxes**: Isolated execution environments via Vercel Sandbox
- **Git Integration**: Automatic branch creation, commits, and PR management
- **MCP Server Support**: Extensible via Model Context Protocol servers (Claude only)
- **Real-time Task Monitoring**: WebSocket-based live logs and progress updates
- **Multi-user Isolation**: Each user has their own tasks, API keys, and resources

---

## Tech Stack

### Frontend
- **Framework**: Next.js 15 (App Router with React Server Components)
- **React**: v19.1.0
- **Styling**: Tailwind CSS v4.1.13
- **UI Components**: shadcn/ui (Radix UI primitives)
- **State Management**: Jotai v2.15.0
- **Themes**: next-themes v0.4.6

### Backend
- **Runtime**: Node.js (Next.js API Routes)
- **Database**: PostgreSQL (via Neon in production)
- **ORM**: Drizzle ORM v0.36.4
- **Authentication**: Custom OAuth implementation with Arctic v3.7.0
- **Encryption**: jose v6.1.0 (JWE for sessions, custom crypto for API keys)

### AI & Agents
- **AI SDK**: Vercel AI SDK v5.0.51
- **AI Gateway**: Vercel AI Gateway for model routing and observability
- **Agents**: Claude Code, Codex CLI, Copilot CLI, Cursor CLI, Gemini CLI, opencode
- **Sandbox**: @vercel/sandbox v0.0.21

### Developer Tools
- **TypeScript**: v5
- **Linting**: ESLint v9
- **Formatting**: Prettier v3.6.2
- **Git Hooks**: Husky v9.1.7
- **Package Manager**: pnpm

---

## Architecture Overview

### Multi-Tier Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                           │
│  Next.js App Router (React 19) + Tailwind CSS               │
│  - Server Components for data fetching                      │
│  - Client Components for interactivity                      │
│  - Real-time updates via streaming and polling              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    API ROUTES LAYER                          │
│  Next.js Route Handlers (app/api/*)                         │
│  - Authentication & Authorization                           │
│  - Rate Limiting                                            │
│  - Request Validation (Zod schemas)                         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   BUSINESS LOGIC LAYER                       │
│  lib/* modules                                              │
│  - Task execution orchestration                             │
│  - Sandbox lifecycle management                             │
│  - Agent dispatching and communication                      │
│  - Git operations (branching, commits, PRs)                 │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    DATA LAYER                                │
│  PostgreSQL + Drizzle ORM                                   │
│  - Users, Tasks, Connectors, Accounts, Keys, Settings       │
│  - Encrypted sensitive data (tokens, API keys)              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   EXTERNAL SERVICES                          │
│  - Vercel Sandbox (code execution)                          │
│  - GitHub API (repo access, PRs)                            │
│  - AI Model Providers (Anthropic, OpenAI, Google, Cursor)   │
│  - MCP Servers (extensibility)                              │
└─────────────────────────────────────────────────────────────┘
```

### Request Flow for Task Execution

1. **User creates task** → POST `/api/tasks`
2. **Task record created** → Database (status: `pending`)
3. **Branch name generation** → AI Gateway (async, non-blocking via `after()`)
4. **Sandbox creation** → Vercel Sandbox API
5. **Agent execution** → Selected agent runs in sandbox
6. **Git operations** → Commits and pushes changes to branch
7. **Task completion** → Update database (status: `completed`)
8. **Cleanup** → Sandbox destroyed (unless `keepAlive` is true)

---

## Directory Structure

```
coding-agent-template/
├── .github/              # GitHub workflows and configuration
├── .husky/               # Git hooks (pre-commit)
├── app/                  # Next.js App Router
│   ├── api/              # API route handlers
│   │   ├── api-keys/     # User API key management
│   │   ├── auth/         # Authentication (GitHub, Vercel, sessions)
│   │   ├── connectors/   # MCP server management
│   │   ├── github/       # GitHub API proxies (repos, orgs, users)
│   │   ├── github-stars/ # GitHub stars integration
│   │   ├── repos/        # Repository operations (commits, issues, PRs)
│   │   ├── sandboxes/    # Sandbox management (list, cleanup)
│   │   ├── tasks/        # Task CRUD and execution
│   │   └── vercel/       # Vercel API integration
│   ├── repos/            # Repository viewer pages
│   │   └── [owner]/[repo]/
│   │       ├── commits/
│   │       ├── issues/
│   │       └── pull-requests/
│   ├── tasks/            # Task pages
│   │   └── [id]/         # Individual task view
│   ├── globals.css       # Global styles
│   ├── layout.tsx        # Root layout with providers
│   └── page.tsx          # Home page
├── components/           # React components
│   ├── auth/             # Authentication UI components
│   ├── connectors/       # MCP connector components
│   ├── icons/            # Custom icon components
│   ├── logos/            # Logo components
│   ├── providers/        # Context providers (auth, connectors)
│   ├── ui/               # shadcn/ui components
│   └── *.tsx             # Feature components
├── lib/                  # Utility libraries and business logic
│   ├── actions/          # Server actions
│   ├── api-keys/         # API key management
│   ├── atoms/            # Jotai state atoms
│   ├── auth/             # Authentication utilities
│   ├── crypto.ts         # Encryption/decryption utilities
│   ├── db/               # Database configuration and schema
│   │   ├── client.ts     # Drizzle client
│   │   ├── schema.ts     # Database tables and Zod schemas
│   │   ├── settings.ts   # User settings helpers
│   │   └── users.ts      # User management helpers
│   ├── github/           # GitHub API client and utilities
│   ├── github-stars.ts   # GitHub stars integration
│   ├── hooks/            # React hooks
│   ├── jwe/              # JSON Web Encryption for sessions
│   ├── sandbox/          # Sandbox orchestration
│   │   ├── agents/       # Agent implementations (claude, codex, etc.)
│   │   ├── commands.ts   # Sandbox command execution
│   │   ├── config.ts     # Sandbox configuration
│   │   ├── creation.ts   # Sandbox lifecycle
│   │   ├── git.ts        # Git operations in sandbox
│   │   ├── package-manager.ts  # Dependency installation
│   │   └── types.ts      # Sandbox type definitions
│   ├── session/          # Session management
│   ├── utils/            # Utility functions
│   ├── vercel-client/    # Vercel API client
│   ├── constants.ts      # Global constants
│   └── utils.ts          # General utilities
├── public/               # Static assets
├── scripts/              # Utility scripts
├── .env.local            # Local environment variables (gitignored)
├── package.json          # Dependencies and scripts
├── tsconfig.json         # TypeScript configuration
├── next.config.ts        # Next.js configuration
├── drizzle.config.ts     # Drizzle ORM configuration
├── tailwind.config.ts    # Tailwind CSS configuration (implicit)
├── AGENTS.md             # Agent development guidelines
├── README.md             # User-facing documentation
└── CLAUDE.md             # This file
```

---

## Database Schema

The application uses PostgreSQL with the following tables:

### Core Tables

#### `users`
Primary user records (one per OAuth login).

**Key Fields**:
- `id` (PK): Internal user identifier
- `provider`: Primary OAuth provider (`github` | `vercel`)
- `externalId`: External OAuth user ID
- `accessToken`: Encrypted OAuth access token
- `username`, `email`, `name`, `avatarUrl`: Profile info
- `createdAt`, `updatedAt`, `lastLoginAt`: Timestamps

**Constraints**:
- Unique index on `(provider, externalId)` - prevents duplicate signups

#### `accounts`
Additional linked accounts (e.g., Vercel users connecting GitHub).

**Key Fields**:
- `id` (PK): Account identifier
- `userId` (FK → users): Owner of the linked account
- `provider`: Currently only `github`
- `externalUserId`: GitHub user ID
- `accessToken`: Encrypted GitHub token
- `username`: GitHub username

**Constraints**:
- Unique index on `(userId, provider)` - one GitHub account per user

#### `tasks`
User-created coding tasks.

**Key Fields**:
- `id` (PK): Task identifier
- `userId` (FK → users): Task owner
- `prompt`: Task description
- `repoUrl`: Repository URL
- `selectedAgent`: Agent choice (`claude`, `codex`, `copilot`, `cursor`, `gemini`, `opencode`)
- `selectedModel`: Model identifier
- `status`: `pending` | `processing` | `completed` | `error` | `stopped`
- `progress`: Integer 0-100
- `logs`: JSONB array of log entries
- `branchName`: Git branch created
- `sandboxId`: Vercel sandbox identifier
- `agentSessionId`: Agent session ID (for resumable agents like Cursor)
- `prUrl`, `prNumber`, `prStatus`: Pull request info
- `mcpServerIds`: Array of connected MCP server IDs
- `installDependencies`, `maxDuration`, `keepAlive`: Configuration flags
- `deletedAt`: Soft delete timestamp

#### `connectors`
MCP servers connected by users.

**Key Fields**:
- `id` (PK): Connector identifier
- `userId` (FK → users): Connector owner
- `name`, `description`: Connector metadata
- `type`: `local` | `remote`
- `baseUrl`: For remote MCP servers
- `command`: For local MCP servers
- `env`: Encrypted environment variables (text, not JSONB)
- `status`: `connected` | `disconnected`

#### `keys`
User-provided API keys for AI providers.

**Key Fields**:
- `id` (PK): Key identifier
- `userId` (FK → users): Key owner
- `provider`: `anthropic` | `openai` | `cursor` | `gemini` | `aigateway`
- `value`: Encrypted API key

**Constraints**:
- Unique index on `(userId, provider)` - one key per provider per user

#### `taskMessages`
Chat messages for tasks (user follow-ups and agent responses).

**Key Fields**:
- `id` (PK): Message identifier
- `taskId` (FK → tasks): Associated task
- `role`: `user` | `agent`
- `content`: Message text
- `createdAt`: Timestamp

#### `settings`
User-specific settings (key-value pairs).

**Key Fields**:
- `id` (PK): Setting identifier
- `userId` (FK → users): Setting owner
- `key`: Setting name (e.g., `maxMessagesPerDay`)
- `value`: Setting value (stored as text)

**Constraints**:
- Unique index on `(userId, key)`

### Data Relationships

```
users
  ├─→ accounts (1:N) - linked OAuth accounts
  ├─→ keys (1:N) - API keys
  ├─→ tasks (1:N) - coding tasks
  │     └─→ taskMessages (1:N) - task chat messages
  ├─→ connectors (1:N) - MCP servers
  └─→ settings (1:N) - user preferences
```

All foreign keys use `ON DELETE CASCADE` for automatic cleanup.

---

## Key Features & Workflows

### 1. User Authentication Flow

**OAuth Providers**: GitHub and/or Vercel (configurable via `NEXT_PUBLIC_AUTH_PROVIDERS`)

**Sign-in Process**:
1. User clicks "Sign in with GitHub" or "Sign in with Vercel"
2. Redirect to OAuth provider
3. Callback to `/api/auth/github/callback` or `/api/auth/callback/vercel`
4. User record created/updated in database
5. Encrypted session token stored in HTTP-only cookie
6. Redirect to home page

**Identity Merging**:
- If a user signs in with Vercel, then connects GitHub, then later signs in directly with GitHub → same user account
- Matching is done via GitHub external user ID

**GitHub Access**:
- GitHub OAuth users: automatic repo access
- Vercel OAuth users: must connect GitHub account from profile

### 2. Task Execution Workflow

**Task Creation**:
1. User fills out task form (repo URL, prompt, agent, options)
2. POST to `/api/tasks` → validates and creates task record
3. Async branch name generation via AI Gateway (Next.js `after()`)
4. Returns task ID immediately

**Task Processing** (handled by background process):
1. Create Vercel sandbox with repo cloned
2. Install dependencies (if `installDependencies: true`)
3. Execute selected agent with prompt
4. Collect logs and stream to database
5. Commit and push changes to branch
6. Mark task as `completed` or `error`
7. Destroy sandbox (unless `keepAlive: true`)

**Agent-Specific Behavior**:
- **Claude**: MCP server support, structured output
- **Codex**: OpenAI Codex CLI
- **Copilot**: GitHub Copilot CLI
- **Cursor**: Session-based, resumable via `agentSessionId`
- **Gemini**: Google Gemini CLI
- **Opencode**: OpenCode CLI

### 3. Git Operations

**Branch Creation**:
- AI-generated branch names: `feature/add-auth-A1b2C3`
- Fallback: timestamp-based names
- Created from default branch (main/master)

**Commit Flow**:
1. Detect changes via `git status`
2. Stage all changes: `git add .`
3. Commit with descriptive message (from agent)
4. Push to remote: `git push -u origin <branch>`

**Pull Request Support**:
- Tasks store `prUrl` and `prNumber`
- UI allows creating PRs directly from task page
- PR status tracking: `open`, `closed`, `merged`

### 4. MCP Server Integration (Claude Only)

**Adding MCP Servers**:
1. Navigate to Connectors tab
2. Click "Add MCP Server"
3. Configure name, type (local/remote), base URL, env vars
4. Encrypted storage of credentials

**Usage**:
- When creating a task with Claude agent, select MCP servers to connect
- MCP server IDs stored in `task.mcpServerIds`
- Sandbox provisions MCP server connections before agent execution

### 5. Sandbox Lifecycle

**Creation**:
- Vercel Sandbox API: `POST https://api.vercel.com/v1/sandboxes`
- Environment variables injected (API keys, GitHub token, etc.)
- Timeout configured via `maxDuration` (default: 300 minutes)

**Keep Alive Behavior**:
- `keepAlive: false` (default): Sandbox destroyed immediately after task completion
- `keepAlive: true`: Sandbox stays alive until timeout, enables follow-up messages

**Cleanup**:
- Automatic: sandbox expires after timeout
- Manual: DELETE request to Vercel API
- Registry tracking: `lib/sandbox/sandbox-registry.ts`

---

## Development Guidelines

### Critical Rules

#### 🚨 **NEVER Log Dynamic Values**

**Rule**: All log statements MUST use static strings only. No exceptions.

**Why**: Logs are displayed directly in the UI and can leak sensitive data (tokens, user IDs, file paths, etc.).

**Bad**:
```typescript
await logger.info(`Task ${taskId} created`)
console.log(`User ${userId} logged in`)
```

**Good**:
```typescript
await logger.info('Task created')
console.log('User logged in')
```

See [AGENTS.md](./AGENTS.md) for complete logging guidelines and redaction patterns.

#### 🚨 **NEVER Run Dev Servers**

**Rule**: Do NOT run `pnpm dev`, `npm run dev`, `next dev`, or any long-running development servers.

**Why**:
- Blocks terminal sessions indefinitely
- Causes port conflicts
- May already be running in user's environment

**What to Do Instead**:
- Use `pnpm build` to verify production builds
- Use `pnpm type-check` for TypeScript validation
- Use `pnpm lint` for linting
- Let the user run dev servers themselves

#### 🔒 **Always Run Quality Checks**

**Rule**: After editing TypeScript/TSX files, ALWAYS run:
```bash
pnpm format      # Prettier formatting
pnpm type-check  # TypeScript validation
pnpm lint        # ESLint checks
```

Fix all errors before considering the task complete.

### Code Style Conventions

**File Organization**:
- Server components: `app/**/page.tsx`
- Client components: `'use client'` directive at top
- Server actions: `lib/actions/*.ts`
- API routes: `app/api/**/route.ts`

**Naming Conventions**:
- Components: PascalCase (`TaskForm.tsx`)
- Utilities: camelCase (`getUserById.ts`)
- Constants: UPPER_SNAKE_CASE (`MAX_SANDBOX_DURATION`)
- Database tables: snake_case (`task_messages`)

**Import Aliases**:
- Use `@/*` for all imports: `import { db } from '@/lib/db/client'`

**Prettier Configuration** (package.json):
```json
{
  "semi": false,
  "singleQuote": true,
  "printWidth": 120,
  "trailingComma": "all"
}
```

### Security Best Practices

**Environment Variables**:
- **NEVER** expose server-only env vars to client
- Client vars MUST have `NEXT_PUBLIC_` prefix
- Store secrets in environment variables, never in code

**Encryption**:
- Use `lib/crypto.ts` for encrypting/decrypting sensitive data
- All tokens and API keys MUST be encrypted at rest
- Encryption key: `ENCRYPTION_KEY` env var (32-byte hex string)

**Session Management**:
- Sessions use JWE (JSON Web Encryption)
- HTTP-only, secure cookies
- Secret: `JWE_SECRET` env var (base64-encoded)

**Authorization**:
- Always check `userId` matches resource owner
- Use `lib/session/get-user-from-session.ts` helper
- Rate limiting: `lib/auth/rate-limit.ts`

---

## API Routes Reference

### Authentication Routes

| Route | Method | Purpose | Auth Required |
|-------|--------|---------|---------------|
| `/api/auth/signin/github` | GET | Initiate GitHub OAuth | No |
| `/api/auth/signin/vercel` | GET | Initiate Vercel OAuth | No |
| `/api/auth/github/callback` | GET | GitHub OAuth callback | No |
| `/api/auth/callback/vercel` | GET | Vercel OAuth callback | No |
| `/api/auth/signout` | POST | Sign out user | Yes |
| `/api/auth/info` | GET | Get current user info | Yes |
| `/api/auth/github/status` | GET | Check GitHub connection | Yes |
| `/api/auth/github/disconnect` | POST | Disconnect GitHub account | Yes |
| `/api/auth/rate-limit` | GET | Check rate limit status | Yes |

### Task Routes

| Route | Method | Purpose | Auth Required |
|-------|--------|---------|---------------|
| `/api/tasks` | POST | Create new task | Yes |
| `/api/tasks` | GET | List user's tasks | Yes |
| `/api/tasks/[taskId]` | GET | Get task details | Yes |
| `/api/tasks/[taskId]` | PATCH | Update task | Yes |
| `/api/tasks/[taskId]` | DELETE | Delete task (soft) | Yes |

### Repository Routes

| Route | Method | Purpose | Auth Required |
|-------|--------|---------|---------------|
| `/api/github/repos` | GET | List user's repos | Yes |
| `/api/github/orgs` | GET | List user's orgs | Yes |
| `/api/github/user` | GET | Get GitHub user info | Yes |
| `/api/repos/[owner]/[repo]/commits` | GET | Get repo commits | Yes |
| `/api/repos/[owner]/[repo]/issues` | GET | Get repo issues | Yes |
| `/api/repos/[owner]/[repo]/pull-requests` | GET | Get repo PRs | Yes |

### API Key Routes

| Route | Method | Purpose | Auth Required |
|-------|--------|---------|---------------|
| `/api/api-keys` | POST | Save user API key | Yes |
| `/api/api-keys` | GET | Get user API keys (masked) | Yes |
| `/api/api-keys` | DELETE | Delete user API key | Yes |
| `/api/api-keys/check` | GET | Check which keys are configured | Yes |

### Connector Routes (MCP Servers)

| Route | Method | Purpose | Auth Required |
|-------|--------|---------|---------------|
| `/api/connectors` | POST | Create MCP connector | Yes |
| `/api/connectors` | GET | List user's connectors | Yes |
| `/api/connectors` | PATCH | Update connector | Yes |
| `/api/connectors` | DELETE | Delete connector | Yes |

### Sandbox Routes

| Route | Method | Purpose | Auth Required |
|-------|--------|---------|---------------|
| `/api/sandboxes` | GET | List active sandboxes | Yes |
| `/api/sandboxes` | DELETE | Delete specific sandbox | Yes |

---

## Common Development Tasks

### Adding a New AI Agent

1. **Create agent implementation**: `lib/sandbox/agents/my-agent.ts`
   ```typescript
   import { AgentExecutionResult } from '../types'
   import { Sandbox } from '@vercel/sandbox'

   export async function executeMyAgent(
     sandbox: Sandbox,
     prompt: string,
     options: any
   ): Promise<AgentExecutionResult> {
     // Implementation
   }
   ```

2. **Register in agent index**: `lib/sandbox/agents/index.ts`
   ```typescript
   export { executeMyAgent } from './my-agent'
   ```

3. **Update schema**: `lib/db/schema.ts`
   ```typescript
   selectedAgent: z.enum(['claude', 'codex', 'copilot', 'cursor', 'gemini', 'opencode', 'my-agent'])
   ```

4. **Update UI**: `components/task-form.tsx` - add agent to dropdown

5. **Test thoroughly**: Create task with new agent, verify execution

### Adding a New Database Table

1. **Define schema**: `lib/db/schema.ts`
   ```typescript
   export const myTable = pgTable('my_table', {
     id: text('id').primaryKey(),
     userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
     // ... other fields
   })

   export const insertMyTableSchema = z.object({ /* ... */ })
   export const selectMyTableSchema = z.object({ /* ... */ })
   export type MyTable = z.infer<typeof selectMyTableSchema>
   ```

2. **Generate migration**: `pnpm db:generate`

3. **Apply migration**: `pnpm db:push`

4. **Create helpers**: `lib/db/my-table.ts` (CRUD operations)

### Adding a New API Route

1. **Create route file**: `app/api/my-route/route.ts`
   ```typescript
   import { NextRequest, NextResponse } from 'next/server'
   import { getUserFromSession } from '@/lib/session/get-user-from-session'

   export async function GET(req: NextRequest) {
     const user = await getUserFromSession(req)
     if (!user) {
       return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
     }

     // Implementation
     return NextResponse.json({ data: result })
   }
   ```

2. **Add request/response types** (if needed)

3. **Test with curl or Postman**

4. **Update this documentation** (API Routes Reference section)

### Adding a New UI Component

1. **Create component**: `components/my-component.tsx`
   ```typescript
   'use client'

   import { useState } from 'react'

   export function MyComponent() {
     // Implementation
   }
   ```

2. **If using shadcn/ui**: `pnpx shadcn-ui@latest add <component-name>`

3. **Import and use** in page or parent component

4. **Style with Tailwind** classes

### Running Database Migrations

```bash
# Generate migration from schema changes
pnpm db:generate

# Push changes to database (development)
pnpm db:push

# Open Drizzle Studio (GUI for database)
pnpm db:studio
```

---

## Security Considerations

### Authentication & Authorization

**Session Security**:
- Sessions encrypted with JWE (JSON Web Encryption)
- HTTP-only, secure, SameSite=Lax cookies
- 7-day expiration, auto-refresh on activity

**Resource Ownership**:
- All API routes MUST verify `userId` matches resource owner
- Use database queries with `WHERE userId = ?` filters
- Foreign key constraints enforce data isolation

**Rate Limiting**:
- Default: 5 messages (tasks + follow-ups) per user per day
- Configurable via `MAX_MESSAGES_PER_DAY` env var
- User-specific overrides via `settings` table

### Encryption

**API Keys and Tokens**:
- All sensitive data encrypted at rest using AES-256-GCM
- Encryption key: `ENCRYPTION_KEY` env var (32-byte hex)
- Encryption/decryption functions: `lib/crypto.ts`

**Fields That MUST Be Encrypted**:
- `users.accessToken`, `users.refreshToken`
- `accounts.accessToken`, `accounts.refreshToken`
- `keys.value`
- `connectors.env`

**Encryption Flow**:
```typescript
import { encrypt, decrypt } from '@/lib/crypto'

// Encrypting
const encryptedToken = await encrypt(rawToken)
await db.insert(users).values({ accessToken: encryptedToken })

// Decrypting
const user = await db.query.users.findFirst(/* ... */)
const rawToken = await decrypt(user.accessToken)
```

### Sensitive Data Handling

**NEVER Log or Expose**:
- `SANDBOX_VERCEL_TOKEN`, `SANDBOX_VERCEL_TEAM_ID`, `SANDBOX_VERCEL_PROJECT_ID`
- User IDs, GitHub tokens, API keys
- File paths, repository URLs, branch names
- Any decrypted sensitive data

**Redaction**:
- `lib/utils/logging.ts` has `redactSensitiveInfo()` function
- Automatically redacts known patterns (API keys, tokens, etc.)
- **Primary defense: never log dynamic values**

**Client-Side Exposure**:
- Only `NEXT_PUBLIC_*` variables are exposed to client
- Never send API keys or tokens to client
- Mask API keys in responses (show last 4 chars only)

---

## Testing & Quality Assurance

### Pre-Commit Checklist

Before committing changes:

- [ ] No template literals with `${}` in log statements
- [ ] All logger/console calls use static strings
- [ ] No sensitive data in error messages or logs
- [ ] Ran `pnpm format` - code is properly formatted
- [ ] Ran `pnpm format:check` - formatting verified
- [ ] Ran `pnpm type-check` - all type errors fixed
- [ ] Ran `pnpm lint` - all linting errors fixed
- [ ] Tested changes manually in development
- [ ] No hardcoded credentials or secrets

### Testing Commands

```bash
# Format code with Prettier
pnpm format

# Check if code is formatted
pnpm format:check

# Type-check TypeScript
pnpm type-check

# Lint code with ESLint
pnpm lint

# Build for production (verifies build succeeds)
pnpm build
```

### Manual Testing Workflow

1. **Start development server** (user does this, not you):
   ```bash
   pnpm dev
   ```

2. **Test authentication**:
   - Sign in with GitHub
   - Sign in with Vercel
   - Sign out
   - Check session persistence

3. **Test task creation**:
   - Create task with different agents
   - Monitor logs in real-time
   - Verify branch creation
   - Check PR creation

4. **Test error handling**:
   - Invalid repo URLs
   - Missing API keys
   - Network failures
   - Rate limit exceeded

5. **Test database operations**:
   - Open Drizzle Studio: `pnpm db:studio`
   - Verify data integrity
   - Check foreign key constraints
   - Test soft deletes

### Debugging Tips

**Server-Side Logs**:
```typescript
// These appear in terminal, not shown to users
console.log('Debug info:', { data })
console.error('Error details:', error)
```

**Client-Side Logs**:
```typescript
// These appear in browser console
console.log('Client state:', state)
```

**Database Queries**:
```typescript
// Enable Drizzle query logging
import { db } from '@/lib/db/client'
// Check drizzle.config.ts for logging configuration
```

**Sandbox Debugging**:
- Check sandbox logs in Vercel dashboard
- Use `sandbox.commands.run('bash', ['-c', 'your-command'])` for debugging
- Verify environment variables are set correctly

---

## Additional Resources

- **User Documentation**: [README.md](./README.md)
- **Agent Guidelines**: [AGENTS.md](./AGENTS.md)
- **Next.js Docs**: https://nextjs.org/docs
- **Vercel Sandbox Docs**: https://vercel.com/docs/vercel-sandbox
- **Drizzle ORM Docs**: https://orm.drizzle.team/
- **shadcn/ui Docs**: https://ui.shadcn.com/

---

## Quick Reference

### Environment Variables

**Required (Infrastructure)**:
- `POSTGRES_URL` - Database connection string
- `SANDBOX_VERCEL_TOKEN` - Vercel API token
- `SANDBOX_VERCEL_TEAM_ID` - Vercel team ID
- `SANDBOX_VERCEL_PROJECT_ID` - Vercel project ID
- `JWE_SECRET` - Session encryption secret
- `ENCRYPTION_KEY` - Data encryption key

**Required (Authentication)** - at least one:
- `NEXT_PUBLIC_AUTH_PROVIDERS` - Enabled auth providers
- `NEXT_PUBLIC_GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` - GitHub OAuth
- `NEXT_PUBLIC_VERCEL_CLIENT_ID`, `VERCEL_CLIENT_SECRET` - Vercel OAuth

**Optional (AI Providers)** - global fallbacks:
- `ANTHROPIC_API_KEY` - Claude agent
- `OPENAI_API_KEY` - Codex and OpenCode agents
- `AI_GATEWAY_API_KEY` - AI Gateway (branch names, Codex)
- `CURSOR_API_KEY` - Cursor agent
- `GEMINI_API_KEY` - Gemini agent

**Optional (Configuration)**:
- `MAX_SANDBOX_DURATION` - Default sandbox timeout (minutes, default: 300)
- `MAX_MESSAGES_PER_DAY` - Rate limit (default: 5)

### Common Commands

```bash
# Development
pnpm dev                # Start dev server (user does this)
pnpm build              # Production build
pnpm start              # Start production server

# Database
pnpm db:generate        # Generate migrations
pnpm db:push            # Push schema to database
pnpm db:studio          # Open Drizzle Studio

# Code Quality
pnpm format             # Format with Prettier
pnpm format:check       # Check formatting
pnpm type-check         # TypeScript validation
pnpm lint               # ESLint checks
```

### File Locations

| What | Where |
|------|-------|
| Database schema | `lib/db/schema.ts` |
| API routes | `app/api/**/route.ts` |
| Server actions | `lib/actions/*.ts` |
| UI components | `components/*.tsx` |
| Agent implementations | `lib/sandbox/agents/*.ts` |
| Encryption utilities | `lib/crypto.ts` |
| Session management | `lib/session/*.ts` |
| GitHub client | `lib/github/*.ts` |
| Constants | `lib/constants.ts` |

---

**Last Updated**: December 23, 2025
**Maintained By**: Vercel Labs
**For Questions**: See GitHub Issues at https://github.com/vercel-labs/coding-agent-template/issues
