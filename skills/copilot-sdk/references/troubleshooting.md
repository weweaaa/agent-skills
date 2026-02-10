# Troubleshooting

## Common Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| Events fire but content is empty | Accessing wrong property | Use `event.data.content`, not `event.content` |
| Handler never fires | Registered after `send()` | Register handler **before** calling `send()` |
| `get_messages()` returns events, not messages | Method returns all `SessionEvent` objects | Filter by `event.type` and extract from `.data` |
| Response in console but not captured | SDK logs internally | Capture via event handler, not console output |
| Python: `event.type` returns enum object | Enum needs `.value` | Use `event.type.value` for the string |
| Go: nil pointer dereference | Content fields are pointers | Check `!= nil` before dereferencing |
| BYOK session resume fails | Provider config not persisted | Must re-provide `provider` in `resumeSession()` |
| Missing `ask_user` tool | No handler provided | Provide `onUserInputRequest` handler in SessionConfig |
| Hooks not firing | Hooks not configured | Provide `hooks` object in SessionConfig |
| MCP server tools not available | Tools not listed | Set `tools: ["*"]` or list specific tool names |
| `sendAndWait()` times out | Default 60s timeout | Pass longer timeout as second argument |
| Session not persisting | Infinite sessions disabled | Ensure `infiniteSessions.enabled` is `true` (default) |
| Auth fails despite valid token | Wrong priority order | Check auth priority: explicit token > env vars > OAuth > gh CLI |
| Skill not loading | Wrong directory | Check `skillDirectories` paths are absolute and correct |
| Reasoning events missing | Model doesn't support it | Use model with `reasoningEffort` capability; check `listModels()` |
| Compaction not happening | Thresholds too high | Lower `backgroundCompactionThreshold` (default 0.80) |

---

## Event Handler Timing

**Correct pattern:**
```
1. Create session
2. Register event handler(s)
3. Call send()
4. Wait for session.idle or use sendAndWait()
```

**Incorrect pattern (misses events):**
```
1. Create session
2. Call send()
3. Register event handler  ← Too late, events already fired
```

---

## Debugging Events

To inspect all events during development:

**TypeScript:**
```typescript
session.on((event) => {
  console.log(JSON.stringify(event, null, 2));
});
```

**Python:**
```python
def debug_handler(event):
    print(f"Event: {event.type.value}")
    print(f"Data: {event.data}")

session.on(debug_handler)
```

**Go:**
```go
session.On(func(event copilot.SessionEvent) {
    fmt.Printf("Event: %s\n", event.Type)
    fmt.Printf("Data: %+v\n", event.Data)
})
```

**C#:**
```csharp
session.On(evt => {
    Console.WriteLine($"Event: {evt.GetType().Name}");
    Console.WriteLine($"Data: {System.Text.Json.JsonSerializer.Serialize(evt)}");
});
```

---

## Debugging Hooks

Hooks emit `hook.start` and `hook.end` events you can monitor:

```typescript
session.on((event) => {
  if (event.type === "hook.start") {
    console.log(`Hook started: ${event.data.hookType} (${event.data.hookInvocationId})`);
    console.log(`Input:`, event.data.input);
  }
  if (event.type === "hook.end") {
    console.log(`Hook ended: ${event.data.hookType}`);
    console.log(`Success: ${event.data.success}`);
    if (event.data.error) console.error(`Error: ${event.data.error}`);
    console.log(`Output:`, event.data.output);
  }
});
```

Common hook issues:

| Problem | Cause | Solution |
|---------|-------|----------|
| Hook handler errors silently | SDK catches exceptions | Add try/catch in hook handler, log errors |
| `onPreToolUse` not blocking tool | Missing `permissionDecision` | Return `{ permissionDecision: "deny" }` |
| Modified args not applied | Wrong return format | Return `{ modifiedArgs: newArgs }` |
| `onSessionStart` config not applied | Wrong key format | Return `{ modifiedConfig: { key: value } }` |

---

## Debugging MCP Servers

MCP connection issues:

| Problem | Cause | Solution |
|---------|-------|----------|
| MCP server not starting | Command not found | Check `command` path is correct and executable |
| MCP tools not visible | Tools not listed | Set `tools: ["*"]` or list specific names |
| MCP timeout | Operation too slow | Increase `timeout` value (milliseconds) |
| Environment variables not passed | Wrong config | Use `env` field in MCP server config |
| Remote MCP auth fails | Missing headers | Add `headers` with proper auth |

Monitor MCP tool calls via events:

