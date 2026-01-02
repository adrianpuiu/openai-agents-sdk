# Advanced Agent Patterns

Complex patterns for production-ready OpenAI agents.

## Patterns Covered

1. Agent as Tool (delegation pattern)
2. Conversation Memory
3. Streaming Responses
4. Error Recovery

## 1. Agent as Tool Pattern

Delegating specialized work to another agent by wrapping it as a tool.

```typescript
// src/agents/analyst.ts
import { Agent } from 'openai-agents-sdk';
import { deepDataQuery } from '../tools';

export const dataAnalystAgent = new Agent({
  name: 'data-analyst',
  instructions: 'You are a data analyst. Query databases and analyze results.',
  tools: [deepDataQuery]
});
```

```typescript
// src/agents/manager.ts
import { agents } from 'openai-agents-sdk';
import { dataAnalystAgent } from './analyst';
import { z } from 'zod';

export const managerAgent = new Agent({
  name: 'manager',
  instructions: 'You coordinate tasks and delegate to specialists when needed.',
  tools: [
    {
      name: 'consult_analyst',
      description: 'Get data analysis from the specialist',
      parameters: z.object({
        query: z.string().describe('The data query to analyze')
      }),
      execute: async ({ query }) => {
        const result = await agents.run({
          agent: dataAnalystAgent,
          message: `Analyze: ${query}`
        });
        return result.output;
      }
    }
  ]
});
```

## 2. Memory Pattern

Remembering information across the conversation.

```typescript
// src/memory/ConversationMemory.ts
export class ConversationMemory {
  private store = new Map<string, unknown>();

  set(key: string, value: unknown): void {
    this.store.set(key, value);
  }

  get(key: string): unknown {
    return this.store.get(key);
  }

  getAll(): Record<string, unknown> {
    return Object.fromEntries(this.store);
  }
}
```

```typescript
// src/tools/remember.ts
import { z } from 'zod';

export const rememberTool = {
  name: 'remember',
  description: 'Store information to remember for later in the conversation',
  parameters: z.object({
    key: z.string().describe('Key to store under'),
    value: z.any().describe('Value to remember')
  }),
  execute: async ({ key, value }, context) => {
    const memory = context.memory as ConversationMemory;
    memory.set(key, value);
    return { remembered: true, key };
  }
};
```

```typescript
// src/index.ts
import { agents } from 'openai-agents-sdk';
import { ConversationMemory } from './memory/ConversationMemory';

async function main() {
  const memory = new ConversationMemory();

  const response = await agents.run({
    agent: myAgent,
    message: 'Remember that my name is Alice',
    context: { memory }
  });

  // Later in conversation
  const response2 = await agents.run({
    agent: myAgent,
    message: 'What is my name?',
    context: { memory }
  });
}
```

## 3. Streaming Pattern

Streaming responses for long-form content.

```typescript
// src/index.ts
import { agents } from 'openai-agents-sdk';

async function main() {
  const response = await agents.run({
    agent: myAgent,
    message: 'Tell me a long story',
    stream: true
  });

  for await (const chunk of response) {
    process.stdout.write(chunk.delta);
  }
}
```

## 4. Error Recovery Pattern

Graceful error handling with retries.

```typescript
// src/tools/resilientTool.ts
import { z } from 'zod';

export const resilientTool = {
  name: 'resilient_operation',
  description: 'Operation with automatic retry on failure',
  parameters: z.object({
    input: z.string()
  }),
  execute: async ({ input }, context) => {
    const maxRetries = 3;
    let attempt = 0;

    while (attempt < maxRetries) {
      try {
        const result = await performOperation(input);
        return { success: true, data: result };
      } catch (error) {
        attempt++;

        if (attempt >= maxRetries) {
          return {
            success: false,
            error: error instanceof Error ? error.message : 'Unknown error',
            attempts: attempt
          };
        }

        // Exponential backoff
        await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempt) * 1000));
      }
    }
  }
};
```

## 5. Parallel Tool Execution

Running multiple tools in parallel when independent.

```typescript
// src/agents/parallelAgent.ts
import { Agent } from 'openai-agents-sdk';
import { z } from 'zod';

export const parallelAgent = new Agent({
  name: 'parallel',
  instructions: 'You can run multiple independent operations in parallel.',
  tools: [
    {
      name: 'parallel_lookup',
      description: 'Look up multiple items in parallel',
      parameters: z.object({
        itemIds: z.array(z.string()).describe('List of item IDs to look up')
      }),
      execute: async ({ itemIds }) => {
        // Run all lookups in parallel
        const results = await Promise.all(
          itemIds.map(id => lookupItemById(id))
        );

        return {
          items: results,
          count: results.length
        };
      }
    }
  ]
});
```

## 6. Context Enrichment

Passing enriched context to handoffs.

```typescript
// src/agents/enrichedHandoff.ts
import { Agent, handoff } from 'openai-agents-sdk';

const specialistAgent = new Agent({
  name: 'specialist',
  instructions: 'You receive enriched context from the triage agent.'
});

const enrichedHandoff = handoff({
  to: specialistAgent,
  description: 'Escalate with full conversation context',
  context: (sourceContext) => ({
    conversationHistory: sourceContext.history,
    userIntent: sourceContext.intent,
    attemptedSolutions: sourceContext.attempts || [],
    escalationReason: sourceContext.reason
  })
});
```

## Key Takeaways

1. **Agent as Tool** enables delegation and specialization
2. **Memory patterns** allow stateful conversations
3. **Streaming** improves UX for long responses
4. **Error recovery** makes agents production-ready
5. **Parallel execution** improves performance
6. **Context enrichment** improves handoff quality
