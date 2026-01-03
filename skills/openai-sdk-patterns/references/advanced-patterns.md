# Advanced OpenAI Agent Patterns

> **CRITICAL REMINDER**: All tools must use the `tool()` helper with Zod schemas.
> Never use manual JSON Schema objects - they cause parameter parsing failures and infinite loops.
> See `agent-api.md` for proper tool definition syntax.

## Multi-Agent Orchestration

### Agent Network Pattern

```typescript
import { Agent, agents, handoff } from 'openai-agents-sdk';

// Define specialized agents
const triageAgent = new Agent({
  name: 'triage',
  instructions: 'Categorize requests and route to appropriate agent',
  handoffs: [
    handoff({ to: salesAgent, description: 'Sales inquiries' }),
    handoff({ to: supportAgent, description: 'Technical support' }),
    handoff({ to: billingAgent, description: 'Billing questions' })
  ]
});

const salesAgent = new Agent({
  name: 'sales',
  instructions: 'Handle product inquiries and sales',
  tools: [productCatalog, pricingTool]
});

const supportAgent = new Agent({
  name: 'support',
  instructions: 'Provide technical support',
  tools: [knowledgeBase, ticketSystem]
});

const billingAgent = new Agent({
  name: 'billing',
  instructions: 'Handle billing and account issues',
  tools: [accountLookup, paymentTool]
});

// Entry point
const response = await agents.run({
  agent: triageAgent,
  message: 'I need to update my credit card'
});
```

## Shared Context Across Agents

### Context Accumulation Pattern

```typescript
interface ConversationContext {
  userId: string;
  history: Message[];
  accumulatedData: Record<string, any>;
}

const contextAccumulator = {
  beforeHandoff: (context: ConversationContext) => {
    // Add data before handoff
    context.accumulatedData.lastHandoff = new Date();
    return context;
  },
  afterHandoff: (context: ConversationContext, handoffResult: any) => {
    // Process handoff result
    context.accumulatedData.handoffResults ??= [];
    context.accumulatedData.handoffResults.push(handoffResult);
    return context;
  }
};
```

## Tool Composition

### Tool Chaining

```typescript
import { tool } from '@openai/agents';
import { z } from 'zod';

const validateAndExecute = tool({
  description: 'Validate input then execute action',
  parameters: z.object({
    action: z.enum(['create', 'update', 'delete']).describe('Action to perform'),
    data: z.string().describe('JSON string of data to validate and process')
  }),
  execute: async ({ action, data }) => {
    // Parse JSON string to object
    const parsedData = JSON.parse(data);

    // Validation step
    const validation = await validateSchema(parsedData);
    if (!validation.valid) {
      return { error: validation.errors };
    }

    // Execution step
    switch (action) {
      case 'create':
        return await createRecord(parsedData);
      case 'update':
        return await updateRecord(parsedData);
      case 'delete':
        return await deleteRecord(parsedData.id);
    }
  }
});
```

## Error Handling

### Graceful Tool Failures

```typescript
import { tool } from '@openai/agents';
import { z } from 'zod';

const robustTool = tool({
  description: 'Operation with comprehensive error handling',
  parameters: z.object({
    input: z.string().describe('Input to process')
  }),
  execute: async ({ input }) => {
    try {
      const result = await performOperation(input);
      return { success: true, data: result };
    } catch (error) {
      // Log error for monitoring
      console.error('Tool error:', error);

      // Return user-friendly message
      return {
        success: false,
        error: error instanceof Error
          ? error.message
          : 'Unknown error occurred',
        retryable: isRetryable(error)
      };
    }
  }
});

function isRetryable(error: unknown): boolean {
  return error instanceof RetryableError;
}
```

## Streaming Responses

```typescript
const response = await agents.run({
  agent: myAgent,
  message: 'Tell me a story',
  stream: true
});

for await (const chunk of response) {
  process.stdout.write(chunk.delta);
}
```

## Agent Tool Use

### Agent as Tool Pattern

```typescript
import { tool, Agent, agents } from '@openai/agents';
import { z } from 'zod';

const specialist = new Agent({
  name: 'specialist',
  instructions: 'Deep expertise in specific domain',
  tools: [specializedDatabaseTool]
});

// Use tool() helper to wrap agent calls
const consultSpecialistTool = tool({
  description: 'Get specialist input on complex questions',
  parameters: z.object({
    question: z.string().describe('The question to ask the specialist')
  }),
  execute: async ({ question }) => {
    const result = await agents.run({
      agent: specialist,
      message: question
    });
    return { answer: result.output };
  }
});

const generalist = new Agent({
  name: 'generalist',
  instructions: 'Handle general queries, delegate when needed',
  tools: [consultSpecialistTool]
});
```

## Memory and State

### Persistent Conversation Memory

```typescript
import { tool } from '@openai/agents';
import { z } from 'zod';

class AgentMemory {
  private store = new Map<string, any>();

  set(key: string, value: any) {
    this.store.set(key, value);
  }

  get(key: string) {
    return this.store.get(key);
  }

  getAll() {
    return Object.fromEntries(this.store);
  }
}

const memoryTool = tool({
  description: 'Store information for later use in the conversation',
  parameters: z.object({
    key: z.string().describe('The key to store the value under'),
    value: z.string().describe('The value to store (use JSON for complex data)')
  }),
  execute: async ({ key, value }) => {
    // Note: context passing varies by implementation
    // This is a simplified example
    memory.set(key, value);
    return { stored: true, key };
  }
});
```
