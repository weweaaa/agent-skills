# Practical Examples — Real-World Applications

This document contains complete, production-ready examples demonstrating how to build real-world applications with the GitHub Copilot SDK. Each example includes a full application scenario, comprehensive hook usage, and best practices.

---

## Table of Contents

1. [Session Lifecycle Management](#session-lifecycle-management)
2. [Error Handling Strategy](#error-handling-strategy)
3. [Compliance Audit System](#compliance-audit-system)

---

## Session Lifecycle Management

**Scenario:** Project Development Assistant

A smart assistant that automatically detects project type, loads user preferences, handles session resume, tracks metrics, and cleans up resources at session end.

### Features

- Auto-detect project type (language, package manager) at session start
- Load user preferences (language, code style, verbosity)
- Handle session resume with previous context restoration
- Track session duration, prompt count, tool call count
- Clean up temporary resources (temp files, etc.)
- Output comprehensive session summary report

### Hooks Used

- `onSessionStart` - Inject project context, initialize tracking
- `onSessionEnd` - Generate summary, cleanup resources, save state
- `onUserPromptSubmitted` - Count prompts
- `onPreToolUse` - Count tool calls

### TypeScript Implementation

```typescript
import { CopilotClient } from "@github/copilot-sdk";
import fs from "node:fs/promises";
import path from "node:path";

// Helper function: Detect project type
async function detectProjectType(cwd: string) {
  const checks = [
    { file: "package.json", type: "Node.js", language: "JavaScript/TypeScript", packageManager: "npm" },
    { file: "pyproject.toml", type: "Python", language: "Python", packageManager: "pip" },
    { file: "go.mod", type: "Go", language: "Go", packageManager: "go modules" },
    { file: "*.csproj", type: ".NET", language: "C#", packageManager: "NuGet" },
  ];

  for (const check of checks) {
    try {
      await fs.access(path.join(cwd, check.file));
      return { type: check.type, language: check.language, packageManager: check.packageManager };
    } catch {
      continue;
    }
  }

  return { type: "Unknown", language: "Not detected", packageManager: "Not detected" };
}

// Helper function: Load previous session state
async function loadSessionState(sessionId: string) {
  // In real app, load from database or filesystem
  return {
    lastTopic: "Refactoring user authentication module",
    openFiles: ["src/auth/login.ts", "src/auth/middleware.ts"],
  };
}

// Helper function: Save session state
async function saveSessionState(sessionId: string, state: any) {
  console.log(`[State Saved] Session ${sessionId}:`, JSON.stringify(state, null, 2));
}

// Helper function: Load user preferences
async function loadUserPreferences() {
  return {
    language: "Traditional Chinese",
    codeStyle: "ESLint + Prettier",
    verbosity: "concise",
  };
}

// Helper function: Record metrics
async function recordMetrics(metrics: any) {
  console.log("[Metrics Recorded]", JSON.stringify(metrics, null, 2));
}

// Shared state for tracking
const sessionStartTimes = new Map<string, number>();
const sessionResources = new Map<string, { tempFiles: string[] }>();
const sessionData: Record<string, { prompts: number; tools: number; startTime: number }> = {};

// Main application
const client = new CopilotClient();

const session = await client.createSession({
  model: "gpt-4.1",
  streaming: true,

  hooks: {
    // Session Start — Initialize session, inject context
    onSessionStart: async (input, invocation) => {
      const sid = invocation.sessionId;
      console.log(`✦ Session ${sid} started (source: ${input.source})`);

      // Initialize tracking
      sessionStartTimes.set(sid, input.timestamp);
      sessionResources.set(sid, { tempFiles: [] });
      sessionData[sid] = { prompts: 0, tools: 0, startTime: input.timestamp };

      // Handle session resume
      if (input.source === "resume") {
        const previousState = await loadSessionState(sid);
        return {
          additionalContext: `
Session resumed. Previous context:
- Last topic: ${previousState.lastTopic}
- Open files: ${previousState.openFiles.join(", ")}
Please continue from where we left off.
          `.trim(),
        };
      }

      // New session — detect project and load preferences
      const projectInfo = await detectProjectType(input.cwd);
      const preferences = await loadUserPreferences();

      const contextParts = [
        `This is a ${projectInfo.type} project.`,
        `Main language: ${projectInfo.language}`,
        `Package manager: ${projectInfo.packageManager}`,
        `User preferred language: ${preferences.language}`,
        `Code style: ${preferences.codeStyle}`,
      ];

      if (preferences.verbosity === "concise") {
        contextParts.push("Keep responses brief and to the point.");
      }

      return {
        additionalContext: contextParts.join("\n"),
      };
    },

    // User Prompt Submitted — Count prompts
    onUserPromptSubmitted: async (_input, invocation) => {
      sessionData[invocation.sessionId].prompts++;
      return null;
    },

    // Pre-Tool Use — Count tool calls
    onPreToolUse: async (_input, invocation) => {
      sessionData[invocation.sessionId].tools++;
      return { permissionDecision: "allow" };
    },

    // Session End — Generate summary and cleanup
    onSessionEnd: async (input, invocation) => {
      const sid = invocation.sessionId;

      // Track duration metrics
      const startTime = sessionStartTimes.get(sid);
      const duration = startTime ? input.timestamp - startTime : 0;

      await recordMetrics({
        sessionId: sid,
        duration,
        endReason: input.reason,
      });

      sessionStartTimes.delete(sid);

      // Clean up temp resources
      const resources = sessionResources.get(sid);
      if (resources) {
        for (const file of resources.tempFiles) {
          await fs.unlink(file).catch(() => {});
        }
        sessionResources.delete(sid);
        console.log(`[Cleanup] Removed temp resources for session ${sid}`);
      }

      // Save state for future resume (if not error)
      if (input.reason !== "error") {
        await saveSessionState(sid, {
          endTime: input.timestamp,
          cwd: input.cwd,
          reason: input.reason,
        });
      }

      // Output session summary
      const data = sessionData[sid];
      if (data) {
        const durationSec = (input.timestamp - data.startTime) / 1000;
        const reasonLabel = {
          complete: "Complete",
          error: "Error",
          abort: "Aborted",
          timeout: "Timeout",
          user_exit: "User Exit",
        }[input.reason] || input.reason;

        console.log(`
╔══════════════════════════════════════╗
║        Session Summary Report        ║
╠══════════════════════════════════════╣
║  ID: ${sid}
║  Duration: ${durationSec.toFixed(1)}s
║  Prompts: ${data.prompts}
║  Tool calls: ${data.tools}
║  End reason: ${reasonLabel}
╚══════════════════════════════════════╝
        `.trim());

        delete sessionData[sid];
      }

      return null;
    },
  },
});

// Listen for streaming responses
session.on("assistant.message_delta", (event) => {
  process.stdout.write(event.data.deltaContent);
});

session.on("session.idle", () => {
  console.log(); // New line
});

// Send test prompt
await session.sendAndWait({
  prompt: "Please briefly introduce the project structure and suggest three areas for optimization.",
});

// End session
await client.stop();
```

### Key Takeaways

1. **Keep onSessionStart fast** - Users are waiting for the session to start
2. **Handle all end reasons** - Don't assume sessions end cleanly
3. **Clean up resources** - Use onSessionEnd to free allocated resources
4. **Store minimal state** - Keep tracking data lightweight
5. **Make cleanup idempotent** - onSessionEnd might not be called if process crashes

---

## Error Handling Strategy

**Scenario:** Smart Code Review Assistant

An AI assistant for code reviews that may encounter various errors: model call failures (rate limits), tool execution exceptions (lint tool crashes), system errors, etc.

### Features

- Structured error logging (error source, recoverability, context)
- Error reporting to monitoring platform (simulate Sentry integration)
- Track error patterns and warn on frequent occurrences
- Critical error instant alerts
- Automatic handling strategy based on error type:
  - Recoverable tool errors → Suppress output, provide recovery suggestions
  - Model call rate limits → Auto-retry 3 times
  - Other errors → Display friendly user messages

### Error Sources

- `model_call` - Model communication issues
- `tool_execution` - Tool execution failures
- `system` - System errors
- `user_input` - Input validation errors

### Hooks Used

- `onPreToolUse` - Track last tool used
- `onUserPromptSubmitted` - Track last prompt
- `onErrorOccurred` - Core error handling
- `onSessionEnd` - Output error statistics summary

### TypeScript Implementation

```typescript
import { CopilotClient } from "@github/copilot-sdk";

// Simulate external services
async function sendAlert(alert: any) {
  console.log("\n🚨 [Alert]", JSON.stringify(alert, null, 2));
}

function captureException(error: Error, context: any) {
  console.log("\n📡 [Monitoring]", {
    message: error.message,
    tags: context.tags,
    extra: context.extra,
  });
}

// Friendly error messages
const ERROR_MESSAGES: Record<string, string> = {
  model_call: "There was an issue communicating with the AI model. Please try again.",
  tool_execution: "Tool execution failed. Please check your inputs and try again.",
  system: "A system error occurred. Please try again later.",
  user_input: "There was an issue with your input. Please check and try again.",
};

// Critical error sources
const CRITICAL_CONTEXTS = ["system", "model_call"];

// Error pattern tracking
const errorStats = new Map<string, { count: number; lastOccurred: number; sessions: string[] }>();

// Session context tracking (combined with other hooks for error diagnosis)
const sessionContext = new Map<string, { lastTool?: string; lastPrompt?: string }>();

// Main application
const client = new CopilotClient();

const session = await client.createSession({
  model: "gpt-4.1",
  streaming: true,

  hooks: {
    // Pre-Tool Use — Track last tool used
    onPreToolUse: async (input, invocation) => {
      const ctx = sessionContext.get(invocation.sessionId) || {};
      ctx.lastTool = input.toolName;
      sessionContext.set(invocation.sessionId, ctx);
      return { permissionDecision: "allow" };
    },

    // User Prompt Submitted — Track last prompt
    onUserPromptSubmitted: async (input, invocation) => {
      const ctx = sessionContext.get(invocation.sessionId) || {};
      ctx.lastPrompt = input.prompt.substring(0, 100);
      sessionContext.set(invocation.sessionId, ctx);
      return null;
    },

    // Error Occurred — Core error handling
    onErrorOccurred: async (input, invocation) => {
      const sid = invocation.sessionId;

      // 1. Structured error logging
      const ctx = sessionContext.get(sid);
      console.error(`\n✖ Error in session ${sid}:`);
      console.error(`  Message: ${input.error}`);
      console.error(`  Context: ${input.errorContext}`);
      console.error(`  Recoverable: ${input.recoverable ? "Yes" : "No"}`);
      if (ctx?.lastTool) {
        console.error(`  Last tool: ${ctx.lastTool}`);
      }
      if (ctx?.lastPrompt) {
        console.error(`  Last prompt: ${ctx.lastPrompt}...`);
      }

      // 2. Report to monitoring platform
      captureException(new Error(input.error), {
        tags: {
          sessionId: sid,
          errorContext: input.errorContext,
        },
        extra: {
          error: input.error,
          recoverable: input.recoverable,
          cwd: input.cwd,
          lastTool: ctx?.lastTool,
          lastPrompt: ctx?.lastPrompt,
        },
      });

      // 3. Track error patterns
      const patternKey = `${input.errorContext}:${input.error.substring(0, 50)}`;
      const existing = errorStats.get(patternKey) || {
        count: 0,
        lastOccurred: 0,
        sessions: [],
      };

      existing.count++;
      existing.lastOccurred = input.timestamp;
      existing.sessions.push(sid);
      errorStats.set(patternKey, existing);

      // Warn on recurring errors (5+ occurrences)
      if (existing.count >= 5) {
        console.warn(`\n⚠ Recurring error detected: ${patternKey} (${existing.count} times)`);
      }

      // 4. Critical error instant alerts
      if (CRITICAL_CONTEXTS.includes(input.errorContext) && !input.recoverable) {
        await sendAlert({
          level: "critical",
          message: `Critical error in session ${sid}`,
          error: input.error,
          context: input.errorContext,
          timestamp: new Date(input.timestamp).toISOString(),
        });
      }

      // 5. Error handling strategy

      // Strategy A: Recoverable tool error → Suppress, provide recovery suggestions
      if (input.errorContext === "tool_execution" && input.recoverable) {
        console.log(`  ↳ Strategy: Suppress recoverable tool error`);
        return {
          suppressOutput: true,
          userNotification: `
Tool execution failed. Here are recovery suggestions:
- Check if required dependencies are installed
- Verify file paths are correct
- Try a simpler approach
          `.trim(),
        };
      }

      // Strategy B: Model call rate limit → Auto-retry
      if (input.errorContext === "model_call" && input.error.includes("rate")) {
        console.log(`  ↳ Strategy: Rate limit, auto-retry 3 times`);
        return {
          errorHandling: "retry",
          retryCount: 3,
          userNotification: "Rate limit hit. Retrying...",
        };
      }

      // Strategy C: Other errors → Display friendly message
      const friendlyMessage = ERROR_MESSAGES[input.errorContext];
      if (friendlyMessage) {
        return {
          userNotification: friendlyMessage,
        };
      }

      // Strategy D: Unknown error → Use default handling
      return null;
    },

    // Session End — Output error statistics summary
    onSessionEnd: async (input, invocation) => {
      const sid = invocation.sessionId;

      // Cleanup context
      sessionContext.delete(sid);

      // Output error statistics for this session
      if (errorStats.size > 0) {
        console.log("\n╔══════════════════════════════════════╗");
        console.log("║        Error Statistics Summary       ║");
        console.log("╠══════════════════════════════════════╣");
        for (const [key, stats] of errorStats.entries()) {
          if (stats.sessions.includes(sid)) {
            console.log(`║  ${key}`);
            console.log(`║    Count: ${stats.count}`);
            console.log(`║    Last: ${new Date(stats.lastOccurred).toLocaleString()}`);
          }
        }
        console.log("╚══════════════════════════════════════╝");
      }

      const reasonLabel = {
        complete: "Complete",
        error: "Error",
        abort: "Aborted",
        timeout: "Timeout",
        user_exit: "User Exit",
      }[input.reason] || input.reason;

      console.log(`\n✦ Session ${sid} ended (reason: ${reasonLabel})`);
      return null;
    },
  },
});

// Listen for streaming responses
session.on("assistant.message_delta", (event) => {
  process.stdout.write(event.data.deltaContent);
});

session.on("session.idle", () => {
  console.log(); // New line
});

// Send test prompt
await session.sendAndWait({
  prompt: "Please review the code quality in the current project and identify potential performance issues and security vulnerabilities.",
});

// End session
await client.stop();
```

### Key Takeaways

1. **Always log errors** - Even if suppressed from users, keep logs for debugging
2. **Categorize errors** - Use errorContext to handle different errors appropriately
3. **Don't swallow critical errors** - Only suppress non-critical recoverable errors
4. **Keep hooks fast** - Error handling shouldn't slow down recovery
5. **Provide helpful context** - Additional context can help the model recover
6. **Monitor error patterns** - Track recurring errors to identify systemic issues

---

## Compliance Audit System

**Scenario:** Compliance-Oriented Development Assistant

A fintech company's internal development assistant where all AI and tool interactions must go through strict permission control, operation logging, result transformation, and compliance auditing.

### Features

- **Session Initialization** - Inject user preferences, project context, and compliance reminders
- **Prompt Interception** - Log every user prompt to audit log
- **Tool Permission Control** - Block dangerous tools (shell/bash/exec), validate by user role
- **Result Security Filtering** - Auto-redact API keys, passwords, emails from tool outputs
- **Smart Error Handling** - Rate limit retry, skip recoverable errors, abort unrecoverable errors
- **Full Audit Report** - Generate detailed operation log and statistics at session end

### All Six Hooks Used

| Hook | Purpose |
|------|---------|
| `onSessionStart` | Initialize, inject context, load preferences |
| `onUserPromptSubmitted` | Log prompt to audit trail |
| `onPreToolUse` | Permission control, block dangerous tools |
| `onPostToolUse` | Filter sensitive info, add context |
| `onErrorOccurred` | Error classification, auto-retry |
| `onSessionEnd` | Generate audit report, cleanup |

### TypeScript Implementation

```typescript
import { CopilotClient } from "@github/copilot-sdk";

// Helper functions
async function loadUserPreferences() {
  return {
    language: "Traditional Chinese",
    codeStyle: "ESLint + Prettier",
    verbosity: "concise",
    role: "Backend Engineer",
  };
}

async function detectProjectType(cwd: string) {
  return { type: "Node.js", language: "TypeScript", packageManager: "npm" };
}

async function checkToolPermission(role: string, toolName: string) {
  // Admins can execute all tools; engineers cannot use shell
  const restrictedForEngineers = ["shell", "bash", "exec", "run_command"];
  if (role !== "Admin" && restrictedForEngineers.includes(toolName)) {
    return { allowed: false, reason: "Only admins can execute shell commands" };
  }
  return { allowed: true };
}

function redactSensitiveData(text: any): any {
  if (typeof text !== "string") return text;
  // Redact API keys, passwords, etc.
  return text
    .replace(/(?:api[_-]?key|token|secret|password)\s*[:=]\s*["']?[^\s"']+/gi, "[REDACTED]")
    .replace(/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}\b/gi, "[EMAIL_REDACTED]");
}

// Blocked tools list
const BLOCKED_TOOLS = ["shell", "bash", "exec"];

// Audit logs and session state
const auditLogs = new Map<string, Array<any>>();
const sessionStartTimes = new Map<string, number>();
const sessionRoles = new Map<string, string>();

// Main application
const client = new CopilotClient();

const session = await client.createSession({
  model: "gpt-4.1",
  streaming: true,

  hooks: {
    // Session Start — Initialize, inject context
    onSessionStart: async (input, invocation) => {
      const sid = invocation.sessionId;
      console.log(`\n✦ Session ${sid} started (source: ${input.source})`);

      // Initialize audit log
      auditLogs.set(sid, []);
      sessionStartTimes.set(sid, input.timestamp);

      // Load user preferences
      const userPrefs = await loadUserPreferences();
      sessionRoles.set(sid, userPrefs.role);

      // Detect project info
      const projectInfo = await detectProjectType(input.cwd);

      // Log to audit trail
      auditLogs.get(sid)!.push({
        time: new Date(input.timestamp).toISOString(),
        action: "session_start",
        detail: `Source: ${input.source}, Role: ${userPrefs.role}`,
      });

      // Compose context
      const contextParts = [
        `This is a ${projectInfo.type} project (${projectInfo.language}).`,
        `Package manager: ${projectInfo.packageManager}`,
        `User role: ${userPrefs.role}`,
        `Preferred language: ${userPrefs.language}`,
        `Code style: ${userPrefs.codeStyle}`,
      ];

      if (userPrefs.verbosity === "concise") {
        contextParts.push("Keep responses brief and to the point.");
      }

      // Compliance reminders
      contextParts.push(
        "",
        "【Compliance Reminders】",
        "- Do not include any API keys, passwords, or personal data in responses.",
        "- All file changes must be logged to the audit trail.",
        "- Direct shell command execution is prohibited; use approved tools instead."
      );

      return {
        additionalContext: contextParts.join("\n"),
      };
    },

    // User Prompt Submitted — Log to audit trail
    onUserPromptSubmitted: async (input, invocation) => {
      const sid = invocation.sessionId;
      const log = auditLogs.get(sid);

      if (log) {
        log.push({
          time: new Date().toISOString(),
          action: "user_prompt",
          detail: input.prompt.substring(0, 120) + (input.prompt.length > 120 ? "…" : ""),
        });
      }

      console.log(`[Prompt] ${input.prompt.substring(0, 80)}…`);

      return null;
    },

    // Pre-Tool Use — Permission control, block dangerous tools
    onPreToolUse: async (input, invocation) => {
      const sid = invocation.sessionId;
      const role = sessionRoles.get(sid) || "Unknown";
      const timestamp = new Date().toISOString();

      console.log(`[${timestamp}] Tool call: ${input.toolName}, Args: ${JSON.stringify(input.toolArgs)}`);

      // Log to audit trail
      const log = auditLogs.get(sid);
      if (log) {
        log.push({
          time: timestamp,
          action: "tool_call",
          tool: input.toolName,
          args: input.toolArgs,
          role,
        });
      }

      // Check 1: Absolute block list
      if (BLOCKED_TOOLS.includes(input.toolName)) {
        console.log(`  ✖ Blocked: ${input.toolName} (in block list)`);
        return {
          permissionDecision: "deny",
          permissionDecisionReason: `Tool "${input.toolName}" is blocked by security policy. Direct shell execution is not allowed.`,
        };
      }

      // Check 2: Role-based permission validation
      const permCheck = await checkToolPermission(role, input.toolName);
      if (!permCheck.allowed) {
        console.log(`  ✖ Access denied: ${permCheck.reason}`);
        return {
          permissionDecision: "deny",
          permissionDecisionReason: permCheck.reason,
        };
      }

      // Passed all checks: allow execution
      return { permissionDecision: "allow" };
    },

    // Post-Tool Use — Filter sensitive info, log results
    onPostToolUse: async (input, invocation) => {
      const sid = invocation.sessionId;
      const timestamp = new Date().toISOString();

      console.log(`[${timestamp}] Tool completed: ${input.toolName}`);

      // Log to audit trail
      const log = auditLogs.get(sid);
      if (log) {
        log.push({
          time: timestamp,
          action: "tool_result",
          tool: input.toolName,
          resultPreview: JSON.stringify(input.toolResult).substring(0, 200),
        });
      }

      // Filter sensitive data
      let modifiedResult = input.toolResult;
      if (typeof modifiedResult === "string") {
        modifiedResult = redactSensitiveData(modifiedResult);
      } else if (typeof modifiedResult === "object" && modifiedResult !== null) {
        // Redact string values in objects
        modifiedResult = JSON.parse(
          redactSensitiveData(JSON.stringify(modifiedResult))
        );
      }

      return {
        modifiedResult,
        additionalContext: `Tool "${input.toolName}" executed successfully.`,
      };
    },

    // Error Occurred — Error classification, auto-retry
    onErrorOccurred: async (input, invocation) => {
      const sid = invocation.sessionId;
      const timestamp = new Date().toISOString();

      console.error(`\n✖ [${timestamp}] Error in session ${sid}:`);
      console.error(`  Message: ${input.error}`);
      console.error(`  Context: ${input.errorContext}`);
      console.error(`  Recoverable: ${input.recoverable ? "Yes" : "No"}`);

      // Log to audit trail
      const log = auditLogs.get(sid);
      if (log) {
        log.push({
          time: timestamp,
          action: "error",
          context: input.errorContext,
          error: input.error,
          recoverable: input.recoverable,
        });
      }

      // Strategy A: Model call rate limit → Auto-retry
      if (input.errorContext === "model_call" && input.error.includes("rate")) {
        return {
          errorHandling: "retry",
          retryCount: 3,
          userNotification: "Rate limit hit. Retrying...",
        };
      }

      // Strategy B: Recoverable tool error → Skip and notify
      if (input.errorContext === "tool_execution" && input.recoverable) {
        return {
          suppressOutput: true,
          userNotification: "Tool execution encountered an issue. Skipped this step. Please try an alternative approach.",
        };
      }

      // Strategy C: Unrecoverable system error → Abort
      if (!input.recoverable) {
        return {
          errorHandling: "abort",
          userNotification: "An unrecoverable error occurred. Session will end.",
        };
      }

      // Strategy D: Other errors → Friendly message
      const friendlyMessages: Record<string, string> = {
        model_call: "There was an issue communicating with the AI model. Please try again.",
        tool_execution: "Tool execution failed. Please check parameters and retry.",
        system: "A system error occurred. Please contact the administrator.",
        user_input: "There was an issue with your input. Please check and retry.",
      };

      return {
        userNotification: friendlyMessages[input.errorContext] || null,
      };
    },

    // Session End — Generate audit report, cleanup
    onSessionEnd: async (input, invocation) => {
      const sid = invocation.sessionId;
      const startTime = sessionStartTimes.get(sid);
      const duration = startTime ? (input.timestamp - startTime) / 1000 : 0;

      const reasonLabel: Record<string, string> = {
        complete: "Complete",
        error: "Error",
        abort: "Aborted",
        timeout: "Timeout",
        user_exit: "User Exit",
      };

      // Count audit log entries
      const log = auditLogs.get(sid) || [];
      const promptCount = log.filter((e) => e.action === "user_prompt").length;
      const toolCalls = log.filter((e) => e.action === "tool_call").length;
      const toolDenied = log.filter((e) => e.action === "tool_call" && e.denied).length;
      const errorCount = log.filter((e) => e.action === "error").length;

      // Output audit report
      console.log(`
╔══════════════════════════════════════════════╗
║            Compliance Audit Report           ║
╠══════════════════════════════════════════════╣
║  Session ID: ${sid}
║  Duration: ${duration.toFixed(1)}s
║  End reason: ${reasonLabel[input.reason] || input.reason}
╠──────────────────────────────────────────────╣
║  Prompts: ${promptCount}
║  Tool calls: ${toolCalls}
║  Tools denied: ${toolDenied}
║  Errors: ${errorCount}
╠──────────────────────────────────────────────╣
║  Detailed operation log:
${log.map((e) => `║    [${e.time}] ${e.action}${e.tool ? ` → ${e.tool}` : ""}${e.detail ? ` — ${e.detail}` : ""}`).join("\n")}
╚══════════════════════════════════════════════╝
      `.trim());

      // Cleanup session data
      auditLogs.delete(sid);
      sessionStartTimes.delete(sid);
      sessionRoles.delete(sid);

      return null;
    },
  },
});

// Listen for streaming responses
session.on("assistant.message_delta", (event) => {
  process.stdout.write(event.data.deltaContent);
});

session.on("session.idle", () => {
  console.log(); // New line
});

// Send test prompt
await session.sendAndWait({
  prompt: "Please check the project for any leaked API keys or passwords, and list recommended improvements.",
});

// End session
await client.stop();
```

### Key Takeaways

1. **Layer security controls** - Multiple checkpoints (block list, role-based permissions)
2. **Redact sensitive data** - Always filter outputs before sending to LLM
3. **Comprehensive audit trail** - Log every interaction for compliance
4. **User-friendly error messages** - Don't expose internal error details
5. **graceful degradation** - Skip recoverable errors, abort unrecoverable ones
6. **Structured reporting** - Clear summary makes compliance review easy

---

## Best Practices Summary

### Hook Design Principles

1. **Keep hooks fast** - Don't block session flow with slow operations
2. **Return null when no changes** - SDK uses default behavior
3. **Handle all edge cases** - Sessions may end unexpectedly
4. **Log everything** - Audit trails are essential for production
5. **Fail gracefully** - Don't let hook errors crash the session

### Error Handling Strategies

1. **Classify errors** - Different contexts need different handling
2. **Track patterns** - Recurring errors indicate systemic issues
3. **Provide context** - Help users understand what went wrong
4. **Auto-retry judiciously** - Only for transient failures
5. **Alert on critical errors** - Don't let important issues go unnoticed

### Security & Compliance

1. **Permission control** - Block dangerous operations by default
2. **Data redaction** - Filter sensitive info from all outputs
3. **Audit everything** - Log all interactions with timestamps
4. **Role-based access** - Different user roles, different permissions
5. **Compliance reports** - Generate summaries for audit review

---

## Related Documentation

- [Hooks Overview](https://github.com/github/copilot-sdk/blob/main/docs/hooks/overview.md)
- [Session Lifecycle Hooks](https://github.com/github/copilot-sdk/blob/main/docs/hooks/session-lifecycle.md)
- [Error Handling Hook](https://github.com/github/copilot-sdk/blob/main/docs/hooks/error-handling.md)
- [Debugging Guide](https://github.com/github/copilot-sdk/blob/main/docs/debugging.md)
