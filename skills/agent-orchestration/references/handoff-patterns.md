# Advanced Handoff Patterns

## Conditional Handoffs

Handoffs based on runtime conditions:

```typescript
const conditionalHandoff = handoff({
  to: escalationAgent,
  description: 'Escalate when issue severity is high',
  shouldHandoff: async (context) => {
    // Determine if handoff is appropriate
    return context.severity === 'high' || context.attempts > 3;
  }
});
```

## Handoff Callbacks

Execute code before/after handoff:

```typescript
const handoffWithLogging = handoff({
  to: specialistAgent,
  description: 'Handoff with logging',
  onBeforeHandoff: async (context) => {
    console.log(`Handing off from ${context.currentAgent} to specialist`);
    // Log to analytics
    await analytics.track('handoff_initiated', {
      from: context.currentAgent,
      to: 'specialist',
      reason: context.handoffReason
    });
  },
  onAfterHandoff: async (context, result) => {
    console.log('Handoff completed');
    await analytics.track('handoff_completed', { success: true });
  }
});
```

## Dynamic Handoff Selection

Choose target agent based on context:

```typescript
const dynamicHandoff = {
  name: 'route_to_specialist',
  description: 'Route to appropriate specialist based on issue type',
  parameters: z.object({
    issueType: z.enum(['technical', 'billing', 'account'])
  }),
  execute: async ({ issueType }, context) => {
    const agents = {
      technical: technicalAgent,
      billing: billingAgent,
      account: accountAgent
    };

    const targetAgent = agents[issueType];

    return await agents.run({
      agent: targetAgent,
      context: context
    });
  }
};
```

## Multi-Agent Consensus

Multiple agents collaborate on a decision:

```typescript
const consensusAgent = new Agent({
  name: 'consensus',
  instructions: 'Gather input from multiple specialists and synthesize a response',
  tools: [
    {
      name: 'consult_all_specialists',
      description: 'Get input from all specialists',
      parameters: z.object({
        question: z.string()
      }),
      execute: async ({ question }) => {
        // Query all specialists in parallel
        const results = await Promise.all([
          agents.run({ agent: specialist1, message: question }),
          agents.run({ agent: specialist2, message: question }),
          agents.run({ agent: specialist3, message: question })
        ]);

        return {
          specialist1: results[0].output,
          specialist2: results[1].output,
          specialist3: results[2].output
        };
      }
    }
  ]
});
```

## Handoff with State

Maintain state across handoffs:

```typescript
interface HandoffState {
  attempts: number;
  history: string[];
  currentAgent: string;
}

const statefulHandoff = handoff({
  to: nextAgent,
  description: 'Handoff with state tracking',
  context: (sourceContext) => ({
    state: {
      attempts: (sourceContext.state?.attempts || 0) + 1,
      history: [...(sourceContext.state?.history || []), sourceContext.lastMessage],
      currentAgent: sourceContext.agentName
    }
  })
});
```

## Human-in-the-Loop Handoff

```typescript
const humanHandoff = handoff({
  to: humanAgent,
  description: 'Escalate to human when automation fails',
  context: {
    requiresHuman: true,
    escalationPath: 'supervisor'
  }
});

const humanAgent = new Agent({
  name: 'human',
  instructions: 'You facilitate human intervention. Present the context clearly and wait for human input.',
  tools: [
    {
      name: 'await_human_input',
      description: 'Wait for a human to provide input',
      parameters: z.object({
        prompt: z.string()
      }),
      execute: async ({ prompt }) => {
        // Integration with human notification system
        return await notifyHumanAndWait({ prompt });
      }
    }
  ]
});
```

## Bidirectional Handoffs

Agents can handoff back and forth:

```typescript
const salesToSupport = handoff({
  to: supportAgent,
  description: 'Transfer to support for technical questions'
});

const supportToSales = handoff({
  to: salesAgent,
  description: 'Transfer to sales for pricing questions'
});

const salesAgent = new Agent({
  name: 'sales',
  instructions: 'You handle sales inquiries. Ask technical questions to support.',
  handoffs: [salesToSupport]
});

const supportAgent = new Agent({
  name: 'support',
  instructions: 'You handle technical support. Ask pricing questions to sales.',
  handoffs: [supportToSales]
});
```
