# OpenAI Agents SDK Plugin

A Claude Code plugin by **AGP** that helps non-developers create AI agents using the OpenAI Agents SDK for TypeScript (package: `@openai/agents`). Generate complete, working agent projects from natural language descriptions.

**Marketplace**: agp-marketplace
**Version**: Updated with real-world learnings (January 2026)

## Features

- **Natural Language Project Creation**: Describe what you want, get a complete project
- **Automatic Tool Generation**: Tools are generated based on your agent's purpose
- **Multi-Agent Orchestration**: Support for agent handoffs and coordination
- **Complete Project Structure**: Includes TypeScript config, dependencies, tests, and docs
- **Built-in Verification**: Automatically validates generated code against SDK best practices
- **OpenAI Strict Mode Compliance**: All generated code follows OpenAI's structured output requirements
- **MCP Integration**: Support for Model Context Protocol (MCP) servers for external tool access

## Key Learnings from Real-World Usage

This plugin incorporates lessons learned from actual OpenAI Agents SDK projects:

### Critical API Patterns

1. **Package Name**: The npm package is `@openai/agents`, NOT `openai-agents-sdk`
2. **Tool Definition**: Must use `tool()` helper, not plain objects
3. **Entry Point**: Use `run()` function, not `agents.run()`
4. **Response Access**: Results are in `result.finalOutput`
5. **Agent Creation**: Use `new Agent()` constructor or `Agent.create()` for handoffs
6. **Streaming Events**: Use `run(agent, message, { stream: true })` and handle `raw_model_stream_event` for real-time output
7. **Text Delta Path**: Text deltas are in `event.data.delta` (string) when `event.data.type === 'output_text_delta'`
8. **Agent Updated Event**: Use `agent_updated_stream_event` to track agent changes during handoffs

### Zod Schema Requirements (Strict Mode)

OpenAI's API uses **strict mode** for structured outputs, which means:

| ❌ AVOID | ✅ USE | Reason |
|---------|---------|---------|
| `z.optional()` | Required fields | All fields must be required |
| `z.url()` | `z.string()` + description | "uri" format not supported |
| `z.any()` | `z.string()` for JSON | Schema must have type key |
| `.default()` | Always required | No defaults in strict mode |

### Environment Configuration

```typescript
// Load environment variables BEFORE SDK imports
import 'dotenv/config';
import { run } from '@openai/agents';
```

### Streaming Implementation

```typescript
import { run, MemorySession } from '@openai/agents';

// Enable streaming
const streamResult = await run(agent, message, {
  stream: true,
  maxTurns: 10,
  agentHooks: { /* hooks */ },
  inputGuardrails: [ /* guardrails */ ]
});

// Process stream events
for await (const event of streamResult) {
  switch (event.type) {
    case 'agent_updated_stream_event':
      // Agent changed (handoff)
      console.log(`Agent: ${event.agent.name}`);
      break;

    case 'raw_model_stream_event':
      // Text delta from model
      if (event.data?.type === 'output_text_delta' && typeof event.data.delta === 'string') {
        process.stdout.write(event.data.delta); // Stream in real-time
      }
      break;

    case 'run_handoff_stream_event':
      // Handoff occurred
      console.log(`Handoff: ${event.currentAgent.name} → ${event.targetAgent.name}`);
      break;

    case 'input_guardrail_tripwire_triggered_stream_event':
      // Guardrail blocked input
      const response = event.tripwireTriggeredData.responseData;
      console.error(`Blocked: ${response.refusalReason}`);
      return;
  }
}
```

**Key Streaming Event Types:**
- `agent_updated_stream_event` - Agent changed during execution
- `raw_model_stream_event` - Model response chunks (check `event.data.type === 'output_text_delta'`)
- `run_handoff_stream_event` - Agent handoff occurred
- `run_tool_call_stream_event` - Tool was called
- `input_guardrail_tripwire_triggered_stream_event` - Guardrail blocked input

