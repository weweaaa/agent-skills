# Event System

## Event Lifecycle

Events fire in this typical sequence for a single request:

```
1. user.message              → User prompt recorded
2. assistant.turn_start      → Model begins processing (turnId)
3. session.usage_info        → Token/usage metadata (ephemeral)
4. assistant.intent          → Detected intent (ephemeral)
5. assistant.message_delta   → Streaming chunks (if enabled, repeats, ephemeral)
6. assistant.message         → Final complete response
7. assistant.reasoning       → Chain-of-thought (if available)
8. assistant.usage           → Model usage metrics (ephemeral)
9. assistant.turn_end        → Model finished processing (turnId)
10. session.idle             → Session ready for next prompt (ephemeral)
```

When tools are invoked, additional events appear:
```
   tool.execution_start      → Tool invocation began
   tool.execution_progress   → Progress updates (ephemeral)
   tool.execution_partial_result → Partial results (ephemeral)
   tool.execution_complete   → Tool execution completed
```

When hooks are active:
```
   hook.start                → Hook invocation began
   hook.end                  → Hook invocation completed
```

When skills/subagents are used:
```
   skill.invoked             → Skill was loaded
   subagent.selected         → Subagent was selected
   subagent.started          → Subagent began execution
   subagent.completed        → Subagent finished
   subagent.failed           → Subagent failed
```

---

## Complete Event Types Reference

All 31+ event types from the generated session events:

### Session Events

| Event Type | Purpose | Key Data Fields | Ephemeral |
|------------|---------|-----------------|-----------|
| `session.start` | Session initialized | `sessionId`, `version`, `producer`, `copilotVersion`, `startTime`, `selectedModel?`, `context?` | No |
| `session.resume` | Session resumed | `resumeTime`, `eventCount`, `context?` | No |
| `session.error` | Error occurred | `errorType`, `message`, `stack?`, `statusCode?`, `providerCallId?` | No |
| `session.idle` | Ready for next prompt | — | **Yes** |
| `session.info` | Informational message | `infoType`, `message` | No |
| `session.model_change` | Model switched | `previousModel?`, `newModel` | No |
| `session.handoff` | Session handed off | `handoffTime`, `sourceType`, `repository?`, `context?`, `summary?`, `remoteSessionId?` | No |
| `session.truncation` | Messages truncated | `tokenLimit`, `preTruncationTokensInMessages`, `postTruncationTokensInMessages`, `messagesRemovedDuringTruncation`, `performedBy` | No |
| `session.snapshot_rewind` | Snapshot rewound | `upToEventId`, `eventsRemoved` | **Yes** |
| `session.shutdown` | Session shutting down | `shutdownType`, `errorReason?`, `totalPremiumRequests`, `totalApiDurationMs`, `sessionStartTime`, `codeChanges`, `modelMetrics`, `currentModel?` | **Yes** |
| `session.usage_info` | Token usage data | `tokenLimit`, `currentTokens`, `messagesLength` | **Yes** |
| `session.compaction_start` | Compaction began | — | No |
| `session.compaction_complete` | Compaction finished | `success`, `error?`, `preCompactionTokens?`, `postCompactionTokens?`, `summaryContent?`, `checkpointNumber?`, `checkpointPath?`, `compactionTokensUsed?`, `requestId?` | No |

### User Events

| Event Type | Purpose | Key Data Fields | Ephemeral |
|------------|---------|-----------------|-----------|
| `user.message` | User input recorded | `content`, `transformedContent?`, `attachments?`, `source?` | No |
| `pending_messages.modified` | Pending queue changed | — | **Yes** |

### Assistant Events

| Event Type | Purpose | Key Data Fields | Ephemeral |
|------------|---------|-----------------|-----------|
| `assistant.turn_start` | Processing began | `turnId` | No |
| `assistant.intent` | Detected intent | `intent` | **Yes** |
| `assistant.reasoning` | Chain-of-thought | `reasoningId`, `content` | No |
| `assistant.reasoning_delta` | Reasoning chunk | `reasoningId`, `deltaContent` | **Yes** |
| `assistant.message` | Complete response | `messageId`, `content`, `toolRequests?`, `reasoningOpaque?`, `reasoningText?`, `encryptedContent?`, `parentToolCallId?` | No |
| `assistant.message_delta` | Streaming chunk | `messageId`, `deltaContent`, `totalResponseSizeBytes?`, `parentToolCallId?` | **Yes** |
| `assistant.turn_end` | Processing finished | `turnId` | No |
| `assistant.usage` | Model usage metrics | `model`, `inputTokens?`, `outputTokens?`, `cacheReadTokens?`, `cacheWriteTokens?`, `cost?`, `duration?`, `initiator?`, `apiCallId?`, `providerCallId?`, `parentToolCallId?`, `quotaSnapshots?` | **Yes** |

### Tool Events

| Event Type | Purpose | Key Data Fields | Ephemeral |
|------------|---------|-----------------|-----------|
| `abort` | Request aborted | `reason` | No |
| `tool.user_requested` | User requested tool | `toolCallId`, `toolName`, `arguments?` | No |
| `tool.execution_start` | Tool invocation began | `toolCallId`, `toolName`, `arguments?`, `mcpServerName?`, `mcpToolName?`, `parentToolCallId?` | No |
| `tool.execution_partial_result` | Partial tool output | `toolCallId`, `partialOutput` | **Yes** |
| `tool.execution_progress` | Progress message | `toolCallId`, `progressMessage` | **Yes** |
| `tool.execution_complete` | Tool completed | `toolCallId`, `success`, `isUserRequested?`, `result?`, `error?`, `toolTelemetry?`, `parentToolCallId?` | No |

