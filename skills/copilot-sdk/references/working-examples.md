# Complete Working Examples

**Critical:** Register event handlers **before** calling `send()` to capture all events.

---

## TypeScript/Node.js — Basic Usage

```typescript
import { CopilotClient, SessionEvent } from "@github/copilot-sdk";

async function main() {
  const client = new CopilotClient();
  await client.start();

  const session = await client.createSession({
    model: "gpt-5",
    streaming: true,
  });

  const done = new Promise<string>((resolve) => {
    let content = "";

    // Register handler BEFORE send()
    session.on((event: SessionEvent) => {
      if (event.type === "assistant.message_delta") {
        process.stdout.write(event.data.deltaContent ?? "");
      }
      if (event.type === "assistant.message") {
        content = event.data.content ?? "";
      }
      if (event.type === "session.idle") {
        resolve(content);
      }
    });
  });

  await session.send({ prompt: "What is 2 + 2?" });
  const response = await done;

  console.log("\n\nFinal response:", response);

  await session.destroy();
  await client.stop();
}

main();
```

## TypeScript — Using sendAndWait()

```typescript
import { CopilotClient } from "@github/copilot-sdk";

const client = new CopilotClient({ logLevel: "info" });
const session = await client.createSession();

// Listen to all events
session.on((event) => {
  console.log(`Event [${event.type}]:`, JSON.stringify(event.data, null, 2));
});

// sendAndWait blocks until session.idle and returns the assistant.message event
const result = await session.sendAndWait({ prompt: "Tell me 2+2" });
console.log("Response:", result?.data.content);

await session.destroy();
await client.stop();
```

---

## Python

```python
import asyncio
from copilot import CopilotClient

async def main():
    client = CopilotClient()
    await client.start()

    session = await client.create_session({
        "model": "gpt-5",
        "streaming": True,
    })

    done = asyncio.Event()
    final_content = ""

    def on_event(event):
        nonlocal final_content
        # Access type via .value for the string
        event_type = event.type.value

        if event_type == "assistant.message_delta":
            print(event.data.delta_content, end="", flush=True)
        elif event_type == "assistant.message":
            final_content = event.data.content
        elif event_type == "session.idle":
            done.set()

    # Register handler BEFORE send()
    session.on(on_event)

    await session.send({"prompt": "What is 2 + 2?"})
    await done.wait()

    print(f"\n\nFinal response: {final_content}")

    await session.destroy()
    await client.stop()

asyncio.run(main())
```

---

## Go

```go
package main

import (
    "context"
    "fmt"
    copilot "github.com/github/copilot-sdk/go"
)

func main() {
    ctx := context.Background()
    client := copilot.NewClient(nil)
    if err := client.Start(ctx); err != nil {
        panic(err)
    }
    defer client.Stop()

    session, err := client.CreateSession(ctx, &copilot.SessionConfig{
        Model:     "gpt-5",
        Streaming: true,
    })
    if err != nil {
        panic(err)
    }
    defer session.Destroy()

    done := make(chan string)
    var finalContent string

    // Register handler BEFORE Send()
    session.On(func(event copilot.SessionEvent) {
        switch event.Type {
        case "assistant.message_delta":
            if event.Data.DeltaContent != nil {
                fmt.Print(*event.Data.DeltaContent)
            }
        case "assistant.message":
            if event.Data.Content != nil {
                finalContent = *event.Data.Content
            }
        case "session.idle":
            done <- finalContent
        }
    })

    _, err = session.Send(ctx, copilot.MessageOptions{Prompt: "What is 2 + 2?"})
    if err != nil {
        panic(err)
    }

    response := <-done
    fmt.Printf("\n\nFinal response: %s\n", response)
}
```

---

## .NET (C#)