### Session Management

```typescript
import { MemorySession } from '@openai/agents';

// Create a session for conversation context
const session = new MemorySession();

// Pass to run() for multi-turn conversations
const result = await run(agent, message, { session });

// Session maintains context across requests automatically
```

### Handoff Syntax

```typescript
// Correct pattern
handoff(agent, { toolDescriptionOverride: '...' })

// NOT this
handoff({ to: agent, ... })
```

### MCP Integration (TypeScript SDK)

The TypeScript SDK uses a different MCP pattern than Python:

```typescript
// Import MCP server classes
import { MCPServerStreamableHttp, MCPServerStdio, MCPServerSSE } from '@openai/agents';

// Create server instance (not mcp.createServer())
const mcpServer = new MCPServerStreamableHttp({
  url: 'https://gitmcp.io/openai/codex',
  name: 'GitMCP Server',
});

// Pass to agent
const agent = new Agent({
  name: 'Assistant',
  mcpServers: [mcpServer],  // Note: mcpServers (not mcp_servers)
});

// Always connect/close
await mcpServer.connect();
// ... use agent ...
await mcpServer.close();
```

**MCP Server Types:**
- `MCPServerStreamableHttp` - HTTP with streaming (recommended for remote servers)
- `MCPServerSSE` - Server-Sent Events
- `MCPServerStdio` - Local stdio communication

## Components

### Commands

- `/openai-agent-create` [description] - Create a new OpenAI agent project

### Skills

- `openai-sdk-patterns` - OpenAI Agents SDK patterns and best practices
- `agent-orchestration` - Agent handoffs and multi-agent coordination

### Agents

- `openai-agent-verifier` - Validates OpenAI agent projects

## Installation

1. Clone this repository or add to your Claude Code plugins directory:
   ```bash
   cp -r openai-agents-sdk ~/.claude/plugins/
   ```

2. Enable the plugin in Claude Code settings

## Usage

### Create an Agent

Describe what you need in natural language:

```
/openai-agent-create "I need a customer support agent that can look up orders, process returns, and escalate complex issues"
```

The plugin will:
1. Ask clarifying questions if needed
2. Generate a complete project structure
3. Create agent code with tools (using correct `tool()` helper)
4. Set up multi-agent handoffs if needed
5. Run verification to ensure everything compiles

### Interactive Mode

Run without arguments for interactive guidance:

```
/openai-agent-create
```

### Example Output

Generated project includes:
```
customer-support-agent/
├── package.json          # Uses @openai/agents (correct package name)
├── tsconfig.json          # ES2022 with bundler module resolution
├── .env.example           # OPENAI_API_KEY placeholder
├── .gitignore
├── README.md
├── src/
│   ├── agents/
│   │   ├── main-agent.ts
│   │   └── escalation-agent.ts
│   ├── tools/
│   │   ├── lookupOrder.ts    # Uses tool() helper
│   │   └── processReturn.ts  # Strict mode Zod schemas
│   └── index.ts             # Has dotenv/config import
└── tests/
    └── basic.test.ts
```

## What Gets Generated

### Agent Code
- Proper `@openai/agents` imports
- Clear agent instructions
- Tools using `tool()` helper (not plain objects)
- Handoff configuration: `handoff(agent, options)`

### Tools (Strict Mode Compliant)
- All Zod fields are **required** (no `.optional()`)
- No `z.url()` - use `z.string()` with descriptions
- No `z.any()` - use `z.string()` for JSON data
- Return `string` values: `JSON.stringify(result, null, 2)`
- Proper error handling with try-catch blocks

### Configuration
- `package.json`: `@openai/agents` + `dotenv` + `zod`
- `tsconfig.json`: ES2022, bundler resolution
- `.env.example`: API key template
- `.gitignore`: Security best practices

### Documentation
- README with setup instructions
- Code comments explaining each part
- Usage examples
- TypeScript compilation instructions

