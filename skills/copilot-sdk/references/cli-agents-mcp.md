# CLI, Custom Agents, MCP Servers, and Skills

## GitHub Copilot CLI Configuration

The SDK uses Copilot CLI as its engine. Configuration is stored in `~/.copilot/` (or `$XDG_CONFIG_HOME/copilot/`):

- `config.json` - General configuration
- `mcp-config.json` - MCP server definitions

For full CLI documentation, see [Using GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli).

---

# Custom Agents

Custom agents are specialized configurations with tailored expertise for specific tasks.

See [Creating Custom Agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents) and [Custom Agents Configuration Reference](https://docs.github.com/en/copilot/reference/custom-agents-configuration).

## Agent Profile Structure (Markdown files)

Custom agents use Markdown files with YAML frontmatter:

```yaml
---
name: my-agent
description: A specialized agent for specific tasks
tools: ["read", "edit", "search"]
target: github-copilot
mcp-servers:
  custom-mcp:
    type: 'local'
    command: 'mcp-server'
    args: ['--flag']
---

# Agent Instructions

Detailed behavioral instructions here (max 30,000 characters).
```

## YAML Frontmatter Properties

| Property | Type | Purpose |
|----------|------|---------|
| `name` | string | Display name (optional, defaults to filename) |
| `description` | string | **Required** - describes purpose and capabilities |
| `target` | string | Environment: `vscode` or `github-copilot` |
| `tools` | list/string | Available tools; defaults to all if unset |
| `infer` | boolean | Allow automatic selection based on context |
| `mcp-servers` | object | MCP server configurations (org/enterprise level) |

## Tools Configuration

Configure tool access:

- **Enable all:** Omit `tools` or use `tools: ["*"]`
- **Enable specific:** `tools: ["read", "edit", "search"]`
- **Disable all:** `tools: []`

**Tool Aliases:**
- `execute` - Run shell commands
- `read` - Access file contents
- `edit` - Modify files
- `search` - Find files/text (grep, glob)
- `agent` - Invoke other custom agents
- `web` - Web search/fetch

**MCP Tool References:** `mcp-server-name/tool-name` or `mcp-server-name/*`

## Agent Storage Locations

- **Repository:** `.github/agents/` directory
- **Organization/Enterprise:** Configured via GitHub settings
- **System:** CLI includes default agents

**Priority (naming conflicts):** System > Repository > Organization

## Programmatic Custom Agents (via SDK)

Custom agents can also be defined programmatically in SessionConfig:

```typescript
const session = await client.createSession({
  model: "gpt-5",
  customAgents: [
    {
      name: "code-reviewer",
      displayName: "Code Reviewer",
      description: "Reviews code for quality and security issues",
      prompt: "You are an expert code reviewer...",
      tools: ["read", "search"],   // null for all tools
      infer: true,                 // Allow auto-selection
      mcpServers: {                // Agent-specific MCP servers
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

### CustomAgentConfig fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier |
| `displayName` | string | No | UI display name |
| `description` | string | No | Agent description |
| `tools` | string[] \| null | No | Tool whitelist (null = all) |
| `prompt` | string | Yes | Agent instructions |
| `mcpServers` | Record<string, MCPServerConfig> | No | Agent-specific MCP servers |
| `infer` | boolean | No | Allow auto-selection (default: true) |

### Subagent Events

When subagents are invoked, monitor via events:
- `subagent.selected` — agent was chosen (includes `tools` list)
- `subagent.started` — execution began (includes `agentDescription`)
- `subagent.completed` — execution finished
- `subagent.failed` — execution failed (includes `error`)

---

# MCP Server Integration

Model Context Protocol (MCP) servers extend Copilot's capabilities with external tools and resources.

See [Enhancing Agent Mode with MCP](https://docs.github.com/en/copilot/tutorials/enhance-agent-mode-with-mcp) and [Setting up the GitHub MCP Server](https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp/set-up-the-github-mcp-server).

## Default MCP Server

Copilot CLI comes with the GitHub MCP server pre-configured, enabling:
- Repository access and management
- Pull request operations
- Issue tracking
- GitHub workflow management

## MCP Configuration via CLI Config File

MCP servers are configured in `~/.copilot/mcp-config.json`:

```json
{
  "mcpServers": {
    "server-name": {
      "type": "local",
      "command": "mcp-server-command",
      "args": ["--arg1", "--arg2"],
      "env": {
        "API_KEY": "$COPILOT_MCP_API_KEY"
      }
    }
  }
}
```

## MCP Configuration via SDK (Programmatic)

MCP servers can be configured per-session in SessionConfig:

### Local/Stdio MCP Server

```typescript
const session = await client.createSession({
  mcpServers: {
    "filesystem": {
      type: "local",       // "local" | "stdio"
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      tools: ["*"],        // All tools, or specific: ["read_file", "write_file"]
      env: { NODE_ENV: "production" },
      cwd: "/path/to/working/dir",
      timeout: 30000,
    },
  },
});
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"local"` \| `"stdio"` | No | Server type (default: "local") |
| `command` | string | Yes | Executable command |
| `args` | string[] | Yes | Command arguments |
| `tools` | string[] | Yes | Tool filter (`["*"]` for all, `[]` for none) |
| `env` | Record<string, string> | No | Environment variables |
| `cwd` | string | No | Working directory |
| `timeout` | number | No | Timeout in milliseconds |

### Remote MCP Server (HTTP/SSE)

```typescript
const session = await client.createSession({
  mcpServers: {
    "remote-api": {
      type: "http",        // "http" | "sse"
      url: "https://mcp.example.com/api",
      headers: {
        "Authorization": "Bearer token123",
      },
      tools: ["search", "fetch"],
      timeout: 60000,
    },
  },
});
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"http"` \| `"sse"` | Yes | Server type |
| `url` | string | Yes | Server URL |
| `headers` | Record<string, string> | No | HTTP headers |
| `tools` | string[] | Yes | Tool filter |
| `timeout` | number | No | Timeout in milliseconds |

## MCP Language Conventions

| Concept | TypeScript | Python | Go | .NET |
|---------|-----------|--------|-----|------|
| MCP config key | `mcpServers` | `mcp_servers` | `MCPServers` | `McpServers` |
| Local config type | `MCPLocalServerConfig` | `MCPLocalServerConfig` | `MCPLocalServerConfig` | `McpLocalServerConfig` |
| Remote config type | `MCPRemoteServerConfig` | `MCPRemoteServerConfig` | `MCPRemoteServerConfig` | `McpRemoteServerConfig` |

## Environment Variables in MCP Config

Supported patterns in `mcp-config.json`:
- `$COPILOT_MCP_VAR`
- `${COPILOT_MCP_VAR}`
- `${{ secrets.COPILOT_MCP_VAR }}`
- `${{ var.COPILOT_MCP_VAR }}`

## MCP Tool Events

Monitor MCP tool execution:
- `tool.execution_start` — includes `mcpServerName` and `mcpToolName` fields
- `tool.execution_complete` — includes `success`, `result`, `error`
- `tool.execution_progress` — progress updates (ephemeral)
- `tool.execution_partial_result` — partial results (ephemeral)

---

# Skills System

Skills are reusable instruction sets that can be loaded into sessions to provide specialized knowledge.

See [Skills Guide](https://github.com/github/copilot-sdk/blob/main/docs/guides/skills.md).

## Configuring Skills

```typescript
const session = await client.createSession({
  model: "gpt-5",
  skillDirectories: ["/path/to/skills"],   // Directories to scan for skills
  disabledSkills: ["unwanted-skill"],       // Skills to exclude
});
```

## Skill Language Conventions

| Concept | TypeScript | Python | Go | .NET |
|---------|-----------|--------|-----|------|
| Skill directories | `skillDirectories` | `skill_directories` | `SkillDirectories` | `SkillDirectories` |
| Disabled skills | `disabledSkills` | `disabled_skills` | `DisabledSkills` | `DisabledSkills` |

## Skill Events

When skills are loaded, a `skill.invoked` event fires:

```typescript
session.on((event) => {
  if (event.type === "skill.invoked") {
    console.log(`Name: ${event.data.name}`);
    console.log(`Path: ${event.data.path}`);
    console.log(`Content: ${event.data.content}`);
    console.log(`Allowed tools: ${event.data.allowedTools}`);
  }
});
```

---

# Tool Access Control

## Available Tools (Whitelist)

Only allow specific tools — all others are disabled:

```typescript
const session = await client.createSession({
  availableTools: ["read", "search", "my-custom-tool"],
  // All other tools disabled
});
```

## Excluded Tools (Blacklist)

Disable specific tools — all others remain available:

```typescript
const session = await client.createSession({
  excludedTools: ["execute", "edit"],
  // All other tools available
});
```

**Priority:** `availableTools` takes precedence over `excludedTools`.

## Tool Access Language Conventions

| Concept | TypeScript | Python | Go | .NET |
|---------|-----------|--------|-----|------|
| Whitelist | `availableTools` | `available_tools` | `AvailableTools` | `AvailableTools` |
| Blacklist | `excludedTools` | `excluded_tools` | `ExcludedTools` | `ExcludedTools` |