```csharp
using GitHub.Copilot.SDK;

await using var client = new CopilotClient();
await client.StartAsync();

await using var session = await client.CreateSessionAsync(new SessionConfig
{
    Model = "gpt-5",
    Streaming = true
});

var done = new TaskCompletionSource<string>();
var finalContent = "";

// Register handler BEFORE SendAsync()
session.On(evt =>
{
    switch (evt)
    {
        case AssistantMessageDeltaEvent delta:
            Console.Write(delta.Data.DeltaContent);
            break;
        case AssistantMessageEvent msg:
            finalContent = msg.Data.Content ?? "";
            break;
        case SessionIdleEvent:
            done.SetResult(finalContent);
            break;
    }
});

await session.SendAsync(new MessageOptions { Prompt = "What is 2 + 2?" });
var response = await done.Task;

Console.WriteLine($"\n\nFinal response: {response}");
```

---

# Custom Tools

The SDK allows defining tools that Copilot can invoke during conversations.

## TypeScript (using Zod)

```typescript
import { defineTool } from "@github/copilot-sdk";
import { z } from "zod";

const myTool = defineTool("lookup_issue", {
  description: "Fetch issue details from tracker",
  parameters: z.object({
    id: z.string().describe("Issue identifier"),
  }),
  handler: async ({ id }) => {
    // Return string → auto-wrapped as success
    return JSON.stringify({ status: "found", title: "Bug report" });
  },
});

// Use in session
const session = await client.createSession({
  model: "gpt-5",
  tools: [myTool],
});
```

### TypeScript — Raw JSON Schema (no Zod)

```typescript
import { Tool } from "@github/copilot-sdk";

const myTool: Tool = {
  name: "lookup_issue",
  description: "Fetch issue details from tracker",
  parameters: {
    type: "object",
    properties: {
      id: { type: "string", description: "Issue identifier" },
    },
    required: ["id"],
  },
  handler: async (args, invocation) => {
    // invocation has: sessionId, toolCallId, toolName, arguments
    return { status: "found", title: "Bug report" };
  },
};
```

### TypeScript — ToolResultObject (full control)

```typescript
const myTool = defineTool("process_data", {
  description: "Process some data",
  parameters: z.object({ data: z.string() }),
  handler: async ({ data }) => {
    // Return ToolResultObject for full control
    return {
      textResultForLlm: "Processing complete",
      resultType: "success",  // "success" | "failure" | "rejected" | "denied"
      binaryResultsForLlm: [{
        data: btoa("binary content"),
        mimeType: "image/png",
        type: "image",
        description: "Generated chart",
      }],
      sessionLog: "Processed 100 records",
      toolTelemetry: { recordCount: 100 },
    };
  },
});
```

## Python (using Pydantic)

```python
from pydantic import BaseModel, Field
from copilot import CopilotClient, define_tool

class IssueParams(BaseModel):
    id: str = Field(description="Issue identifier")

# Decorator usage (recommended)
@define_tool(description="Fetch issue details from tracker")
async def lookup_issue(params: IssueParams) -> dict:
    return {"status": "found", "title": "Bug report"}

# Use in session
session = await client.create_session({
    "model": "gpt-5",
    "tools": [lookup_issue],
})
```

### Python — Multiple handler signatures

```python
from copilot import define_tool, ToolInvocation

# No params
@define_tool(description="Get current time")
def get_time() -> str:
    return datetime.now().isoformat()

# Params only
@define_tool(description="Lookup issue")
def lookup(params: IssueParams) -> dict:
    return fetch(params.id)

# Params + invocation context
@define_tool(description="Scoped lookup")
def scoped_lookup(params: IssueParams, inv: ToolInvocation) -> dict:
    print(f"Session: {inv['session_id']}, Call: {inv['tool_call_id']}")
    return fetch(params.id)

# Invocation only (no params)
@define_tool(description="Session info")
def session_info(inv: ToolInvocation) -> str:
    return f"Session: {inv['session_id']}"
```

## Go (using DefineTool with generics)

```go
type IssueParams struct {
    ID string `json:"id" jsonschema:"Issue identifier"`
}

tool := copilot.DefineTool("lookup_issue", "Fetch issue details from tracker",
    func(params IssueParams, inv copilot.ToolInvocation) (any, error) {
        return map[string]string{"status": "found", "title": "Bug report"}, nil
    })

// Use in session
session, _ := client.CreateSession(ctx, &copilot.SessionConfig{
    Model: "gpt-5",
    Tools: []copilot.Tool{tool},
})
```