## Prerequisites

- Node.js 18+ installed
- OpenAI API key from https://platform.openai.com/api-keys
- npm or yarn package manager

## After Creation

1. Add your API key:
   ```bash
   cd your-project-name
   cp .env.example .env
   # Edit .env and add your OPENAI_API_KEY
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Run your agent:
   ```bash
   npm start "Your test message"
   ```

## Verification

The plugin automatically runs the verifier agent after project creation. To manually verify:

1. Navigate to your project
2. Run type check: `npm run typecheck`
3. The verifier checks:
   - Correct package name (`@openai/agents`)
   - TypeScript configuration
   - SDK usage patterns (`tool()` helper, `run()` function)
   - Zod schema compliance (no `.optional()`, `.url()`, or `z.any()`)
   - Environment variable setup (dotenv)
   - Handoff syntax
   - Security (API keys not hardcoded)
   - Documentation completeness

## Common Patterns

### Single Agent
```
/openai-agent-create "A research assistant that searches the web and summarizes findings"
```

### Multi-Agent with Handoffs
```
/openai-agent-create "A support system with a triage agent that routes to billing and technical specialists"
```

### Agent with Specific Tools
```
/openai-agent-create "An analytics agent that can query databases, generate charts, and export reports"
```

## Troubleshooting Common Issues

### "Missing credentials" error
- **Cause**: `.env` file not loaded
- **Fix**: Ensure `import 'dotenv/config'` is FIRST in `src/index.ts`

### "Invalid schema" error
- **Cause**: Zod schema issues (`.optional()`, `.url()`, or `z.any()`)
- **Fix**: Use required fields, `z.string()` with descriptions, or `z.string()` for JSON

### "uri is not a valid format" error
- **Cause**: Using `z.string().url()`
- **Fix**: Use `z.string()` with a description like "URL (e.g., https://example.com)"

### "tool() is not a function" error
- **Cause**: Not importing `tool` from `@openai/agents`
- **Fix**: `import { tool } from '@openai/agents';`

## Skills Reference

The plugin includes skills that provide detailed guidance:

- **openai-sdk-patterns**: Core SDK patterns, tool definitions, agent configuration
- **agent-orchestration**: Handoffs, multi-agent coordination, triage patterns

These skills activate automatically when relevant.

## Examples & Templates

The plugin includes comprehensive examples and templates to help you get started:

### Examples (`examples/`)

**Core Patterns:**
- **single-agent-with-tools.md** - Complete single agent with multiple tools
- **multi-agent-with-handoffs.md** - Multi-agent triage pattern implementation
- **advanced-patterns.md** - Agent-as-tool, memory, streaming, error recovery
- **triage-pattern.md** - Detailed guide to triage pattern best practices
- **mcp-integration.md** - Model Context Protocol (MCP) server integration
- **streaming-cli.md** - **NEW**: Complete streaming CLI with sessions, guardrails, and hooks

**Advanced Patterns:**
- **human-in-the-loop.md** - Human approval workflows for sensitive operations
- **guardrails.md** - Input/output validation and safety checks
- **parallelization.md** - Running multiple agents in parallel
- **structured-outputs.md** - JSON schema outputs and Zod validation
- **built-in-tools.md** - Using web search, file search, and computer automation tools

### Templates (`templates/`)

Ready-to-customize agent project templates:

- **customer-support-agent/** - Full customer support agent with order lookup, returns, and escalation
- **research-agent/** - Research and data analysis agent with web search and summarization
- **workflow-automation-agent/** - Workflow automation with record management and notifications

## Contributing

This plugin follows the Claude Code plugin development patterns. See the [plugin-dev](https://github.com/anthropics/claude-code-plugins) repository for plugin development guidelines.

## License

MIT

## Author

**AGP** - Created for the agp-marketplace.
Updated with real-world learnings from building production multi-agent systems with the OpenAI Agents SDK.

