# Guardrails Pattern

Guardrails are safety checks that validate agent inputs and outputs. They act as filters to prevent agents from processing harmful content or generating inappropriate responses.

## Types of Guardrails

1. **Input Guardrails** - Validate user input before agent processes it
2. **Output Guardrails** - Validate agent output before returning to user
3. **Streaming Guardrails** - Real-time validation during streaming responses

## Input Guardrails

Protect the agent from harmful or inappropriate inputs:

```typescript
import { Agent, run } from '@openai/agents';
import { z } from 'zod';

async function main() {
  // Create a guardrail agent to detect unwanted content
  const mathHomeworkGuardrail = new Agent({
    name: 'Math Homework Detector',
    instructions: 'Check if the user is asking you to do their math homework.',
    outputType: z.object({
      isMathHomework: z.boolean(),
      reason: z.string(),
    }),
  });

  // Main agent with input guardrail
  const customerServiceAgent = new Agent({
    name: 'Customer Support',
    instructions: 'You are a helpful customer support agent.',
    inputGuardrails: [
      {
        name: 'Math Homework Guardrail',
        execute: async ({ input, context }) => {
          // Run guardrail agent to check input
          const result = await run(mathHomeworkGuardrail, input, { context });

          return {
            tripwireTriggered: result.finalOutput?.isMathHomework ?? false,
            outputInfo: {
              blocked: true,
              reason: result.finalOutput?.reason || 'Math homework detected',
            },
          };
        },
      },
    ],
  });

  // Test inputs
  const inputs = [
    'What is the capital of California?',
    'Can you help me solve for x: 2x + 5 = 11?',
  ];

  for (const input of inputs) {
    try {
      const result = await run(customerServiceAgent, input);
      console.log(`✓ ${input}`);
      console.log(`  Response: ${result.finalOutput}\n`);
    } catch (e) {
      console.log(`✗ ${input}`);
      console.log(`  Blocked: ${e}\n`);
    }
  }
}

main().catch(console.error);
```

## Output Guardrails

Validate and filter agent outputs:

```typescript
import { Agent, run } from '@openai/agents';
import { z } from 'zod';

async function main() {
  // Guardrail to detect sensitive information
  const piiGuardrail = new Agent({
    name: 'PII Detector',
    instructions: 'Detect if the output contains personally identifiable information.',
    outputType: z.object({
      hasPII: z.boolean(),
      detectedTypes: z.array(z.string()),
    }),
  });

  const assistantAgent = new Agent({
    name: 'Assistant',
    instructions: 'You are a helpful assistant.',
    outputGuardrails: [
      {
        name: 'PII Guardrail',
        execute: async ({ agentOutput }) => {
          const result = await run(piiGuardrail, agentOutput);

          return {
            tripwireTriggered: result.finalOutput?.hasPII ?? false,
            outputInfo: {
              original: agentOutput,
              detectedTypes: result.finalOutput?.detectedTypes || [],
            },
          };
        },
      },
    ],
  });

  try {
    const result = await run(assistantAgent, 'My phone number is 650-123-4567');
    console.log(result.finalOutput);
  } catch (e) {
    console.log('Output blocked due to PII detection');
  }
}

main().catch(console.error);
```

## Multiple Guardrails

Chain multiple guardrails for comprehensive protection:

```typescript
const agent = new Agent({
  name: 'Secure Assistant',
  instructions: 'You are a secure assistant.',
  inputGuardrails: [
    {
      name: 'Toxic Content Filter',
      execute: async ({ input }) => {
        const isToxic = await checkToxicContent(input);
        return {
          tripwireTriggered: isToxic,
          outputInfo: { reason: 'Toxic content detected' },
        };
      },
    },
    {
      name: 'PII Input Filter',
      execute: async ({ input }) => {
        const hasPII = await detectPII(input);
        return {
          tripwireTriggered: hasPII,
          outputInfo: { reason: 'PII in input' },
        };
      },
    },
  ],
  outputGuardrails: [
    {
      name: 'Code Injection Guardrail',
      execute: async ({ agentOutput }) => {
        const hasInjection = await detectCodeInjection(agentOutput);
        return {
          tripwireTriggered: hasInjection,
          outputInfo: { reason: 'Code injection detected' },
        };
      },
    },
  ],
});
```