### Go — Raw Tool struct

```go
tool := copilot.Tool{
    Name:        "lookup_issue",
    Description: "Fetch issue details",
    Parameters:  map[string]any{
        "type": "object",
        "properties": map[string]any{
            "id": map[string]any{"type": "string", "description": "Issue identifier"},
        },
        "required": []string{"id"},
    },
    Handler: func(inv copilot.ToolInvocation) (copilot.ToolResult, error) {
        // inv.SessionID, inv.ToolCallID, inv.ToolName, inv.Arguments
        return copilot.ToolResult{
            TextResultForLLM: `{"status": "found"}`,
            ResultType:       "success",
        }, nil
    },
}
```

## .NET (using AIFunctionFactory)

```csharp
using System.ComponentModel;
using Microsoft.Extensions.AI;

var tool = AIFunctionFactory.Create(
    ([Description("Issue identifier")] string id) =>
        new { Status = "found", Title = "Bug report" },
    "lookup_issue",
    "Fetch issue details from tracker"
);

// Use in session
var session = await client.CreateSessionAsync(new SessionConfig
{
    Model = "gpt-5",
    Tools = new[] { tool }
});
```

---

# Hooks

## TypeScript — All Hooks

```typescript
const session = await client.createSession({
  model: "gpt-5",
  hooks: {
    // Before tool execution — control permissions, modify args
    onPreToolUse: async (input, { sessionId }) => {
      console.log(`Pre-tool: ${input.toolName}`, input.toolArgs);
      return {
        permissionDecision: "allow",      // "allow" | "deny" | "ask"
        permissionDecisionReason: "Approved by policy",
        modifiedArgs: input.toolArgs,     // Optionally modify arguments
        additionalContext: "Extra context for LLM",
        suppressOutput: false,
      };
    },

    // After tool execution — modify result, add context
    onPostToolUse: async (input, { sessionId }) => {
      console.log(`Post-tool: ${input.toolName}`, input.toolResult);
      return {
        modifiedResult: input.toolResult,  // ToolResultObject
        additionalContext: "Tool completed successfully",
        suppressOutput: false,
      };
    },

    // Before prompt is processed — modify prompt, add context
    onUserPromptSubmitted: async (input, { sessionId }) => {
      return {
        modifiedPrompt: input.prompt + "\nPlease be concise.",
        additionalContext: "User prefers brief answers",
        suppressOutput: false,
      };
    },

    // Session lifecycle start — modify config
    onSessionStart: async (input, { sessionId }) => {
      // input.source: "startup" | "resume" | "new"
      // input.initialPrompt?: string
      return {
        additionalContext: "Session started",
        modifiedConfig: { /* session config overrides */ },
      };
    },

    // Session lifecycle end — cleanup
    onSessionEnd: async (input, { sessionId }) => {
      // input.reason: "complete" | "error" | "abort" | "timeout" | "user_exit"
      // input.finalMessage?: string
      // input.error?: string
      return {
        suppressOutput: false,
        cleanupActions: ["save_state", "notify_user"],
        sessionSummary: "Completed 3 tasks",
      };
    },

    // Error handling — retry logic
    onErrorOccurred: async (input, { sessionId }) => {
      // input.error: string
      // input.errorContext: "model_call" | "tool_execution" | "system" | "user_input"
      // input.recoverable: boolean
      return {
        suppressOutput: false,
        errorHandling: input.recoverable ? "retry" : "abort",  // "retry" | "skip" | "abort"
        retryCount: 3,
        userNotification: "An error occurred, retrying...",
      };
    },
  },
});
```

## Python — All Hooks

```python
session = await client.create_session({
    "model": "gpt-5",
    "hooks": {
        "on_pre_tool_use": lambda input, inv: {
            "permissionDecision": "allow",
            "additionalContext": f"Tool {input['toolName']} approved",
        },
        "on_post_tool_use": lambda input, inv: {
            "additionalContext": f"Tool {input['toolName']} completed",
        },
        "on_user_prompt_submitted": lambda input, inv: {
            "modifiedPrompt": input["prompt"] + "\nBe concise.",
        },
        "on_session_start": lambda input, inv: {
            "additionalContext": f"Started via {input['source']}",
        },
        "on_session_end": lambda input, inv: {
            "sessionSummary": f"Ended: {input['reason']}",
        },
        "on_error_occurred": lambda input, inv: {
            "errorHandling": "retry" if input["recoverable"] else "abort",
            "retryCount": 3,
        },
    },
})
```