### Skill & Subagent Events

| Event Type | Purpose | Key Data Fields | Ephemeral |
|------------|---------|-----------------|-----------|
| `skill.invoked` | Skill loaded | `name`, `path`, `content`, `allowedTools?` | No |
| `subagent.selected` | Subagent selected | `agentName`, `agentDisplayName`, `tools` | No |
| `subagent.started` | Subagent began | `toolCallId`, `agentName`, `agentDisplayName`, `agentDescription` | No |
| `subagent.completed` | Subagent finished | `toolCallId`, `agentName` | No |
| `subagent.failed` | Subagent failed | `toolCallId`, `agentName`, `error` | No |

### Hook Events

| Event Type | Purpose | Key Data Fields | Ephemeral |
|------------|---------|-----------------|-----------|
| `hook.start` | Hook invocation began | `hookInvocationId`, `hookType`, `input?` | No |
| `hook.end` | Hook invocation finished | `hookInvocationId`, `hookType`, `output?`, `success`, `error?` | No |

### System Events

| Event Type | Purpose | Key Data Fields | Ephemeral |
|------------|---------|-----------------|-----------|
| `system.message` | System message | `content`, `role`, `name?`, `metadata?` | No |

---

## Ephemeral Events

Events marked as **ephemeral** are NOT persisted in session history. They are only available during real-time streaming. Important implications:

- `session.idle` is ephemeral — you cannot detect idle state from `getMessages()`
- `assistant.message_delta` is ephemeral — only the final `assistant.message` is persisted
- `assistant.usage` is ephemeral — usage metrics not available on resume
- `session.usage_info` is ephemeral — token counts not persisted

---

## SessionEvent Structure

Events are wrapped in a `SessionEvent` object (exact field names vary by language):

```
SessionEvent {
  type: string | enum       // Event type identifier
  data: {                   // Event-specific payload
    content?: string        // Full text (final events)
    deltaContent?: string   // Incremental text (delta events)
    messageId?: string      // Message identifier
    turnId?: string         // Turn identifier
    toolCallId?: string     // Tool call identifier
    // Other fields vary by event type
  }
  id?: string               // Event identifier
  timestamp?: string        // When event occurred
}
```

**Important access patterns:**
- TypeScript: `event.type` (string), `event.data.content`, `event.data.deltaContent`
- Python: `event.type.value` (enum requires `.value`), `event.data.content`, `event.data.delta_content`
- Go: `event.Type` (string), `*event.Data.Content`, `*event.Data.DeltaContent` (pointers — check `!= nil`)
- .NET: Pattern match on event class, then `evt.Data.Content`, `evt.Data.DeltaContent`

---

## Streaming Behavior

The `streaming` configuration option controls **when** content arrives, not **which** events fire:

| Setting | Behavior |
|---------|----------|
| `streaming: false` (default) | Content arrives all at once in `assistant.message` |
| `streaming: true` | Content arrives incrementally via `assistant.message_delta` events |

**Important:** Final events (`assistant.message`, `assistant.reasoning`) **always fire** regardless of streaming setting. They contain the complete accumulated content.

To build a response progressively:
1. Accumulate `deltaContent` from each `assistant.message_delta` event
2. Or wait for `assistant.message` which contains the complete text

---

## Session Lifecycle Events (Client-level)

These events are separate from session events — they notify about session state changes:

| Event Type | Description | Metadata |
|------------|-------------|----------|
| `session.created` | New session was created | `startTime`, `modifiedTime`, `summary?` |
| `session.deleted` | Session was deleted | — |
| `session.updated` | Session was updated (new messages) | `startTime`, `modifiedTime`, `summary?` |
| `session.foreground` | Session became TUI foreground | `startTime`, `modifiedTime`, `summary?` |
| `session.background` | Session no longer TUI foreground | `startTime`, `modifiedTime`, `summary?` |

Subscribe via `client.on(handler)` or `client.on(eventType, handler)`.

---

## Language-Specific Conventions

| Concept | TypeScript | Python | Go | .NET |
|---------|------------|--------|----|----|
| Client class | `CopilotClient` | `CopilotClient` | `Client` | `CopilotClient` |
| Create session | `createSession()` | `create_session()` | `CreateSession()` | `CreateSessionAsync()` |
| Send message | `send()` / `sendAndWait()` | `send()` / `send_and_wait()` | `Send()` | `SendAsync()` / `SendAndWaitAsync()` |
| Session config | `{ model, streaming }` | `{"model", "streaming"}` | `&SessionConfig{}` | `new SessionConfig{}` |
| Event handler | `session.on(fn)` | `session.on(fn)` | `session.On(fn)` | `session.On(fn)` |
| Typed event handler | `session.on("assistant.message", fn)` | — | — | — |
| Delta content | `deltaContent` | `delta_content` | `DeltaContent` | `DeltaContent` |
| System message | `systemMessage` | `system_message` | `SystemMessage` | `SystemMessage` |
| CLI path option | `cliPath` | `cli_path` | `CLIPath` | `CliPath` |
| Event type access | `event.type` (string) | `event.type.value` (enum) | `event.Type` (string) | pattern match on class |
| Content access | `event.data.content` | `event.data.content` | `*event.Data.Content` (ptr) | `evt.Data.Content` |
| Nil/null safety | automatic | automatic | check `!= nil` | nullable types |
