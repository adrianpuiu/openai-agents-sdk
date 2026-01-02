---
name: openai-sdk-patterns
description: Use when creating agents with OpenAI Agents SDK for TypeScript, setting up agent projects, configuring tools, implementing handoffs, or needing guidance on OpenAI agent patterns like agents.run(), tool definitions, and orchestration.
---

# OpenAI Agents SDK Patterns

## Overview

The OpenAI Agents SDK for TypeScript provides modern tools for building AI agents with native orchestration, tool integration, and agent-to-agent handoffs. This skill covers essential patterns for creating production-ready agents.

**Core principle:** Agents are reusable units with specific tools, instructions, and the ability to hand off conversations to other agents.

## When to Use This

- Creating a new OpenAI agent project
- Defining agent tools and capabilities
- Implementing agent handoffs
- Configuring agent orchestration
- Troubleshooting agent setup issues

## Quick Reference

| Concept | SDK Pattern | Key Points |
|---------|-------------|------------|
| **Agent Creation** | `new Agent()` | Define name, instructions, tools |
| **Tools** | `tools: [...]` array | Define functions agent can call |
| **Handoffs** | `handoffs: [...]` array | Enable agent-to-agent transfers |
| **Running** | `agents.run()` | Execute agent with context |
| **Response** | `await response` | Get agent's response |

## Core SDK Patterns

### Agent Definition

```typescript
import { Agent, agents } from 'openai-agents-sdk';

const customerAgent = new Agent({
  name: 'customer-support',
  instructions: 'You help customers with orders and returns.',
  tools: [lookupOrder, processReturn],
  handoffs: [ escalationAgent ]
});
```

### Tool Definition

Tools are functions agents can call to interact with external systems:

```typescript
import { z } from 'zod';

const lookupOrder = {
  name: 'lookup_order',
  description: 'Look up a customer order by ID',
  parameters: z.object({
    orderId: z.string().describe('The order ID to look up')
  }),
  execute: async ({ orderId }) => {
    // Implementation
    return { orderId, status: 'shipped' };
  }
};
```

**Tool requirements:**
- `name`: Snake_case identifier
- `description`: What the tool does
- `parameters`: Zod schema for validation
- `execute`: Async function that implements the logic

### Handoffs

Handoffs enable agents to transfer conversations to specialized agents:

```typescript
import { handoff } from 'openai-agents-sdk';

const escalateToHuman = handoff({
  to: humanAgent,
  description: 'Escalate to a human agent'
});

const supportAgent = new Agent({
  name: 'support',
  instructions: 'Help with basic issues, escalate when needed',
  handoffs: [escalateToHuman]
});
```

### Running Agents

```typescript
import { agents } from 'openai-agents-sdk';

const response = await agents.run({
  agent: customerAgent,
  context: {
    customerId: '12345',
    message: 'Where is my order?'
  }
});

console.log(response.output);
```

## Project Structure

Standard OpenAI agent project layout:

```
my-agent/
├── package.json           # Dependencies: openai-agents-sdk, zod
├── tsconfig.json          # TypeScript config (ES modules)
├── .env                   # OPENAI_API_KEY
├── src/
│   ├── agents/           # Agent definitions
│   │   ├── customer.ts
│   │   └── escalation.ts
│   ├── tools/            # Tool implementations
│   │   ├── lookupOrder.ts
│   │   └── processReturn.ts
│   └── index.ts          # Entry point
└── tests/                # Agent tests
```

## Configuration Requirements

### package.json

```json
{
  "name": "my-agent",
  "type": "module",
  "scripts": {
    "start": "node --loader ts-node/esm src/index.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "openai-agents-sdk": "^1.0.0",
    "zod": "^3.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "ts-node": "^10.0.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

### .env.example

```
OPENAI_API_KEY=your_api_key_here
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `type: "module"` in package.json | Add it for ES modules support |
| Not defining Zod schemas for tools | All tool parameters need validation |
| Missing handoff descriptions | Clear descriptions help agents choose when to handoff |
| Hardcoding API keys | Use environment variables |

## Environment Setup

1. **Install dependencies:**
   ```bash
   npm install openai-agents-sdk zod
   ```

2. **Set API key:**
   ```bash
   export OPENAI_API_KEY=sk-...
   ```

3. **Run type check:**
   ```bash
   npm run typecheck
   ```

## For Detailed Reference

See `references/` directory for:
- Complete API documentation
- Advanced patterns (multi-agent, shared memory)
- Tool implementation examples
