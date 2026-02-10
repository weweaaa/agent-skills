---
name: copilot-sdk
description: This skill provides guidance for creating agents and applications with the GitHub Copilot SDK. It should be used when the user wants to create, modify, or work on software that uses the GitHub Copilot SDK in TypeScript, Python, Go, or .NET. The skill covers SDK usage patterns, CLI configuration, custom tools, MCP servers, hooks, BYOK providers, infinite sessions, skills, and custom agents.
---

# GitHub Copilot SDK

## Overview

The GitHub Copilot SDK is a multi-platform agent runtime that embeds Copilot's agentic workflows into applications. It exposes the same engine behind Copilot CLI, enabling programmatic invocation without requiring custom orchestration development.

**Status:** Technical Preview (suitable for development and testing)

**Supported Languages:** TypeScript/Node.js, Python, Go, .NET

**Protocol Version:** 2

## Primary Documentation

- [GitHub Copilot SDK Repository](https://github.com/github/copilot-sdk)
- [Getting Started Guide](https://github.com/github/copilot-sdk/blob/main/docs/getting-started.md)

### Language-Specific SDK Docs

- [Node.js/TypeScript SDK](https://github.com/github/copilot-sdk/blob/main/nodejs/README.md)
- [Python SDK](https://github.com/github/copilot-sdk/blob/main/python/README.md)
- [Go SDK](https://github.com/github/copilot-sdk/tree/main/go)
- [.NET SDK](https://github.com/github/copilot-sdk/tree/main/dotnet)

### CLI and Configuration Docs

- [About GitHub Copilot CLI](https://docs.github.com/en/copilot/concepts/agents/about-copilot-cli)
- [Using GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli)
- [Creating Custom Agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents)
- [Custom Agents Configuration Reference](https://docs.github.com/en/copilot/reference/custom-agents-configuration)
- [Enhancing Agent Mode with MCP](https://docs.github.com/en/copilot/tutorials/enhance-agent-mode-with-mcp)
- [Supported AI Models](https://docs.github.com/en/copilot/reference/ai-models/supported-models)

### Feature-Specific Docs

- [Hooks Overview](https://github.com/github/copilot-sdk/blob/main/docs/hooks/overview.md)
- [MCP Server Integration](https://github.com/github/copilot-sdk/blob/main/docs/mcp/overview.md)
- [Session Persistence](https://github.com/github/copilot-sdk/blob/main/docs/guides/session-persistence.md)
- [Skills System](https://github.com/github/copilot-sdk/blob/main/docs/guides/skills.md)
- [BYOK Authentication](https://github.com/github/copilot-sdk/blob/main/docs/auth/byok.md)
- [Debugging Guide](https://github.com/github/copilot-sdk/blob/main/docs/debugging.md)

---

## Prerequisites

1. **GitHub Copilot Subscription** - Pro, Pro+, Business, or Enterprise
2. **GitHub Copilot CLI** - Installed and authenticated (`copilot --version`)
3. **Runtime:** Node.js 18+, Python 3.8+, Go 1.21+, or .NET 8.0+

## Installation

| Language | Command |
|----------|---------|
| TypeScript/Node.js | `npm install @github/copilot-sdk` |
| Python | `pip install github-copilot-sdk` |
| Go | `go get github.com/github/copilot-sdk/go` |
| .NET | `dotnet add package GitHub.Copilot.SDK` |

## Architecture

```
Application → SDK Client → JSON-RPC (stdio or TCP) → Copilot CLI (server mode)
```

The SDK manages CLI lifecycle automatically. External server connections supported via `cliUrl` / `cli_url`.

---

## Quick Start (TypeScript)

```typescript
import { CopilotClient } from "@github/copilot-sdk";

const client = new CopilotClient();
await client.start();

const session = await client.createSession({ model: "gpt-5" });

// Register handler BEFORE send()
session.on((event) => {
  if (event.type === "assistant.message") {
    console.log(event.data.content);
  }
});

await session.send({ prompt: "What is 2 + 2?" });

await session.destroy();
await client.stop();
```

**Critical:** Register event handlers **before** calling `send()` to capture all events.

For complete examples in all languages, see `references/working-examples.md`.

---

## Core Concepts

### Client

Main entry point. Manages CLI server lifecycle and session creation.

**Operations:** `start()`, `stop()`, `forceStop()`, `createSession()`, `resumeSession()`, `listSessions()`, `deleteSession()`, `ping()`, `getState()`, `listModels()`, `getAuthStatus()`, `getForegroundSessionId()`, `setForegroundSessionId()`, `on()`

**Config (CopilotClientOptions):**

| Option | TypeScript | Python | Go | .NET | Default |
|--------|-----------|--------|-----|------|---------|
| CLI path | `cliPath` | `cli_path` | `CLIPath` | `CliPath` | bundled CLI |
| CLI extra args | `cliArgs` | — | — | `CliArgs` | `[]` |
| Working directory | `cwd` | `cwd` | `Cwd` | `Cwd` | process cwd |
| Port (TCP) | `port` | `port` | `Port` | `Port` | `0` (random) |
| Use stdio | `useStdio` | `use_stdio` | `UseStdio` | `UseStdio` | `true` |
| CLI URL | `cliUrl` | `cli_url` | `CLIUrl` | `CliUrl` | — |
| Log level | `logLevel` | `log_level` | `LogLevel` | `LogLevel` | `"info"` |
| Auto start | `autoStart` | `auto_start` | `AutoStart` | `AutoStart` | `true` |
| Auto restart | `autoRestart` | `auto_restart` | `AutoRestart` | `AutoRestart` | `true` |
| Environment | `env` | `env` | `Env` | `Environment` | inherited |
| GitHub token | `githubToken` | `github_token` | `GithubToken` | `GithubToken` | — |
| Use logged-in user | `useLoggedInUser` | `use_logged_in_user` | `UseLoggedInUser` | `UseLoggedInUser` | `true` |
| Logger | — | — | — | `Logger` (ILogger) | — |

### Session

Individual conversation context with message history.

**Operations:** `send()`, `sendAndWait()`, `on()`, `abort()`, `getMessages()`, `destroy()`

**Properties:** `sessionId`, `workspacePath`

**Config (SessionConfig):**

| Option | TypeScript | Python | Go | .NET | Description |
|--------|-----------|--------|-----|------|-------------|
| Session ID | `sessionId` | `session_id` | `SessionID` | `SessionId` | Custom session ID |
| Model | `model` | `model` | `Model` | `Model` | Model name (required for BYOK) |
| Reasoning effort | `reasoningEffort` | `reasoning_effort` | `ReasoningEffort` | `ReasoningEffort` | `"low"` / `"medium"` / `"high"` / `"xhigh"` |
| Config dir | `configDir` | `config_dir` | `ConfigDir` | `ConfigDir` | Override config directory |
| Tools | `tools` | `tools` | `Tools` | `Tools` | Custom tools |
| System message | `systemMessage` | `system_message` | `SystemMessage` | `SystemMessage` | System prompt config |
| Available tools | `availableTools` | `available_tools` | `AvailableTools` | `AvailableTools` | Tool whitelist |
| Excluded tools | `excludedTools` | `excluded_tools` | `ExcludedTools` | `ExcludedTools` | Tool blacklist |
| Provider (BYOK) | `provider` | `provider` | `Provider` | `Provider` | Custom API provider |
| Permission handler | `onPermissionRequest` | `on_permission_request` | `OnPermissionRequest` | `OnPermissionRequest` | Permission callback |
| User input handler | `onUserInputRequest` | `on_user_input_request` | `OnUserInputRequest` | `OnUserInputRequest` | ask_user callback |
| Hooks | `hooks` | `hooks` | `Hooks` | `Hooks` | Hook handlers |
| Working directory | `workingDirectory` | `working_directory` | `WorkingDirectory` | `WorkingDirectory` | Session working dir |
| Streaming | `streaming` | `streaming` | `Streaming` | `Streaming` | Enable streaming |
| MCP servers | `mcpServers` | `mcp_servers` | `MCPServers` | `McpServers` | MCP server configs |
| Custom agents | `customAgents` | `custom_agents` | `CustomAgents` | `CustomAgents` | Custom agent configs |
| Skill directories | `skillDirectories` | `skill_directories` | `SkillDirectories` | `SkillDirectories` | Skill load paths |
| Disabled skills | `disabledSkills` | `disabled_skills` | `DisabledSkills` | `DisabledSkills` | Disabled skill names |
| Infinite sessions | `infiniteSessions` | `infinite_sessions` | `InfiniteSessions` | `InfiniteSessions` | Auto-compaction config |

### Events

Key events during processing:

| Event | Purpose | Ephemeral |
|-------|---------|-----------|
| `assistant.message` | Complete response | No |
| `assistant.message_delta` | Streaming chunk | Yes |
| `assistant.reasoning` | Chain-of-thought | No |
| `assistant.reasoning_delta` | Reasoning chunk | Yes |
| `session.idle` | Ready for next prompt | Yes |
| `tool.execution_start` | Tool invocation began | No |
| `tool.execution_complete` | Tool execution completed | No |
| `session.usage_info` | Token/usage metadata | Yes |
| `skill.invoked` | Skill was loaded | No |
| `subagent.started` | Subagent began | No |
| `hook.start` / `hook.end` | Hook lifecycle | No |

For full event lifecycle (31+ event types) and SessionEvent structure, see `references/event-system.md`.

### Streaming

- `streaming: false` (default) - Content arrives all at once
- `streaming: true` - Content arrives incrementally via `assistant.message_delta`

Final `assistant.message` **always fires** regardless of streaming setting.

---

## System Message Configuration

Two modes for system prompt customization:

| Mode | Behavior |
|------|----------|
| `append` (default) | SDK foundation + optional custom content appended |
| `replace` | Full control, caller provides entire system message (removes all SDK guardrails) |

```typescript
// Append mode (default) — adds to existing system message
{ mode: "append", content: "Extra instructions..." }

// Replace mode — full control, removes guardrails
{ mode: "replace", content: "Complete system message..." }
```

---

## Custom Tools

**TypeScript (Zod):**
```typescript
import { defineTool } from "@github/copilot-sdk";
import { z } from "zod";

const tool = defineTool("lookup_issue", {
  description: "Fetch issue details",
  parameters: z.object({ id: z.string() }),
  handler: async ({ id }) => fetchIssue(id),
});
```

**Python (Pydantic):**
```python
from pydantic import BaseModel, Field
from copilot import define_tool

class IssueParams(BaseModel):
    id: str = Field(description="Issue identifier")

@define_tool(description="Fetch issue details")
async def lookup_issue(params: IssueParams) -> dict:
    return fetch_issue(params.id)
```

**Go (DefineTool with generics):**
```go
type IssueParams struct {
    ID string `json:"id" jsonschema:"Issue identifier"`
}

tool := copilot.DefineTool("lookup_issue", "Fetch issue details",
    func(params IssueParams, inv copilot.ToolInvocation) (any, error) {
        return fetchIssue(params.ID), nil
    })
```

**C# (AIFunctionFactory):**
```csharp
using Microsoft.Extensions.AI;

var tool = AIFunctionFactory.Create(
    (string id) => new { Status = "found", Title = "Bug report" },
    "lookup_issue",
    "Fetch issue details"
);
```

### Tool Result Types

Tool handlers can return:
- `string` — passes through as success
- `ToolResultObject` — full control over result type, binary data, telemetry
- Any other type — JSON-serialized as success

**ToolResultType values:** `"success"`, `"failure"`, `"rejected"`, `"denied"`

For complete tool examples in all languages, see `references/working-examples.md`.

---

## Hook System

Hooks intercept and modify behavior at key points in the session lifecycle. Six hook types available:

| Hook | Trigger | Can Modify |
|------|---------|------------|
| `onPreToolUse` | Before tool execution | Args, permission decision |
| `onPostToolUse` | After tool execution | Result, additional context |
| `onUserPromptSubmitted` | User sends prompt | Prompt text, additional context |
| `onSessionStart` | Session begins | Config, additional context |
| `onSessionEnd` | Session ends | Cleanup actions, summary |
| `onErrorOccurred` | Error happens | Error handling strategy |

**TypeScript example:**
```typescript
const session = await client.createSession({
  hooks: {
    onPreToolUse: async (input, { sessionId }) => ({
      permissionDecision: "allow",
      additionalContext: "Approved by hook",
    }),
    onPostToolUse: async (input) => ({
      additionalContext: `Tool ${input.toolName} completed`,
    }),
    onUserPromptSubmitted: async (input) => ({
      modifiedPrompt: input.prompt + "\nBe concise.",
    }),
    onErrorOccurred: async (input) => ({
      errorHandling: input.recoverable ? "retry" : "abort",
      retryCount: 3,
    }),
  },
});
```

For full hook API and all language examples, see `references/working-examples.md`.

---

## BYOK (Bring Your Own Key)

Use custom model providers (OpenAI-compatible, Azure, Anthropic):

```typescript
const session = await client.createSession({
  model: "my-model",
  provider: {
    type: "openai",         // "openai" | "azure" | "anthropic"
    baseUrl: "https://api.example.com/v1",
    apiKey: "sk-...",
    wireApi: "completions",  // "completions" | "responses"
  },
});
```

**Provider config fields:** `type`, `baseUrl`, `apiKey`, `bearerToken` (takes precedence over apiKey), `wireApi`, `azure.apiVersion`

---

## Infinite Sessions

Automatic context management with compaction and workspace persistence:

```typescript
const session = await client.createSession({
  infiniteSessions: {
    enabled: true,                         // default: true
    backgroundCompactionThreshold: 0.80,   // start background compaction at 80%
    bufferExhaustionThreshold: 0.95,       // block at 95% until compaction finishes
  },
});
```

Workspace path available via `session.workspacePath`.

---

## Permission Handler

Handle permission requests for shell execution, file writes, MCP calls, etc.:

```typescript
const session = await client.createSession({
  onPermissionRequest: async (request) => {
    // request.kind: "shell" | "write" | "mcp" | "read" | "url"
    return { kind: "approved" };
    // Or: "denied-by-rules", "denied-interactively-by-user", etc.
  },
});
```

---

## User Input Handler (ask_user)

Enable the agent to ask questions interactively:

```typescript
const session = await client.createSession({
  onUserInputRequest: async (request) => {
    // request.question, request.choices, request.allowFreeform
    return { answer: "Yes", wasFreeform: true };
  },
});
```

---

## Message Attachments

Attach files, directories, or selections to messages:

```typescript
await session.send({
  prompt: "Review this code",
  attachments: [
    { type: "file", path: "/path/to/file.ts" },
    { type: "directory", path: "/path/to/dir" },
    { type: "selection", filePath: "/file.ts", displayName: "snippet",
      selection: { start: { line: 1, character: 0 }, end: { line: 10, character: 0 } } },
  ],
  mode: "enqueue",  // "enqueue" (default) | "immediate"
});
```

---

## Authentication

**Priority order:**
1. Explicit `githubToken` in client options
2. HMAC key (`CAPI_HMAC_KEY` / `COPILOT_HMAC_KEY`)
3. Direct API token (`GITHUB_COPILOT_API_TOKEN` + `COPILOT_API_URL`)
4. Environment variable tokens (`COPILOT_GITHUB_TOKEN` → `GH_TOKEN` → `GITHUB_TOKEN`)
5. Saved OAuth credentials
6. GitHub CLI (`gh auth`)

Check status: `client.getAuthStatus()` → `{ isAuthenticated, authType, host, login, statusMessage }`

---

## Session Lifecycle Events (Client-level)

Subscribe to session lifecycle notifications:

```typescript
client.on("session.created", (event) => { /* event.sessionId, event.metadata */ });
client.on("session.deleted", (event) => { /* ... */ });
client.on("session.updated", (event) => { /* ... */ });
client.on("session.foreground", (event) => { /* ... */ });
client.on("session.background", (event) => { /* ... */ });
```

---

## Session Persistence & Resume

Sessions persist to `~/.copilot/session-state/{sessionId}/` with checkpoints, plan, and files.

```typescript
// Resume a session
const session = await client.resumeSession("session-id-123", {
  model: "gpt-5",         // Can change model on resume
  tools: [myTool],        // Must re-provide tools
  provider: providerCfg,  // Must re-provide BYOK config (not persisted)
  disableResume: false,    // Set true to skip session.resume event
});
```

**Note:** Provider/API keys are **not** persisted for security. CLI has 30-minute idle timeout.

---

## Language Conventions

| Concept | TypeScript | Python | Go | .NET |
|---------|------------|--------|----|----|
| Client class | `CopilotClient` | `CopilotClient` | `Client` | `CopilotClient` |
| Create session | `createSession()` | `create_session()` | `CreateSession()` | `CreateSessionAsync()` |
| Send message | `send()` / `sendAndWait()` | `send()` / `send_and_wait()` | `Send()` | `SendAsync()` / `SendAndWaitAsync()` |
| Resume session | `resumeSession()` | `resume_session()` | `ResumeSession()` | `ResumeSessionAsync()` |
| List sessions | `listSessions()` | `list_sessions()` | `ListSessions()` | `ListSessionsAsync()` |
| Delete session | `deleteSession()` | `delete_session()` | `DeleteSession()` | `DeleteSessionAsync()` |
| List models | `listModels()` | `list_models()` | `ListModels()` | `ListModelsAsync()` |
| Auth status | `getAuthStatus()` | — | — | — |
| Session config | `{ model, streaming }` | `{"model", "streaming"}` | `&SessionConfig{}` | `new SessionConfig{}` |
| Event handler | `session.on(fn)` | `session.on(fn)` | `session.On(fn)` | `session.On(fn)` |
| Delta content | `deltaContent` | `delta_content` | `DeltaContent` | `DeltaContent` |
| System message | `systemMessage` | `system_message` | `SystemMessage` | `SystemMessage` |
| CLI path option | `cliPath` | `cli_path` | `CLIPath` | `CliPath` |
| Define tool | `defineTool()` | `@define_tool()` | `DefineTool()` | `AIFunctionFactory.Create()` |
| Stop errors | `stop()` → string[] | `stop()` → list[StopError] | `Stop()` → error | `StopAsync()` |

---

## CLI Configuration

Config stored in `~/.copilot/` (or `$XDG_CONFIG_HOME/copilot/`):
- `config.json` - General configuration
- `mcp-config.json` - MCP server definitions

For custom agents and MCP setup, see `references/cli-agents-mcp.md`.

---

## Troubleshooting Quick Reference

| Problem | Solution |
|---------|----------|
| Events fire but content empty | Use `event.data.content`, not `event.content` |
| Handler never fires | Register **before** `send()` |
| Python enum issues | Use `event.type.value` |
| Go nil pointer | Check `!= nil` before dereferencing |
| BYOK session resume fails | Must re-provide `provider` config (not persisted) |
| Missing `ask_user` tool | Must provide `onUserInputRequest` handler |
| Hooks not firing | Must provide `hooks` in SessionConfig |

For debugging techniques, see `references/troubleshooting.md`.

---

## Skill References

Detailed documentation in this skill:

- `references/working-examples.md` - Complete examples for all languages, tools, hooks, BYOK, MCP, attachments
- `references/event-system.md` - All 31+ event types, lifecycle, SessionEvent structure, ephemeral flags
- `references/troubleshooting.md` - Common issues, debugging techniques, hook/MCP/BYOK troubleshooting
- `references/cli-agents-mcp.md` - CLI configuration, custom agents, MCP server setup, skills system

---

## Additional Resources

- [SDK Releases](https://github.com/github/copilot-sdk/releases)
- [awesome-copilot](https://github.com/github/awesome-copilot)
