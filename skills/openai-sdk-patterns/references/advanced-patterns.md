# Advanced OpenAI Agent Patterns

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
const validateAndExecute = {
  name: 'validate_and_execute',
  description: 'Validate input then execute action',
  parameters: z.object({
    action: z.enum(['create', 'update', 'delete']),
    data: z.record(z.any())
  }),
  execute: async ({ action, data }) => {
    // Validation step
    const validation = await validateSchema(data);
    if (!validation.valid) {
      return { error: validation.errors };
    }

    // Execution step
    switch (action) {
      case 'create':
        return await createRecord(data);
      case 'update':
        return await updateRecord(data);
      case 'delete':
        return await deleteRecord(data.id);
    }
  }
};
```

## Error Handling

### Graceful Tool Failures

```typescript
const robustTool = {
  name: 'robust_operation',
  description: 'Operation with comprehensive error handling',
  parameters: z.object({ input: z.string() }),
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
};

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
const specialist = new Agent({
  name: 'specialist',
  instructions: 'Deep expertise in specific domain',
  tools: [specializedDatabaseTool]
});

const generalist = new Agent({
  name: 'generalist',
  instructions: 'Handle general queries, delegate when needed',
  tools: [
    // Use agent as a tool
    {
      name: 'consult_specialist',
      description: 'Get specialist input',
      parameters: z.object({
        question: z.string()
      }),
      execute: async ({ question }) => {
        const result = await agents.run({
          agent: specialist,
          message: question
        });
        return result.output;
      }
    }
  ]
});
```

## Memory and State

### Persistent Conversation Memory

```typescript
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

const memoryTool = {
  name: 'remember',
  description: 'Store information for later use',
  parameters: z.object({
    key: z.string(),
    value: z.any()
  }),
  execute: async ({ key, value }, context) => {
    const memory = context.memory as AgentMemory;
    memory.set(key, value);
    return { stored: true, key };
  }
};
```