## Go — Hooks

```go
session, _ := client.CreateSession(ctx, &copilot.SessionConfig{
    Model: "gpt-5",
    Hooks: &copilot.SessionHooks{
        OnPreToolUse: func(input copilot.PreToolUseHookInput, inv copilot.HookInvocation) (*copilot.PreToolUseHookOutput, error) {
            return &copilot.PreToolUseHookOutput{
                PermissionDecision: "allow",
                AdditionalContext:  fmt.Sprintf("Tool %s approved", input.ToolName),
            }, nil
        },
        OnPostToolUse: func(input copilot.PostToolUseHookInput, inv copilot.HookInvocation) (*copilot.PostToolUseHookOutput, error) {
            return &copilot.PostToolUseHookOutput{
                AdditionalContext: fmt.Sprintf("Tool %s completed", input.ToolName),
            }, nil
        },
        OnErrorOccurred: func(input copilot.ErrorOccurredHookInput, inv copilot.HookInvocation) (*copilot.ErrorOccurredHookOutput, error) {
            handling := "abort"
            if input.Recoverable {
                handling = "retry"
            }
            return &copilot.ErrorOccurredHookOutput{
                ErrorHandling: handling,
                RetryCount:    3,
            }, nil
        },
    },
})
```

## C# — Hooks

```csharp
var session = await client.CreateSessionAsync(new SessionConfig
{
    Model = "gpt-5",
    Hooks = new SessionHooks
    {
        OnPreToolUse = async (input, inv) => new PreToolUseHookOutput
        {
            PermissionDecision = "allow",
            AdditionalContext = $"Tool {input.ToolName} approved",
        },
        OnPostToolUse = async (input, inv) => new PostToolUseHookOutput
        {
            AdditionalContext = $"Tool {input.ToolName} completed",
        },
        OnErrorOccurred = async (input, inv) => new ErrorOccurredHookOutput
        {
            ErrorHandling = input.Recoverable ? "retry" : "abort",
            RetryCount = 3,
        },
    },
});
```

---

# BYOK (Bring Your Own Key)

## TypeScript — OpenAI-compatible

```typescript
const session = await client.createSession({
  model: "my-custom-model",
  provider: {
    type: "openai",
    baseUrl: "https://api.openai.com/v1",
    apiKey: process.env.OPENAI_API_KEY,
    wireApi: "completions",  // "completions" | "responses"
  },
});
```

## TypeScript — Azure OpenAI

```typescript
const session = await client.createSession({
  model: "my-deployment-name",
  provider: {
    type: "azure",
    baseUrl: "https://my-resource.openai.azure.com/openai/deployments/my-deployment",
    apiKey: process.env.AZURE_OPENAI_KEY,
    azure: {
      apiVersion: "2024-10-21",
    },
  },
});
```

## TypeScript — Anthropic

```typescript
const session = await client.createSession({
  model: "claude-sonnet-4",
  provider: {
    type: "anthropic",
    baseUrl: "https://api.anthropic.com/v1",
    apiKey: process.env.ANTHROPIC_API_KEY,
  },
});
```

## TypeScript — Bearer Token Auth

```typescript
const session = await client.createSession({
  model: "my-model",
  provider: {
    type: "openai",
    baseUrl: "https://my-service.example.com/v1",
    bearerToken: process.env.BEARER_TOKEN,  // Takes precedence over apiKey
  },
});
```

## TypeScript — Local Ollama (no API key)

```typescript
const session = await client.createSession({
  model: "llama3",
  provider: {
    type: "openai",
    baseUrl: "http://localhost:11434/v1",
    // No apiKey needed for local Ollama
  },
});
```

---

# MCP Servers (Programmatic)

## TypeScript — Local MCP Server

