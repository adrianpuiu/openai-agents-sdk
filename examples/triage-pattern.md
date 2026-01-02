# Triage Pattern Complete Guide

The triage pattern is the most common multi-agent orchestration pattern. A triage agent analyzes user intent and routes to appropriate specialist agents.

## When to Use Triage

```
digraph triage_use_case {
    "Multiple expertise domains?" [shape=diamond];
    "Clear intent categories?" [shape=diamond];
    "Use Triage Pattern" [shape=box];
    "Consider other pattern" [shape=box];

    "Multiple expertise domains?" -> "Clear intent categories?";
    "Clear intent categories?" -> "Use Triage Pattern" [label="yes"];
    "Clear intent categories?" -> "Consider other pattern" [label="no"];
}
```

**Use triage when:**
- Multiple domains of expertise needed
- Clear separation of concerns
- User requests can be categorized
- Specialists have distinct tools

**Don't use when:**
- Single agent can handle all requests
- Intent is ambiguous or requires reasoning
- Specialists overlap significantly

## Implementation Pattern

### Step 1: Define Specialists First

Specialists must be defined before the triage agent can reference them.

```typescript
// src/agents/specialists.ts
import { Agent } from 'openai-agents-sdk';

export const salesSpecialist = new Agent({
  name: 'sales',
  instructions: 'You are a sales specialist...',
  tools: [productCatalog, pricingTool]
});

export const supportSpecialist = new Agent({
  name: 'support',
  instructions: 'You are a technical support specialist...',
  tools: [knowledgeBase, diagnosticTool]
});

export const billingSpecialist = new Agent({
  name: 'billing',
  instructions: 'You are a billing specialist...',
  tools: [invoiceLookup, paymentProcessor]
});
```

### Step 2: Create Triage with Handoffs

The triage agent's handoff descriptions are CRITICAL - they tell the agent WHEN to use each handoff.

```typescript
// src/agents/triage.ts
import { Agent, handoff } from 'openai-agents-sdk';
import { salesSpecialist, supportSpecialist, billingSpecialist } from './specialists';

export const triageAgent = new Agent({
  name: 'triage',
  instructions: `You are a triage agent. Analyze the user's request and route to the appropriate specialist.

Routing criteria:
- Sales: Product questions, pricing, features, comparisons, recommendations
- Support: Technical issues, bugs, errors, troubleshooting, integrations
- Billing: Invoices, payments, subscriptions, refunds, account changes

Be friendly and explain you're connecting them with the right specialist.`,

  handoffs: [
    handoff({
      to: salesSpecialist,
      description: 'Use when user asks about products, pricing, features, comparisons, or needs recommendations'
    }),
    handoff({
      to: supportSpecialist,
      description: 'Use when user reports technical issues, bugs, errors, or needs troubleshooting help'
    }),
    handoff({
      to: billingSpecialist,
      description: 'Use when user asks about invoices, payments, subscriptions, refunds, or account changes'
    })
  ]
});
```

### Step 3: Run Through Triage

```typescript
// src/index.ts
import { agents } from 'openai-agents-sdk';
import { triageAgent } from './agents/triage';

async function main() {
  const userMessage = process.argv[2] || 'Hello!';

  const response = await agents.run({
    agent: triageAgent,
    message: userMessage
  });

  console.log(response.output);
}

main().catch(console.error);
```

## Handoff Description Best Practices

The handoff description is the MOST IMPORTANT part. It tells the triage agent exactly when to transfer.

### Bad Examples

```typescript
// ❌ Too vague
handoff({
  to: salesAgent,
  description: 'Transfer to sales'
})

// ❌ Doesn't specify when
handoff({
  to: supportAgent,
  description: 'For technical stuff'
})

// ❌ Overlapping criteria
handoff({
  to: billingAgent,
  description: 'For questions about the account'  // Could be support too!
})
```

### Good Examples

```typescript
// ✅ Specific and clear
handoff({
  to: salesAgent,
  description: 'Use when user asks about products, pricing, features, product comparisons, or needs purchase recommendations'
})

// ✅ Clear triggering conditions
handoff({
  to: supportAgent,
  description: 'Use when user reports technical issues, bugs, error messages, or needs help troubleshooting or setting up integrations'
})

