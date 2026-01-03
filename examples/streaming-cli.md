# Streaming CLI with OpenAI Agents SDK

Complete example of a command-line interface using the OpenAI Agents SDK with streaming support, session management, and input guardrails.

## Key Features Demonstrated

- **Real-time streaming**: Display agent responses as tokens arrive
- **Session management**: Maintain conversation context across multiple messages
- **Input guardrails**: Validate user input before processing
- **Agent hooks**: Monitor agent execution lifecycle
- **Multi-agent coordination**: Track handoffs between specialist agents

## Implementation

```typescript
// src/index.ts
import { run, MemorySession } from '@openai/agents';
import { mainAgent } from './agents/main-agent.js';
import { safetyGuardrail, qualityGuardrail } from './guardrails/index.js';
import { agentHooks } from './hooks/index.js';
import { initializeSDK, config } from './config/index.js';
import readline from 'readline';

// Initialize SDK configuration
initializeSDK();

/**
 * Run in single-message mode with streaming
 */
async function runSingleMessage(message: string): Promise<void> {
  console.log(`\n🔍 Research Assistant: "${message}"\n`);

  try {
    // Run the agent with streaming enabled
    const streamResult = await run(mainAgent, message, {
      stream: true,
      maxTurns: config.maxTurns,
      agentHooks,
      inputGuardrails: [safetyGuardrail, qualityGuardrail]
    });

    console.log('');
    separator();

    // Process stream events
    let currentAgent = mainAgent.name;
    let finalOutput = '';

    for await (const event of streamResult) {
      switch (event.type) {
        case 'agent_updated_stream_event':
          currentAgent = event.agent.name;
          if (currentAgent !== mainAgent.name) {
            console.log('');
            console.log(`[${currentAgent}]`);
          }
          break;

        case 'raw_model_stream_event':
          // Stream tokens in real-time
          // The text is in event.data.delta as a string when type is 'output_text_delta'
          if (event.data?.type === 'output_text_delta' && typeof event.data.delta === 'string') {
            process.stdout.write(event.data.delta);
            finalOutput += event.data.delta;
          }
          break;

        case 'run_tool_call_stream_event':
          console.log('');
          console.log(`[tool: ${event.toolName}]`);
          break;

        case 'run_handoff_stream_event':
          console.log('');
          console.log(`[handoff: ${event.currentAgent.name} → ${event.targetAgent.name}]`);
          break;

        case 'input_guardrail_tripwire_triggered_stream_event':
          const tripwire = event.tripwireTriggeredData;
          if ('responseData' in tripwire) {
            const response = tripwire.responseData as { refusalReason?: string };
            console.error(`\n❌ ${response.refusalReason || 'Request blocked by guardrail'}\n`);
            return;
          }
          break;
      }
    }

    console.log('');
    separator();
    console.log('\n✓ Done\n');

  } catch (error) {
    console.error('\n❌ Error:', error);
    process.exit(1);
  }
}

/**
 * Run in interactive mode with session management
 */
async function runInteractiveMode(): Promise<void> {
  console.log('\n🔍 Research Assistant - Interactive Mode');
  console.log('Type your message and press Enter. Type "quit" or "exit" to stop.\n');

  // Create a memory session for conversation context
  const session = new MemorySession();

  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
  });

  let messageCount = 0;

  const askQuestion = (): void => {
    rl.question('❯ ', async (input) => {
      const message = input.trim();

      // Check for exit commands
      if (message.toLowerCase() === 'quit' || message.toLowerCase() === 'exit' || message === '') {
        console.log(`\n👋 Session ended. Total messages: ${messageCount}\n`);
        rl.close();
        return;
      }

      messageCount++;
      console.log('');

      try {
        // Run with session context and streaming
        const streamResult = await run(mainAgent, message, {
          session,
          stream: true,
          maxTurns: config.maxTurns,
          agentHooks,
          inputGuardrails: [safetyGuardrail, qualityGuardrail]
        });

        separator();
        let currentAgent = mainAgent.name;

        for await (const event of streamResult) {
          switch (event.type) {
            case 'agent_updated_stream_event':
              currentAgent = event.agent.name;
              if (currentAgent !== mainAgent.name) {
                console.log('');
                console.log(`[${currentAgent}]`);
              }
              break;

            case 'raw_model_stream_event':
              if (event.data?.type === 'output_text_delta' && typeof event.data.delta === 'string') {
                process.stdout.write(event.data.delta);
              }
              break;

            case 'run_handoff_stream_event':
              console.log('');
              console.log(`[handoff: ${event.currentAgent.name} → ${event.targetAgent.name}]`);
              break;

            case 'input_guardrail_tripwire_triggered_stream_event':
              const tripwire = event.tripwireTriggeredData;
              if ('responseData' in tripwire) {
                const response = tripwire.responseData as { refusalReason?: string };
                console.error(`\n❌ ${response.refusalReason || 'Request blocked'}\n`);
                console.log('');
                askQuestion();
                return;
              }
              break;
          }
        }

        console.log('');
        separator();
        console.log('');

      } catch (error) {
        console.error('❌ Error:', error);
      }

      // Continue with next question
      askQuestion();
    });
  };

  // Start the interactive loop
  askQuestion();
}

// Entry point
async function main() {
  const args = process.argv.slice(2);

  // No arguments = interactive mode
  if (args.length === 0) {
    await runInteractiveMode();
    return;
  }

  // Single message mode
  const message = args.join(' ');

  if (message === '--help' || message === '-h') {
    console.log(`
Data Research Assistant

Usage:
  npm start                    Interactive mode (with session memory)
  npm start "your message"     Single message mode

