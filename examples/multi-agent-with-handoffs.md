# Multi-Agent with Handoffs Example

A complete example of a multi-agent system using the triage pattern with specialist handoffs.

## Use Case

This example demonstrates:
- Triage agent that routes to specialists
- Specialist agents with domain-specific tools
- Handoff definitions with clear descriptions
- Context preservation across handoffs

## Architecture

```
User Request
    |
    v
Triage Agent (routes by intent)
    |
    +--> Sales Agent (product questions)
    +--> Support Agent (technical issues)
    +--> Billing Agent (payment issues)
```

## Complete Implementation

```typescript
// src/agents/triage.ts
import { Agent, handoff } from 'openai-agents-sdk';
import { salesAgent } from './sales';
import { supportAgent } from './support';
import { billingAgent } from './billing';

export const triageAgent = new Agent({
  name: 'triage',
  instructions: `You are a triage agent. Your job is to understand the user's request and route them to the appropriate specialist.

Analyze the request and handoff to:
- salesAgent: Product questions, pricing, features, comparisons
- supportAgent: Technical issues, bugs, troubleshooting, integrations
- billingAgent: Invoices, payments, subscriptions, refunds

Be friendly and explain you're connecting them with the right specialist.`,
  handoffs: [
    handoff({
      to: salesAgent,
      description: 'Use when user asks about products, pricing, features, or product comparisons'
    }),
    handoff({
      to: supportAgent,
      description: 'Use when user reports technical issues, bugs, or needs troubleshooting help'
    }),
    handoff({
      to: billingAgent,
      description: 'Use when user asks about invoices, payments, subscriptions, or needs a refund'
    })
  ]
});
```

```typescript
// src/agents/sales.ts
import { Agent } from 'openai-agents-sdk';
import { productCatalog, compareProducts } from '../tools/sales';

export const salesAgent = new Agent({
  name: 'sales',
  instructions: `You are a sales specialist helping customers with product information.

Your tools can:
- Look up products in the catalog
- Compare products side by side

Be helpful and knowledgeable. Ask questions to understand their needs.`,
  tools: [productCatalog, compareProducts]
});
```

```typescript
// src/agents/support.ts
import { Agent } from 'openai-agents-sdk';
import { searchKnowledgeBase, createTicket } from '../tools/support';

export const supportAgent = new Agent({
  name: 'support',
  instructions: `You are a technical support specialist.

Your tools can:
- Search the knowledge base for solutions
- Create support tickets for complex issues

Try to help resolve issues immediately. Create tickets only for complex problems.`,
  tools: [searchKnowledgeBase, createTicket]
});
```

```typescript
// src/agents/billing.ts
import { Agent } from 'openai-agents-sdk';
import { lookupInvoice, processPayment } from '../tools/billing';

export const billingAgent = new Agent({
  name: 'billing',
  instructions: `You are a billing specialist.

Your tools can:
- Look up invoices by ID
- Process payments and refunds

Always verify account details before processing.`,
  tools: [lookupInvoice, processPayment]
});
```

## Entry Point

```typescript
// src/index.ts
import { agents } from 'openai-agents-sdk';
import { triageAgent } from './agents/triage';

async function main() {
  const message = process.argv[2] || 'Hello! How can I help you today?';

  const response = await agents.run({
    agent: triageAgent,
    message
  });

  console.log(response.output);
}

main().catch(console.error);
```

## Handoff Best Practices Used

1. **Clear descriptions** that tell the triage agent WHEN to use each handoff
2. **Specialized agents** with domain-specific tools
3. **Context preservation** - handoffs maintain conversation history
4. **No circular handoffs** - specialists don't handoff back to triage

## Testing Handoffs

```bash
# Test sales routing
npm start "How much does the premium plan cost?"

# Test support routing
npm start "I'm getting a 500 error when I log in"

# Test billing routing
npm start "I need a refund for last month"
```

## Key Takeaways

1. **Triage pattern** is the most common multi-agent architecture
2. **Handoff descriptions** are critical - they tell the agent WHEN to transfer
3. **Specialist agents** have focused tools for their domain
4. **Context flows** automatically between agents via handoffs
5. **No circular handoffs** - prevents routing loops
