---
name: openai-sdk-patterns
description: Use when creating agents with OpenAI Agents SDK for TypeScript (@openai/agents package), setting up agent projects, configuring tools, implementing streaming, using sessions, or needing guidance on OpenAI agent patterns like run(), tool definitions with tool() helper, handoffs, guardrails, and hooks.
---

# OpenAI Agents SDK Patterns

## Overview

The OpenAI Agents SDK for TypeScript (`@openai/agents`) provides modern tools for building AI agents with native orchestration, tool integration, agent-to-agent handoffs, streaming, sessions, and guardrails.

**Core principle:** Agents are reusable units with specific tools, instructions, and the ability to hand off conversations to other agents.

## When to Use This

- Creating a new OpenAI agent project with `@openai/agents`
- Defining agent tools using the `tool()` helper
- Implementing streaming responses
- Setting up session management
- Configuring agent handoffs
- Adding input guardrails or agent hooks

## Quick Reference

| Concept | SDK Pattern | Key Points |
|---------|-------------|------------|
| **Package Name** | `@openai/agents` | NOT `openai-agents-sdk` |
| **Agent Creation** | `new Agent()` | Define name, instructions, tools |
| **Tools** | `tool()` helper | Use `tool()`, not plain objects |
| **Running** | `run()` function | Use `run()`, not `agents.run()` |
| **Streaming** | `run(agent, msg, {stream: true})` | Handle `raw_model_stream_event` |
| **Sessions** | `new MemorySession()` | For conversation context |
| **Handoffs** | `handoff(agent, options)` | Enable agent transfers |
| **Guardrails** | `inputGuardrails` array | Validate input before processing |
| **Hooks** | `agentHooks` object | Monitor agent lifecycle |

## Core SDK Patterns

### Import Statements

```typescript
// Correct package name is @openai/agents
import { Agent, run, tool, handoff, MemorySession } from '@openai/agents';
import 'dotenv/config'; // Must be FIRST import
```

### Agent Definition

```typescript
import { Agent, handoff } from '@openai/agents';

const customerAgent = new Agent({
  name: 'customer-support',
  instructions: 'You help customers with orders and returns.',
  tools: [lookupOrder, processReturn],
  handoffs: [
    handoff(escalationAgent, {
      toolDescriptionOverride: 'Escalate to human agent'
    })
  ]
});
```

### Tool Definition (Critical: Use `tool()` Helper)

```typescript
import { tool } from '@openai/agents';
import { z } from 'zod';

const lookupOrder = tool({
  description: 'Look up a customer order by ID',
  parameters: z.object({
    orderId: z.string().describe('The order ID to look up')
  }),
  execute: async ({ orderId }) => {
    // Tool implementation
    return JSON.stringify({
      orderId,
      status: 'shipped',
      estimatedDelivery: '2025-01-10'
    }, null, 2);
  }
});
```

**Critical requirements:**
- Use `tool()` helper - NOT plain objects
- All parameters must be **required** (no `.optional()`)
- Return `JSON.stringify(result, null, 2)` - NOT objects directly
- Avoid `z.url()` - use `z.string()` with description instead

### Running Agents

```typescript
import { run } from '@openai/agents';

// Non-streaming
const result = await run(mainAgent, 'What is the status of order 12345?');
console.log(result.finalOutput);

// Streaming with events
const streamResult = await run(mainAgent, message, {
  stream: true,
  maxTurns: 10
});

for await (const event of streamResult) {
  if (event.type === 'raw_model_stream_event') {
    // Handle streaming
  }
}
```

## Streaming Implementation

Streaming is essential for real-time user feedback. Here's the correct pattern:

```typescript
const streamResult = await run(agent, message, {
  stream: true,
  maxTurns: 10
});

for await (const event of streamResult) {
  switch (event.type) {
    case 'agent_updated_stream_event':
      // Agent changed (handoff occurred)
      console.log(`\n[${event.agent.name}]`);
      break;

    case 'raw_model_stream_event':
      // Text delta from model
      // IMPORTANT: Check event.data.type === 'output_text_delta'
      // IMPORTANT: event.data.delta is a STRING, not an object
      if (event.data?.type === 'output_text_delta' && typeof event.data.delta === 'string') {
        process.stdout.write(event.data.delta);
      }
      break;

    case 'run_handoff_stream_event':
      // Handoff occurred
      console.log(`[handoff: ${event.currentAgent.name} → ${event.targetAgent.name}]`);
      break;

    case 'run_tool_call_stream_event':
      // Tool was called
      console.log(`[tool: ${event.toolName}]`);
      break;
  }
}
```

**Key streaming facts:**
- Event type is `raw_model_stream_event` (NOT `run_raw_model_stream_event`)
- Text is in `event.data.delta` as a `string`
- Check `event.data.type === 'output_text_delta'` first
- Use `typeof event.data.delta === 'string'` for safety

## Session Management

For multi-turn conversations with context:

```typescript
import { MemorySession } from '@openai/agents';

// Create a session
const session = new MemorySession();

// Use with run()
const result = await run(agent, message, { session });

// Session maintains conversation context automatically
```

## Handoffs

Handoffs enable agents to transfer conversations to specialized agents:

```typescript
import { handoff } from '@openai/agents';

const supportAgent = new Agent({
  name: 'support',
  instructions: 'Help with basic issues, escalate when needed',
  handoffs: [
    handoff(billingAgent, {
      toolDescriptionOverride: 'Delegate to billing specialist for payment issues, refunds, and account questions'
    }),
    handoff(technicalAgent, {
      toolDescriptionOverride: 'Delegate to technical support for complex troubleshooting'
    })
  ]
});
```

## Input Guardrails

Validate user input before processing:

```typescript
import { InputGuardrail } from '@openai/agents';

const safetyGuardrail: InputGuardrail<
  z.ZodObject<{ message: z.ZodString }>,
  {}
> = {
  name: 'safety_check',
  schema: z.object({ message: z.string() }),
  validate: async ({ message }) => {
    if (message.includes('harmful')) {
      return {
        tripwired: true,
        responseData: {
          refusalReason: 'Request blocked: harmful content detected'
        }
      };
    }
    return { tripwired: false };
  }
};

// Use with run()
const result = await run(agent, message, {
  inputGuardrails: [safetyGuardrail]
});
```

## Agent Hooks

Monitor agent lifecycle events:

```typescript
import { AgentHooks } from '@openai/agents';

const agentHooks: AgentHooks = {
  async beforeAgentTurn({ agent, turn }) {
    console.log(`Starting turn ${turn} for ${agent.name}`);
  },
  async afterAgentTurn({ agent, turn }) {
    console.log(`Completed turn ${turn} for ${agent.name}`);
  },
  async beforeToolCall({ agent, toolName }) {
    console.log(`Calling tool: ${toolName}`);
  },
  async onError({ agent, error, turn }) {
    console.error(`Error in ${agent.name} at turn ${turn}:`, error);
  }
};

// Use with run()
const result = await run(agent, message, {
  agentHooks
});
```

## Project Structure

Standard OpenAI agent project layout:

```
my-agent/
├── package.json           # Dependencies: @openai/agents, zod, dotenv
├── tsconfig.json          # TypeScript config (ES2022, bundler resolution)
├── .env.example           # OPENAI_API_KEY template
├── .gitignore
├── README.md
├── src/
│   ├── config/
│   │   └── index.ts       # Centralized configuration
│   ├── agents/           # Agent definitions
│   │   ├── main-agent.ts
│   │   └── specialist.ts
│   ├── tools/            # Tool implementations
│   │   └── myTool.ts     # Use tool() helper
│   ├── guardrails/       # Input validation (optional)
│   │   └── index.ts
│   ├── hooks/            # Agent lifecycle hooks (optional)
│   │   └── index.ts
│   └── index.ts          # Entry point with dotenv/config first
└── tests/
```

## Configuration Requirements

### package.json

```json
{
  "name": "my-agent",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "start": "tsx src/index.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@openai/agents": "^0.3.7",
    "dotenv": "^17.0.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "tsx": "^4.21.0",
    "typescript": "^5.7.0"
  }
}
```

**Use `tsx` instead of `ts-node`** for better ES module support.

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "esModuleInterop": true
  },
  "files": [],
  "references": [
    { "path": "./src" }
  ]
}
```

### .env.example

```bash
# OpenAI Configuration
OPENAI_API_KEY=sk-your-api-key-here
OPENAI_MODEL=gpt-4o-mini

# Application Settings
MAX_TURNS=10
ENABLE_TRACING=false
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Wrong package name `openai-agents-sdk` | Use `@openai/agents` |
| Not using `tool()` helper | Use `tool({...})` for all tools |
| Using `.optional()` in Zod schemas | All fields must be required |
| Using `z.url()` | Use `z.string()` with description |
| Returning objects from tools | Return `JSON.stringify(result, null, 2)` |
| Wrong event type `run_raw_model_stream_event` | Use `raw_model_stream_event` |
| Wrong delta path `event.delta.text` | Use `event.data.delta` (string) |
| Missing `import 'dotenv/config'` | Must be first import |
| Using `ts-node` | Use `tsx` for better ES module support |

## Environment Setup

1. **Install dependencies:**
   ```bash
   npm install @openai/agents zod dotenv
   npm install -D tsx typescript @types/node
   ```

2. **Set API key:**
   ```bash
   cp .env.example .env
   # Edit .env and add OPENAI_API_KEY
   ```

3. **Run type check:**
   ```bash
   npm run typecheck
   ```

4. **Run agent:**
   ```bash
   npm start "Your message here"
   ```

## For Detailed Reference

See `references/` directory for:
- Complete API documentation
- Streaming patterns and event handling
- Guardrail implementation patterns
- Tool implementation examples