Examples:
  npm start "Search for recent AI research papers"
  npm start "Analyze the data in data.csv"
`);
    return;
  }

  await runSingleMessage(message);
}

main().catch(console.error);
```

## Supporting Files

### Configuration Module

```typescript
// src/config/index.ts
import { setDefaultOpenAIKey } from '@openai/agents';
import 'dotenv/config';

export const config = {
  openaiApiKey: process.env.OPENAI_API_KEY || '',
  model: process.env.OPENAI_MODEL || 'gpt-4o-mini',
  maxTurns: parseInt(process.env.MAX_TURNS || '10', 10),
  enableTracing: process.env.ENABLE_TRACING === 'true',
};

export function validateConfig(): void {
  if (!config.openaiApiKey) {
    throw new Error('OPENAI_API_KEY is required');
  }
}

export function initializeSDK(): void {
  validateConfig();
  setDefaultOpenAIKey(config.openaiApiKey);
}
```

### Input Guardrails

```typescript
// src/guardrails/index.ts
import { InputGuardrail } from '@openai/agents';
import { z } from 'zod';

export const safetyGuardrail: InputGuardrail<
  z.ZodObject<{ message: z.ZodString }>,
  {}
> = {
  name: 'safety_check',

  schema: z.object({
    message: z.string()
  }),

  validate: async ({ message }) => {
    const trimmedMessage = message.trim().toLowerCase();

    // Check for harmful patterns
    const harmfulPatterns = [
      /eval\s*\(/i,
      /exec\s*\(/i,
      /\.\.\//, // path traversal
    ];

    for (const pattern of harmfulPatterns) {
      if (pattern.test(message)) {
        return {
          tripwired: true,
          responseData: {
            refusalReason: 'Request blocked: potentially harmful pattern detected',
            allowed: false
          }
        };
      }
    }

    return { tripwired: false };
  }
};

export const qualityGuardrail: InputGuardrail<
  z.ZodObject<{ message: z.ZodString }>,
  {}
> = {
  name: 'quality_check',

  schema: z.object({
    message: z.string()
  }),

  validate: async ({ message }) => {
    if (!message.trim()) {
      return {
        tripwired: true,
        responseData: {
          refusalReason: 'Message is empty. Please provide a meaningful request.',
          allowed: false
        }
      };
    }

    return { tripwired: false };
  }
};
```

### Agent Hooks

```typescript
// src/hooks/index.ts
import { AgentHooks } from '@openai/agents';

export const agentHooks: AgentHooks = {
  async beforeAgentTurn({ agent, turn }) {
    console.log(`Starting turn ${turn} for agent "${agent.name}"`);
  },

  async afterAgentTurn({ agent, turn }) {
    console.log(`Completed turn ${turn} for agent "${agent.name}"`);
  },

  async beforeToolCall({ agent, toolName }) {
    console.log(`Executing tool "${toolName}"`);
  },

  async afterToolCall({ agent, toolName, duration }) {
    if (duration && duration > 5000) {
      console.warn(`Tool "${toolName}" took ${duration}ms`);
    }
  },

  async onError({ agent, error, turn }) {
    console.error(`Error in agent "${agent.name}" at turn ${turn}:`, error);
  }
};
```

## Event Reference

### Stream Event Types

| Event Type | Description | Key Properties |
|------------|-------------|----------------|
| `agent_updated_stream_event` | Agent changed (handoff) | `event.agent.name` |
| `raw_model_stream_event` | Model response chunk | `event.data.type`, `event.data.delta` |
| `run_handoff_stream_event` | Agent handoff occurred | `event.currentAgent`, `event.targetAgent` |
| `run_tool_call_stream_event` | Tool was called | `event.toolName` |
| `input_guardrail_tripwire_triggered_stream_event` | Guardrail blocked input | `event.tripwireTriggeredData` |

### Raw Model Stream Event Data Types

| `event.data.type` | Description | `event.data.delta` |
|-------------------|-------------|-------------------|
| `response_started` | Response generation started | - |
| `model` | Model event metadata | - |
| `output_text_delta` | Text token delta | `string` (the actual text) |
| `output_done` | Response completed | - |

## Common Pitfalls

### 1. Wrong Event Type Name

```typescript
// ❌ Incorrect
case 'run_raw_model_stream_event':

// ✅ Correct
case 'raw_model_stream_event':
```

### 2. Wrong Delta Access Path

```typescript
// ❌ Incorrect
if (event.delta.type === 'text') {
  process.stdout.write(event.delta.text);
}

// ✅ Correct
if (event.data?.type === 'output_text_delta' && typeof event.data.delta === 'string') {
  process.stdout.write(event.data.delta);
}
```

### 3. Missing Type Checks

```typescript
// ❌ Risky - no type check
process.stdout.write(event.data.delta);

// ✅ Safe - check both type and typeof
if (event.data?.type === 'output_text_delta' && typeof event.data.delta === 'string') {
  process.stdout.write(event.data.delta);
}
```

## Environment Variables

```bash
# Required
OPENAI_API_KEY=sk-...

# Optional
OPENAI_MODEL=gpt-4o-mini
MAX_TURNS=10
ENABLE_TRACING=false
```

## Running the Example

```bash
# Interactive mode with session memory
npm start

# Single message
npm start "What is the capital of France?"

# Show help
npm start -- --help
```

## See Also

- [Advanced Patterns](./advanced-patterns.md) - For more complex streaming scenarios
- [Guardrails](./guardrails.md) - For detailed guardrail implementation
- [Agent Orchestration](./multi-agent-with-handoffs.md) - For handoff patterns