```typescript
const session = await client.createSession({
  model: "gpt-5",
  mcpServers: {
    "my-server": {
      type: "local",
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      tools: ["*"],           // All tools, or specific: ["read_file", "write_file"]
      env: { NODE_ENV: "production" },
      cwd: "/path/to/working/dir",
      timeout: 30000,         // 30 seconds
    },
  },
});
```

## TypeScript — Remote MCP Server (HTTP/SSE)

```typescript
const session = await client.createSession({
  model: "gpt-5",
  mcpServers: {
    "remote-server": {
      type: "http",           // "http" | "sse"
      url: "https://mcp.example.com/api",
      headers: {
        "Authorization": `Bearer ${token}`,
      },
      tools: ["search", "fetch"],
      timeout: 60000,
    },
  },
});
```

---

# Custom Agents (Programmatic)

```typescript
const session = await client.createSession({
  model: "gpt-5",
  customAgents: [
    {
      name: "code-reviewer",
      displayName: "Code Reviewer",
      description: "Reviews code for quality and security issues",
      prompt: "You are an expert code reviewer. Focus on security, performance, and best practices.",
      tools: ["read", "search"],        // null for all tools
      infer: true,                      // Allow auto-selection
      mcpServers: {                     // Agent-specific MCP servers
        "linter": {
          type: "local",
          command: "mcp-linter",
          args: [],
          tools: ["*"],
        },
      },
    },
  ],
});
```

---

# Permission Handler

```typescript
const session = await client.createSession({
  model: "gpt-5",
  onPermissionRequest: async (request, { sessionId }) => {
    // request.kind: "shell" | "write" | "mcp" | "read" | "url"
    console.log(`Permission requested: ${request.kind}`);

    // Auto-approve reads, ask for others
    if (request.kind === "read") {
      return { kind: "approved" };
    }

    // Deny dangerous operations
    if (request.kind === "shell") {
      return {
        kind: "denied-by-rules",
        rules: [{ reason: "Shell execution not allowed" }],
      };
    }

    return { kind: "approved" };
  },
});
```

---

# User Input Handler (ask_user)

```typescript
const session = await client.createSession({
  model: "gpt-5",
  onUserInputRequest: async (request, { sessionId }) => {
    console.log(`Question: ${request.question}`);
    if (request.choices) {
      console.log(`Choices: ${request.choices.join(", ")}`);
    }
    // request.allowFreeform defaults to true

    // In a real app, prompt the user and return their answer
    return {
      answer: "Option A",
      wasFreeform: false,
    };
  },
});
```

---

# Message Attachments

```typescript
await session.send({
  prompt: "Review this code and suggest improvements",
  attachments: [
    // File attachment
    { type: "file", path: "/path/to/main.ts", displayName: "main.ts" },

    // Directory attachment
    { type: "directory", path: "/path/to/src" },

    // Selection attachment (specific code range)
    {
      type: "selection",
      filePath: "/path/to/file.ts",
      displayName: "handleRequest function",
      selection: {
        start: { line: 10, character: 0 },
        end: { line: 25, character: 0 },
      },
      text: "function handleRequest() { ... }",
    },
  ],
  mode: "enqueue",  // "enqueue" (default) | "immediate"
});
```

---

# Infinite Sessions

```typescript
const session = await client.createSession({
  model: "gpt-5",
  infiniteSessions: {
    enabled: true,
    backgroundCompactionThreshold: 0.80,  // Start compaction at 80% context usage
    bufferExhaustionThreshold: 0.95,       // Block at 95% until compaction completes
  },
});

// Workspace path for file persistence
console.log("Workspace:", session.workspacePath);

// Listen for compaction events
session.on((event) => {
  if (event.type === "session.compaction_start") {
    console.log("Compaction started...");
  }
  if (event.type === "session.compaction_complete") {
    console.log(`Compaction: ${event.data.success ? "success" : "failed"}`);
    console.log(`Tokens: ${event.data.preCompactionTokens} → ${event.data.postCompactionTokens}`);
  }
});
```

---

# Session Persistence & Resume