```typescript
session.on((event) => {
  if (event.type === "tool.execution_start" && event.data.mcpServerName) {
    console.log(`MCP tool: ${event.data.mcpServerName}/${event.data.mcpToolName}`);
  }
  if (event.type === "tool.execution_complete") {
    console.log(`Tool ${event.data.toolCallId}: ${event.data.success ? "OK" : "FAILED"}`);
    if (event.data.error) console.error(event.data.error);
  }
});
```

---

## Debugging BYOK Providers

| Problem | Cause | Solution |
|---------|-------|----------|
| 401 Unauthorized | Wrong API key | Verify `apiKey` or `bearerToken` |
| Model not found | Wrong model name | Check provider's model naming convention |
| Azure 404 | Wrong deployment URL | Include deployment name in `baseUrl` |
| Azure version error | API version mismatch | Set `azure.apiVersion` (default: `"2024-10-21"`) |
| Both apiKey and bearerToken | Conflicting auth | `bearerToken` takes precedence; use only one |
| Resume fails | Provider not re-supplied | Provider config is NOT persisted; re-provide on resume |
| wireApi mismatch | Wrong format | Use `"completions"` (default) or `"responses"` |

---

## Debugging Connection Issues

```typescript
// Check connection state
const state = await client.getState();
console.log(`State: ${state}`);  // "disconnected" | "connecting" | "connected" | "error"

// Ping the server
const pong = await client.ping("test");
console.log(`Pong: ${pong.message}, Protocol: ${pong.protocolVersion}`);

// Check authentication
const auth = await client.getAuthStatus();
console.log(`Authenticated: ${auth.isAuthenticated}`);
console.log(`Auth type: ${auth.authType}`);
console.log(`Login: ${auth.login}`);
console.log(`Message: ${auth.statusMessage}`);
```

---

## Debugging Token Usage

Monitor token consumption for infinite session management:

```typescript
session.on((event) => {
  if (event.type === "session.usage_info") {
    const pct = (event.data.currentTokens / event.data.tokenLimit * 100).toFixed(1);
    console.log(`Tokens: ${event.data.currentTokens}/${event.data.tokenLimit} (${pct}%)`);
    console.log(`Messages: ${event.data.messagesLength}`);
  }
  if (event.type === "assistant.usage") {
    console.log(`Model: ${event.data.model}`);
    console.log(`Input: ${event.data.inputTokens}, Output: ${event.data.outputTokens}`);
    console.log(`Cache read: ${event.data.cacheReadTokens}, write: ${event.data.cacheWriteTokens}`);
    console.log(`Cost: ${event.data.cost}, Duration: ${event.data.duration}ms`);
  }
});
```

---

## Debugging Subagents & Skills

```typescript
session.on((event) => {
  if (event.type === "skill.invoked") {
    console.log(`Skill: ${event.data.name} from ${event.data.path}`);
    console.log(`Allowed tools: ${event.data.allowedTools}`);
  }
  if (event.type === "subagent.selected") {
    console.log(`Subagent selected: ${event.data.agentDisplayName}`);
    console.log(`Tools: ${event.data.tools}`);
  }
  if (event.type === "subagent.started") {
    console.log(`Subagent started: ${event.data.agentName} (${event.data.agentDescription})`);
  }
  if (event.type === "subagent.completed") {
    console.log(`Subagent completed: ${event.data.agentName}`);
  }
  if (event.type === "subagent.failed") {
    console.error(`Subagent failed: ${event.data.agentName} - ${event.data.error}`);
  }
});
```

---

## CLI Log Level

Set verbose logging to debug CLI-level issues:

```typescript
const client = new CopilotClient({
  logLevel: "debug",  // "none" | "error" | "warning" | "info" | "debug" | "all"
});
```

---

## Ephemeral vs. Persisted Events

When debugging with `getMessages()`, remember that **ephemeral events** are NOT included:

- **NOT persisted:** `session.idle`, `assistant.message_delta`, `assistant.reasoning_delta`, `assistant.usage`, `session.usage_info`, `assistant.intent`, `pending_messages.modified`, `session.snapshot_rewind`, `session.shutdown`, `tool.execution_partial_result`, `tool.execution_progress`
- **Persisted:** `assistant.message`, `assistant.reasoning`, `user.message`, `tool.execution_start`, `tool.execution_complete`, `session.start`, `session.error`, `skill.invoked`, `hook.start`, `hook.end`, etc.

If events appear during real-time streaming but not in `getMessages()`, they are ephemeral by design.
