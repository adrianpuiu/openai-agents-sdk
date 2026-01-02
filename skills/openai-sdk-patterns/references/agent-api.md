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

### Tool Interface

```typescript
interface Tool<T extends z.ZodType> {
  name: string;
  description: string;
  parameters: T;
  execute: (params: z.infer<T>) => Promise<any>;
}
```

### Example

```typescript
const searchTool: Tool<z.ZodObject<{query: z.ZodString}>> = {
  name: 'search',
  description: 'Search the web for information',
  parameters: z.object({
    query: z.string().describe('Search query')
  }),
  execute: async ({ query }) => {
    // Implementation
    return results;
  }
};
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