```typescript
// Create and use a session
const session = await client.createSession({ model: "gpt-5" });
const sessionId = session.sessionId;

// ... use session ...
await session.destroy();

// Later: resume the session
const resumed = await client.resumeSession(sessionId, {
  model: "gpt-5",           // Can change model on resume
  tools: [myTool],          // Must re-provide tools (not persisted)
  provider: providerConfig, // Must re-provide BYOK config (not persisted)
  disableResume: false,     // Set true to skip session.resume event
});

// List all sessions
const sessions = await client.listSessions();
for (const meta of sessions) {
  console.log(`${meta.sessionId}: ${meta.summary} (${meta.modifiedTime})`);
}

// Delete a session
await client.deleteSession(sessionId);
```

---

# Model Discovery

```typescript
const models = await client.listModels();
for (const model of models) {
  console.log(`${model.id} (${model.name})`);
  console.log(`  Vision: ${model.capabilities.supports.vision}`);
  console.log(`  Reasoning: ${model.capabilities.supports.reasoningEffort}`);
  if (model.supportedReasoningEfforts) {
    console.log(`  Efforts: ${model.supportedReasoningEfforts.join(", ")}`);
    console.log(`  Default: ${model.defaultReasoningEffort}`);
  }
  if (model.billing) {
    console.log(`  Multiplier: ${model.billing.multiplier}x`);
  }
  console.log(`  Policy: ${model.policy?.state}`);
}
```

---

# Authentication Status

```typescript
const auth = await client.getAuthStatus();
console.log(`Authenticated: ${auth.isAuthenticated}`);
console.log(`Type: ${auth.authType}`);     // "user" | "env" | "gh-cli" | "hmac" | "api-key" | "token"
console.log(`Login: ${auth.login}`);
console.log(`Host: ${auth.host}`);
console.log(`Status: ${auth.statusMessage}`);
```

---

# Session Lifecycle Events (Client-level)

```typescript
// Subscribe to all lifecycle events
client.on((event) => {
  console.log(`Lifecycle: ${event.type} for session ${event.sessionId}`);
  if (event.metadata) {
    console.log(`  Modified: ${event.metadata.modifiedTime}`);
    console.log(`  Summary: ${event.metadata.summary}`);
  }
});

// Subscribe to specific event types
client.on("session.created", (event) => {
  console.log(`New session: ${event.sessionId}`);
});

client.on("session.updated", (event) => {
  console.log(`Session updated: ${event.sessionId}`);
});

// Foreground/background management
const fg = await client.getForegroundSessionId();
console.log(`Foreground: ${fg?.sessionId}`);

await client.setForegroundSessionId(session.sessionId);
```

---

# Skills Configuration

```typescript
const session = await client.createSession({
  model: "gpt-5",
  skillDirectories: ["/path/to/skills", "/another/skills/dir"],
  disabledSkills: ["skill-to-disable"],
});

// Listen for skill invocations
session.on((event) => {
  if (event.type === "skill.invoked") {
    console.log(`Skill loaded: ${event.data.name}`);
    console.log(`  Path: ${event.data.path}`);
    console.log(`  Allowed tools: ${event.data.allowedTools}`);
  }
});
```

---

# System Message Configuration

```typescript
// Append mode (default) — add custom instructions to SDK's system message
const session1 = await client.createSession({
  model: "gpt-5",
  systemMessage: {
    mode: "append",
    content: "Always respond in JSON format. Be concise.",
  },
});

// Replace mode — full control, removes all SDK guardrails
const session2 = await client.createSession({
  model: "gpt-5",
  systemMessage: {
    mode: "replace",
    content: "You are a specialized JSON API. Only respond with valid JSON.",
  },
});
```

---

# Reasoning Effort

```typescript
const session = await client.createSession({
  model: "claude-sonnet-4.5",  // Must support reasoning effort
  reasoningEffort: "high",     // "low" | "medium" | "high" | "xhigh"
});

// Listen for reasoning events
session.on((event) => {
  if (event.type === "assistant.reasoning") {
    console.log("Reasoning:", event.data.content);
  }
  if (event.type === "assistant.reasoning_delta") {
    process.stdout.write(event.data.deltaContent ?? "");
  }
});
```