// ✅ Non-overlapping scope
handoff({
  to: billingAgent,
  description: 'Use when user asks about invoices, payments, subscriptions, refunds, or wants to make account changes'
})
```

## Best Practices

### 1. No Circular Handoffs

Specialists should NOT handoff back to triage. This creates routing loops.

```typescript
// ❌ BAD: Circular handoff
const triageAgent = new Agent({
  handoffs: [handoff({ to: specialist, ... })]
});

const specialistAgent = new Agent({
  handoffs: [handoff({ to: triageAgent, ... })]  // Creates loop!
});

// ✅ GOOD: One-way flow
const triageAgent = new Agent({
  handoffs: [handoff({ to: specialist, ... })]
});

const specialistAgent = new Agent({
  handoffs: []  // No handoff back to triage
});
```

### 2. Clear Specialist Roles

Each specialist should have a distinct domain.

```typescript
// ✅ GOOD: Distinct domains
- Sales: Products, pricing, purchasing
- Support: Technical issues, troubleshooting
- Billing: Invoices, payments, refunds

// ❌ BAD: Overlapping domains
- Sales: Customer questions
- Support: Helping customers
- Billing: Customer service
```

### 3. Specialist Tools Match Domain

Give specialists tools appropriate to their domain.

```typescript
// ✅ GOOD: Domain-aligned tools
salesAgent.tools = [productCatalog, pricingCalculator, compareProducts]
supportAgent.tools = [knowledgeBase, diagnosticTool, ticketSystem]
billingAgent.tools = [invoiceLookup, paymentProcessor, refundTool]

// ❌ BAD: Misaligned tools
salesAgent.tools = [ticketSystem, diagnosticTool]  // These are support tools!
```

### 4. Test Each Routing Path

Verify triage works for each specialist.

```bash
# Test sales routing
npm start "How much does the premium plan cost?"

# Test support routing
npm start "I'm getting a 500 error when I log in"

# Test billing routing
npm start "I need a refund for last month"

# Test ambiguous cases
npm start "I can't access my account"  # Which agent should handle?
```

## Advanced: Conditional Handoffs

Handoffs based on runtime conditions.

```typescript
const priorityHandoff = handoff({
  to: prioritySupportAgent,
  description: 'Escalate to priority support for urgent issues',
  shouldHandoff: async (context) => {
    // Only escalate if user is premium customer OR issue is critical
    return context.userTier === 'premium' || context.severity === 'critical';
  }
});
```

## Advanced: Handoff with Context

Pass additional context to the specialist.

```typescript
const contextualHandoff = handoff({
  to: specialistAgent,
  description: 'Transfer with full context',
  context: (sourceContext) => ({
    // Conversation history
    history: sourceContext.conversationHistory,

    // What triage determined
    detectedIntent: sourceContext.intent,

    // User information
    userId: sourceContext.userId,
    userTier: sourceContext.userTier,

    // Escalation metadata
    escalatedAt: new Date().toISOString(),
    escalationReason: sourceContext.reason
  })
});
```

## Common Patterns

### 3-Level Triage

For complex systems, use multiple triage levels.

```
Level 1 Triage (broad categories)
    ├── Level 2 Triage (sub-categories)
    │   ├── Specialist A
    │   └── Specialist B
    └── Level 2 Triage (sub-categories)
        ├── Specialist C
        └── Specialist D
```

### Fallback Pattern

Triage with a fallback for ambiguous requests.

```typescript
const triageAgent = new Agent({
  name: 'triage',
  instructions: `Route to specialists when intent is clear.
If intent is ambiguous or request is general, handle it yourself.`,
  handoffs: [
    handoff({ to: specialistA, description: '...' }),
    handoff({ to: specialistB, description: '...' })
  ]
  // Triage can also handle simple queries directly
});
```

## Key Takeaways

1. **Specialists first** - Define them before triage
2. **Handoff descriptions** are critical - be specific about WHEN to use
3. **No circular handoffs** - prevents routing loops
4. **Clear domains** - Each specialist has distinct expertise
5. **Tools match domain** - Give specialists appropriate tools
6. **Test each path** - Verify routing works correctly
