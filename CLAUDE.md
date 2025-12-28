# CLAUDE.md - AI Assistant Documentation

This document provides comprehensive guidance for AI assistants (like Claude Code) working on this codebase. It covers architecture, conventions, workflows, and critical rules to follow.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Codebase Structure](#codebase-structure)
4. [Database Schema](#database-schema)
5. [AI Agents](#ai-agents)
6. [Skills System](#skills-system)
7. [Subagents](#subagents)
8. [Development Workflows](#development-workflows)
9. [Security Guidelines](#security-guidelines)
10. [Code Quality Standards](#code-quality-standards)
11. [API Conventions](#api-conventions)
12. [Common Tasks](#common-tasks)
13. [Testing and Validation](#testing-and-validation)

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

## AI Agents

This section explains the AI coding agents that power task execution in this codebase.

### Overview

The application supports **6 different AI coding agents**, each with unique capabilities and implementations. Users can select which agent to use when creating a task.

| Agent | CLI Package | Model Support | MCP Support | Session Resumption |
|-------|-------------|---------------|-------------|-------------------|
| **Claude Code** | `@anthropic-ai/claude-code` | Yes (via `selectedModel`) | ✅ Yes | ✅ Yes |
| **Codex** | `openai-codex-cli` | Yes (via `selectedModel`) | ❌ No | ✅ Yes |
| **Copilot** | `@githubnext/github-copilot-cli` | Yes (via `selectedModel`) | ❌ No | ✅ Yes |
| **Cursor** | `cursor-ai` | Yes (via `selectedModel`) | ❌ No | ✅ Yes |
| **Gemini** | `@google/gemini-cli` | Yes (via `selectedModel`) | ❌ No | ❌ No |
| **OpenCode** | `opencode-cli` | Yes (via `selectedModel`) | ❌ No | ✅ Yes |

### Agent Architecture

All agents follow a common execution pattern:

```typescript
export async function executeAgentInSandbox(
  sandbox: Sandbox,
  instruction: string,
  agentType: AgentType,
  logger: TaskLogger,
  selectedModel?: string,
  mcpServers?: Connector[],
  onCancellationCheck?: () => Promise<boolean>,
  apiKeys?: {...},
  isResumed?: boolean,
  sessionId?: string,
  taskId?: string,
  agentMessageId?: string,
): Promise<AgentExecutionResult>
```

**Execution Flow**:
1. **Check cancellation** - Verify task hasn't been cancelled
2. **Install agent CLI** - Install agent package in sandbox (if not already installed)
3. **Authenticate** - Set up API keys from user or global environment
4. **Configure** - Create agent config files (model selection, MCP servers, etc.)
5. **Execute** - Run agent with user's instruction
6. **Stream logs** - Send real-time output to client via TaskLogger
7. **Detect changes** - Check if agent made any git changes
8. **Return result** - Return execution result with success status

### Agent Implementation Files

Each agent has its own implementation file in `lib/sandbox/agents/`:

```
lib/sandbox/agents/
├── index.ts          # Main dispatcher (routes to specific agent)
├── claude.ts         # Claude Code implementation (~471 lines)
├── codex.ts          # OpenAI Codex implementation (~378 lines)
├── copilot.ts        # GitHub Copilot implementation (~387 lines)
├── cursor.ts         # Cursor implementation (~553 lines)
├── gemini.ts         # Google Gemini implementation (~359 lines)
└── opencode.ts       # OpenCode implementation (~446 lines)
```

### Agent-Specific Details

#### Claude Code (`claude.ts`)

- **Package**: `@anthropic-ai/claude-code`
- **Key Features**:
  - Full MCP server support
  - Session resumption for follow-up messages
  - Configurable model selection
  - Custom config file generation
- **Authentication**: Requires `ANTHROPIC_API_KEY`
- **Config Location**: `$HOME/.config/claude/config.json`
- **Execution**: `claude --dangerouslySkipHuman --config-file <config>`

#### Codex (`codex.ts`)

- **Package**: `openai-codex-cli`
- **Key Features**:
  - Session resumption support
  - AI Gateway integration
  - Configurable model selection
- **Authentication**: Requires `OPENAI_API_KEY` or `AI_GATEWAY_API_KEY`
- **Execution**: `codex <instruction>`

#### Copilot (`copilot.ts`)

- **Package**: `@githubnext/github-copilot-cli`
- **Key Features**:
  - Uses user's GitHub token for authentication
  - Session resumption support
  - Model selection support
- **Authentication**: Requires user's GitHub OAuth token
- **Execution**: `copilot <instruction>`

#### Cursor (`cursor.ts`)

- **Package**: `cursor-ai`
- **Key Features**:
  - Session-based conversation support
  - Session resumption for follow-ups
  - Model configuration
- **Authentication**: Requires `CURSOR_API_KEY`
- **Execution**: `cursor --session <id> <instruction>`

#### Gemini (`gemini.ts`)

- **Package**: `@google/gemini-cli`
- **Key Features**:
  - Google's Gemini models
  - Model selection support
- **Authentication**: Requires `GEMINI_API_KEY`
- **Limitations**: No session resumption
- **Execution**: `gemini <instruction>`

#### OpenCode (`opencode.ts`)

- **Package**: `opencode-cli`
- **Key Features**:
  - Open-source coding agent
  - Session resumption support
  - Model configuration
- **Authentication**: Requires `OPENAI_API_KEY`
- **Execution**: `opencode --session <id> <instruction>`

### Agent Selection

Users select agents via the UI when creating a task. The selected agent is stored in the `tasks.selectedAgent` field:

```typescript
// In database schema
selectedAgent: text('selected_agent', {
  enum: ['claude', 'codex', 'copilot', 'cursor', 'gemini', 'opencode']
}).default('claude')
```

### API Key Management

Agents use a **fallback system** for API keys:

1. **User-provided keys** (from `keys` table) - Takes precedence
2. **Global environment variables** - Fallback if user hasn't provided keys

Example from `lib/sandbox/agents/index.ts`:

```typescript
// Temporarily override process.env with user's API keys if provided
if (apiKeys?.ANTHROPIC_API_KEY) process.env.ANTHROPIC_API_KEY = apiKeys.ANTHROPIC_API_KEY
if (apiKeys?.OPENAI_API_KEY) process.env.OPENAI_API_KEY = apiKeys.OPENAI_API_KEY
// ... etc
```

After execution, original environment variables are restored.

### Adding a New Agent

To add a new agent to the system:

1. **Create agent file**: `lib/sandbox/agents/new-agent.ts`
2. **Implement execution function**:
   ```typescript
   export async function executeNewAgentInSandbox(
     sandbox: Sandbox,
     instruction: string,
     logger: TaskLogger,
     selectedModel?: string,
     mcpServers?: Connector[],
     isResumed?: boolean,
     sessionId?: string,
   ): Promise<AgentExecutionResult>
   ```
3. **Update agent index**: Add to `lib/sandbox/agents/index.ts`
4. **Update schema**: Add to enum in `lib/db/schema.ts`
5. **Add logo**: Create `components/logos/new-agent.tsx`
6. **Update UI**: Add option to task creation form

### MCP Server Integration (Claude Only)

**Model Context Protocol (MCP)** servers extend Claude Code with additional tools and capabilities.

Users can connect MCP servers via the "Connectors" tab:

```typescript
// Connector types
type: 'local' | 'remote'

// Remote MCP servers
baseUrl: string
oauthClientId?: string
oauthClientSecret?: string

// Local MCP servers
command: string

// Environment variables (encrypted)
env: Record<string, string>
```

When executing Claude Code, MCP servers are configured in the agent's config file:

```json
{
  "mcpServers": {
    "server-name": {
      "url": "https://mcp.example.com",
      "auth": {
        "type": "oauth",
        "clientId": "...",
        "clientSecret": "..."
      },
      "env": {
        "API_KEY": "..."
      }
    }
  }
}
```

### Agent Execution Result

All agents return a standardized result:

```typescript
interface AgentExecutionResult {
  success: boolean           // Whether execution succeeded
  output?: string           // Agent's output
  agentResponse?: string    // Agent's final response/message
  cliName?: string          // Name of CLI used
  changesDetected?: boolean // Whether git changes were made
  error?: string            // Error message if failed
  streamingLogs?: unknown[] // Real-time logs
  logs?: LogEntry[]         // Structured log entries
  sessionId?: string        // Session ID for resumption
}
```

### Session Resumption

Most agents support **session resumption**, allowing users to send follow-up messages without restarting:

**How it works**:
1. Initial task execution creates a session ID
2. Session ID stored in `tasks.agentSessionId`
3. Follow-up messages use `isResumed=true` and pass the session ID
4. Agent CLI reconnects to existing session

**Agents with resumption**:
- ✅ Claude Code
- ✅ Codex
- ✅ Copilot
- ✅ Cursor
- ✅ OpenCode
- ❌ Gemini (no session support)

### Task Logger Integration

All agents use the `TaskLogger` class to stream logs:

```typescript
// Info messages
await logger.info('Installing dependencies')

// Commands being executed
await logger.command('npm install')

// Errors
await logger.error('Build failed')

// Success messages
await logger.success('Task completed')

// Progress updates
await logger.updateProgress(50, 'Running tests')
```

**Important**: All log messages must use **static strings only** (see Security Guidelines).

---

## Skills System

**Current Status**: This codebase does **not** currently implement a skills system.

### What is a Skills System?

A **skills system** would allow agents to use pre-defined, reusable capabilities or tools to accomplish specific tasks. Think of skills as specialized functions that agents can invoke.

### Why This Doesn't Exist Yet

The current architecture relies on:
1. **Agent-native capabilities** - Each AI agent (Claude, Codex, etc.) brings its own built-in skills
2. **MCP servers** - Claude Code can connect to MCP servers for extended capabilities
3. **Direct CLI execution** - Agents execute directly in sandboxes with full system access

### How a Skills System Could Be Added

If you wanted to implement a skills system in the future, here's the recommended approach:

#### 1. Define Skill Interface

```typescript
// lib/skills/types.ts
export interface Skill {
  id: string
  name: string
  description: string
  category: 'code' | 'testing' | 'deployment' | 'analysis'
  execute: (context: SkillContext) => Promise<SkillResult>
}

export interface SkillContext {
  sandbox: Sandbox
  logger: TaskLogger
  task: Task
  params: Record<string, unknown>
}

export interface SkillResult {
  success: boolean
  output: string
  error?: string
}
```

#### 2. Create Skill Implementations

```typescript
// lib/skills/code/format.ts
export const formatCodeSkill: Skill = {
  id: 'format-code',
  name: 'Format Code',
  description: 'Format code using Prettier',
  category: 'code',
  execute: async (context) => {
    const { sandbox, logger } = context
    await logger.info('Formatting code with Prettier')

    const result = await runInProject(sandbox, 'pnpm', ['format'])

    return {
      success: result.success,
      output: result.output || '',
      error: result.error,
    }
  },
}
```

#### 3. Create Skill Registry

```typescript
// lib/skills/registry.ts
import { Skill } from './types'
import { formatCodeSkill } from './code/format'
import { runTestsSkill } from './testing/run-tests'

export class SkillRegistry {
  private skills = new Map<string, Skill>()

  constructor() {
    this.register(formatCodeSkill)
    this.register(runTestsSkill)
  }

  register(skill: Skill) {
    this.skills.set(skill.id, skill)
  }

  get(id: string): Skill | undefined {
    return this.skills.get(id)
  }

  list(): Skill[] {
    return Array.from(this.skills.values())
  }
}

export const skillRegistry = new SkillRegistry()
```

#### 4. Integrate with Agents

Agents could invoke skills through a special syntax or API:

```typescript
// In agent execution
if (instruction.includes('@skill:')) {
  const skillId = extractSkillId(instruction)
  const skill = skillRegistry.get(skillId)

  if (skill) {
    const result = await skill.execute({
      sandbox,
      logger,
      task,
      params: extractParams(instruction),
    })
    return result
  }
}
```

#### 5. Database Schema

Add a skills table to track user-defined or custom skills:

```typescript
export const skills = pgTable('skills', {
  id: text('id').primaryKey(),
  userId: text('user_id').references(() => users.id),
  name: text('name').notNull(),
  description: text('description'),
  code: text('code').notNull(), // JavaScript code to execute
  category: text('category'),
  isPublic: boolean('is_public').default(false),
  createdAt: timestamp('created_at').defaultNow(),
})
```

### Alternative: Use MCP Servers

Instead of building a custom skills system, you can leverage **MCP servers** (which Claude Code already supports):

- Create MCP servers for specific capabilities
- Users connect them via the Connectors tab
- Claude Code can use them during task execution

This approach:
- ✅ Already implemented for Claude Code
- ✅ Standard protocol (MCP)
- ✅ Can be shared across different projects
- ✅ More flexible and powerful than custom skills

---

## Subagents

**Current Status**: This codebase does **not** currently implement a subagent system.

### What are Subagents?

**Subagents** are specialized AI agents that handle specific subtasks delegated by a main agent. They enable:
- **Task decomposition** - Breaking complex tasks into smaller pieces
- **Parallel execution** - Running multiple subtasks simultaneously
- **Specialization** - Using different agents for different types of work

### Why This Doesn't Exist Yet

The current architecture uses a **single-agent model**:
1. User creates a task with one selected agent
2. That agent executes the entire task from start to finish
3. Follow-up messages go to the same agent session

This is simpler and works well for most use cases.

### When Subagents Would Be Useful

Subagents could improve the system for:

1. **Complex multi-step tasks**:
   - Main agent: Plans the overall approach
   - Code subagent: Implements the features
   - Test subagent: Writes and runs tests
   - Review subagent: Reviews code quality

2. **Specialized expertise**:
   - Use Claude for planning and architecture
   - Use Codex for code generation
   - Use Gemini for documentation

3. **Parallel execution**:
   - Multiple subagents work on different files simultaneously
   - Faster completion for large tasks

### How Subagents Could Be Implemented

Here's a design for adding subagents:

#### 1. Subagent Types

```typescript
// lib/sandbox/subagents/types.ts
export interface SubagentTask {
  id: string
  type: 'code' | 'test' | 'review' | 'docs' | 'debug'
  instruction: string
  agentType: AgentType
  parentTaskId: string
  dependencies?: string[] // IDs of subtasks that must complete first
}

export interface SubagentResult {
  taskId: string
  success: boolean
  output: string
  changes: GitChange[]
  error?: string
}
```

#### 2. Orchestrator

```typescript
// lib/sandbox/subagents/orchestrator.ts
export class SubagentOrchestrator {
  async executeWithSubagents(
    mainTask: Task,
    sandbox: Sandbox,
    logger: TaskLogger,
  ): Promise<AgentExecutionResult> {
    // 1. Main agent analyzes task and creates subtasks
    const subtasks = await this.planSubtasks(mainTask)

    // 2. Execute subtasks (respecting dependencies)
    const results = await this.executeSubtasks(subtasks, sandbox, logger)

    // 3. Main agent synthesizes results
    const finalResult = await this.synthesizeResults(results, sandbox, logger)

    return finalResult
  }

  private async executeSubtasks(
    subtasks: SubagentTask[],
    sandbox: Sandbox,
    logger: TaskLogger,
  ): Promise<SubagentResult[]> {
    const completed = new Map<string, SubagentResult>()
    const queue = [...subtasks]

    while (queue.length > 0) {
      // Find subtasks with satisfied dependencies
      const ready = queue.filter(task =>
        !task.dependencies ||
        task.dependencies.every(dep => completed.has(dep))
      )

      // Execute ready subtasks in parallel
      const results = await Promise.all(
        ready.map(task => this.executeSubtask(task, sandbox, logger))
      )

      // Mark as completed
      results.forEach(result => completed.set(result.taskId, result))

      // Remove from queue
      queue.splice(0, ready.length)
    }

    return Array.from(completed.values())
  }

  private async executeSubtask(
    subtask: SubagentTask,
    sandbox: Sandbox,
    logger: TaskLogger,
  ): Promise<SubagentResult> {
    await logger.info(`Starting subtask: ${subtask.type}`)

    const result = await executeAgentInSandbox(
      sandbox,
      subtask.instruction,
      subtask.agentType,
      logger,
    )

    return {
      taskId: subtask.id,
      success: result.success,
      output: result.output || '',
      changes: await this.detectChanges(sandbox),
      error: result.error,
    }
  }
}
```

#### 3. Database Schema

Track subagent executions:

```typescript
export const subagentTasks = pgTable('subagent_tasks', {
  id: text('id').primaryKey(),
  parentTaskId: text('parent_task_id')
    .notNull()
    .references(() => tasks.id, { onDelete: 'cascade' }),
  type: text('type', {
    enum: ['code', 'test', 'review', 'docs', 'debug'],
  }).notNull(),
  agentType: text('agent_type').notNull(),
  instruction: text('instruction').notNull(),
  status: text('status', {
    enum: ['pending', 'running', 'completed', 'failed'],
  }).notNull(),
  result: jsonb('result').$type<SubagentResult>(),
  createdAt: timestamp('created_at').defaultNow(),
  completedAt: timestamp('completed_at'),
})
```

#### 4. UI Integration

Show subagent progress in the task detail view:

```typescript
// components/task-details.tsx
<div className="subagents">
  <h3>Subagents</h3>
  {subagentTasks.map(subtask => (
    <SubagentCard
      key={subtask.id}
      subtask={subtask}
      status={subtask.status}
    />
  ))}
</div>
```

#### 5. Agent Integration

Update main agent execution to support delegation:

```typescript
// In lib/sandbox/agents/claude.ts
export async function executeClaudeWithSubagents(
  sandbox: Sandbox,
  instruction: string,
  logger: TaskLogger,
) {
  // 1. Ask Claude to plan subtasks
  const plan = await claudePlanner.analyze(instruction)

  // 2. Create subtasks
  const subtasks = plan.subtasks.map(st => ({
    id: generateId(),
    type: st.type,
    instruction: st.instruction,
    agentType: st.recommendedAgent,
    dependencies: st.dependencies,
  }))

  // 3. Execute with orchestrator
  const orchestrator = new SubagentOrchestrator()
  return await orchestrator.executeWithSubagents(subtasks, sandbox, logger)
}
```

### Simpler Alternative: Sequential Agents

Instead of true subagents, you could implement **sequential agent chaining**:

```typescript
// Execute multiple agents in sequence
const agents = ['claude', 'codex', 'copilot']
let context = initialInstruction

for (const agentType of agents) {
  const result = await executeAgentInSandbox(sandbox, context, agentType, logger)
  context = `Previous result: ${result.output}\n\nNext step: ...`
}
```

This is simpler but less flexible than true subagents.

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
