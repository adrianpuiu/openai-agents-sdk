# OpenAI Agents SDK API Reference

## Agent Class

### Constructor

```typescript
new Agent(options)
```

**Options:**
- `name` (string, required): Agent identifier
- `instructions` (string, required): System prompt for the agent
- `tools` (Tool[], optional): Functions the agent can call
- `handoffs` (Handoff[], optional): Agents this can transfer to
- `model` (string, optional): Model to use (default: gpt-4o)

### Example

```typescript
const agent = new Agent({
  name: 'assistant',
  instructions: 'You help users with tasks.',
  tools: [searchTool, calculatorTool],
  handoffs: [escalationHandoff]
});
```

## Tool Definition

**CRITICAL: Always use the `tool()` helper with Zod schemas. Never define tools as plain objects with manual JSON schemas.**

### Tool Interface (using `tool()` helper)

```typescript
import { tool } from '@openai/agents';
import { z } from 'zod';

// ✅ CORRECT - Using tool() helper with Zod schema
const myTool = tool({
  description: 'Tool description',
  parameters: z.object({
    param1: z.string().describe('First parameter'),
    param2: z.number().describe('Second parameter')
  }),
  execute: async ({ param1, param2 }) => {
    // Implementation - parameters are properly typed
    return { result: 'done' };
  }
});
```

### Why Use `tool()` Helper?

| Feature | `tool()` Helper | Plain Object |
|---------|----------------|--------------|
| Type Safety | Full inference | Manual typing required |
| Runtime Validation | Automatic | None |
| AI Schema Communication | Proper format | May fail |
| Required Casts | None | Needs `as any` |
| Parameter Parsing | Built-in | Manual in execute |

### Common Pitfall - Manual JSON Schema (DO NOT USE)

```typescript
// ❌ WRONG - Causes parameter parsing failures
const badTool = {
  name: 'bad_tool',
  description: 'This will fail',
  parameters: {
    type: 'object',
    properties: { action: { type: 'string' } },
    required: ['action']
  } as any,  // ⚠️ Dangerous!
  execute: async (input: any) => {
    // input.action will be undefined - loop occurs!
  }
};
```

### Example: Correct Tool with All Features

```typescript
import { tool } from '@openai/agents';
import { z } from 'zod';

const searchTool = tool({
  description: 'Search the web for information and return relevant results',
  parameters: z.object({
    query: z.string().min(1).describe('The search query to execute'),
    numResults: z.number().min(1).max(20).default(5).describe('Number of results to return (1-20)')
  }),
  execute: async ({ query, numResults }) => {
    // Parameters are fully typed: query is string, numResults is number
    console.log(`Searching: "${query}" for ${numResults} results`);

    // Your implementation here
    return {
      query,
      results: [],
      count: 0
    };
  }
});
```

## Handoff Function

```typescript
handoff(options)
```

**Options:**
- `to` (Agent, required): Target agent
- `description` (string, required): When to use this handoff
- `context` (object, optional): Context to pass to target

### Example

```typescript
const escalateToSpecialist = handoff({
  to: specialistAgent,
  description: 'Use when the issue requires specialized knowledge',
  context: { escalatedAt: new Date() }
});
```

## agents.run()

```typescript
await agents.run(options)
```

**Options:**
- `agent` (Agent, required): Agent to run
- `context` (object, optional): Context for the conversation
- `message` (string, optional): Initial message

**Returns:** Promise<AgentResponse>

### Example

```typescript
const response = await agents.run({
  agent: supportAgent,
  context: { userId: '123' },
  message: 'I need help with my account'
});
```

## AgentResponse

```typescript
interface AgentResponse {
  output: string;
  toolCalls: ToolCall[];
  handoffs: Handoff[];
  context: Record<string, any>;
}
```

## Utility Functions

### createTool()

Helper for creating tools with runtime validation:

```typescript
const myTool = createTool({
  name: 'calculate',
  description: 'Perform a calculation',
  schema: z.object({
    a: z.number(),
    b: z.number()
  }),
  handler: async ({ a, b }) => a + b
});
```