## Streaming Guardrails

Real-time validation during streaming:

```typescript
import { Agent, run } from '@openai/agents';

const agent = new Agent({
  name: 'Streaming Agent',
  instructions: 'You provide detailed responses.',
  outputGuardrails: [
    {
      name: 'Streaming Content Filter',
      execute: async ({ agentOutput }) => {
        // Check output as it streams
        const hasProhibitedContent = await checkContent(agentOutput);
        return {
          tripwireTriggered: hasProhibitedContent,
          outputInfo: { reason: 'Prohibited content in stream' },
        };
      },
    },
  ],
});

async function main() {
  const result = await run(agent, 'Tell me a long story', { stream: true });

  try {
    for await (const chunk of result.toTextStream()) {
      process.stdout.write(chunk);
    }
  } catch (e) {
    console.log('\n\nStream interrupted by guardrail');
  }
}
```

## Guardrail with Custom Response

Provide alternative responses when guardrails trigger:

```typescript
const agent = new Agent({
  name: 'Polite Assistant',
  instructions: 'You are a helpful assistant.',
  inputGuardrails: [
    {
      name: 'Politeness Filter',
      execute: async ({ input }) => {
        const isRude = await detectRudeness(input);

        return {
          tripwireTriggered: isRude,
          outputInfo: {
            fallbackMessage: 'I appreciate your feedback. Could you please rephrase that more politely?',
          },
        };
      },
    },
  ],
});
```

## Guardrail Use Cases

| Use Case | Guardrail Type | Example |
|----------|----------------|---------|
| Content Moderation | Input | Block toxic, hate speech |
| PII Protection | Input/Output | Detect and redact sensitive data |
| Compliance | Input/Output | Ensure regulatory compliance |
| Security | Input | Detect code injection, SQL injection |
| Brand Safety | Output | Ensure on-brand responses |
| Rate Limiting | Input | Prevent abuse |
| Fact Checking | Output | Verify claims before returning |
| Language Policy | Input/Output | Enforce language requirements |

## Best Practices

1. **Specific Guardrails**: Create focused guardrails for specific concerns
2. **Performance**: Guardrails should be fast (avoid complex chains)
3. **Clear Messaging**: Explain why content was blocked
4. **Logging**: Log all guardrail triggers for monitoring
5. **Testing**: Test guardrails thoroughly with edge cases
6. **Fallbacks**: Provide helpful fallback messages
7. **Minimal False Positives**: Tune to avoid blocking legitimate content

## Implementing Guardrail Checks

```typescript
// Example guardrail implementation
async function checkToxicContent(text: string): Promise<boolean> {
  const toxicPatterns = [
    /hate|violence|threat/i,
    // Add more patterns
  ];

  return toxicPatterns.some(pattern => pattern.test(text));
}

async function detectPII(text: string): Promise<boolean> {
  const piiPatterns = [
    /\d{3}-\d{2}-\d{4}/, // SSN
    /\d{3}-\d{3}-\d{4}/, // Phone
    /\b[\w.-]+@[\w.-]+\.\w{2,}\b/, // Email
  ];

  return piiPatterns.some(pattern => pattern.test(text));
}

async function detectCodeInjection(text: string): Promise<boolean> {
  const injectionPatterns = [
    /<script>/i,
    /javascript:/i,
    /on\w+\s*=/i, // onclick=, onload=, etc.
  ];

  return injectionPatterns.some(pattern => pattern.test(text));
}
```

## Key Takeaways

- Input guardrails validate user input before agent processing
- Output guardrails validate agent responses before returning
- Multiple guardrails can be chained for comprehensive protection
- Guardrails throw exceptions when triggered (catch them appropriately)
- Guardrails should be fast and specific
- Use structured outputs for guardrail agent results
- Always test guardrails with diverse inputs
