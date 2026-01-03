# OpenAI Agents SDK Plugin

A Claude Code plugin that helps non-developers create AI agents using the OpenAI Agents SDK for TypeScript (package: `@openai/agents`). Generate complete, working agent projects from natural language descriptions.

## Features

- **Natural Language Project Creation**: Describe what you want, get a complete project
- **Automatic Tool Generation**: Tools are generated based on your agent's purpose
- **Multi-Agent Orchestration**: Support for agent handoffs and coordination
- **Complete Project Structure**: Includes TypeScript config, dependencies, tests, and docs
- **Built-in Verification**: Automatically validates generated code against SDK best practices
- **OpenAI Strict Mode Compliance**: All generated code follows OpenAI's structured output requirements

## Key Learnings from Real-World Usage

This plugin incorporates lessons learned from actual OpenAI Agents SDK projects:

### Critical API Patterns

1. **Package Name**: The npm package is `@openai/agents`, NOT `openai-agents-sdk`
2. **Tool Definition**: Must use `tool()` helper, not plain objects
3. **Entry Point**: Use `run()` function, not `agents.run()`
4. **Response Access**: Results are in `result.finalOutput`

### Zod Schema Requirements (Critical!)

**IMPORTANT: Always use Zod schemas for tool parameters. Never use manual JSON Schema objects.**

OpenAI's API uses **strict mode** for structured outputs. The `tool()` helper from `@openai/agents` requires Zod schemas:

| ❌ AVOID | ✅ USE | Reason |
|---------|---------|---------|
| Manual JSON Schema with `as any` | `z.object({...})` | Zod provides proper type inference and validation |
| `strict: true` on parameters | Zod schema directly | Manual schema causes parameter parsing failures |
| `z.optional()` | Required fields | All fields must be required in strict mode |
| `z.url()` | `z.string()` + description | "uri" format not supported |
| `z.any()` | `z.string()` for JSON | Schema must have type key |
| `.default()` | Always required | No defaults in strict mode |

**Why Zod is mandatory:**
- Type inference: Parameters are automatically typed in the execute function
- Runtime validation: Invalid inputs are caught before execution
- Proper schema communication: The AI model receives correctly formatted schemas
- No type casting needed: Eliminates dangerous `as any` casts

**Example of correct tool definition:**
```typescript
import { tool } from '@openai/agents';
import { z } from 'zod';

export const myTool = tool({
  description: 'Tool description here',
  parameters: z.object({
    action: z.enum(['add', 'list', 'format']).describe('Action to perform'),
    data: z.string().describe('Data to process'),
    count: z.number().default(5).describe('Number of items')  // default() works in Zod
  }),
  execute: async ({ action, data, count }) => {
    // Parameters are properly typed - no casting needed
    return { success: true, result: `${action}: ${data}` };
  }
});
```

**Common pitfall - manual JSON schema (DO NOT DO THIS):**
```typescript
// ❌ WRONG - Causes "Action: undefined" loops and parameter failures
export const badTool = tool({
  description: 'Tool description',
  strict: true,  // Doesn't help with manual schemas
  parameters: {
    type: 'object',
    properties: { action: { type: 'string' } },
    required: ['action']
  } as any,  // ⚠️ Dangerous! Bypasses type checking
  execute: async (input: any) => {
    // input.action will be undefined because schema isn't parsed correctly
  }
});
```

### Environment Configuration

```typescript
// Load environment variables BEFORE SDK imports
import 'dotenv/config';
import { run } from '@openai/agents';
```

### Handoff Syntax

```typescript
// Correct pattern
handoff(agent, { toolDescriptionOverride: '...' })

// NOT this
handoff({ to: agent, ... })
```

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

- **single-agent-with-tools.md** - Complete single agent with multiple tools
- **multi-agent-with-handoffs.md** - Multi-agent triage pattern implementation
- **advanced-patterns.md** - Agent-as-tool, memory, streaming, error recovery
- **triage-pattern.md** - Detailed guide to triage pattern best practices

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

Created for the Claude Code plugins marketplace.

