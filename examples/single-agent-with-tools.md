# Single Agent with Tools Example

A complete example of a single OpenAI agent with multiple tools for customer data lookup.

## Use Case

This example demonstrates:
- Single agent definition with clear instructions
- Multiple tools with Zod validation schemas
- Proper error handling in tool implementations
- Complete project setup with configuration files

## Agent Definition

```typescript
// src/agents/customer-support.ts
import { Agent } from 'openai-agents-sdk';
import { lookupOrder, processReturn, checkInventory } from '../tools';

export const customerSupportAgent = new Agent({
  name: 'customer-support',
  instructions: `You are a helpful customer support agent.

Your responsibilities:
- Help customers look up order information
- Process return requests
- Check product inventory availability
- Provide friendly, professional service

Always:
- Be polite and empathetic
- Ask for order ID when needed
- Explain actions clearly
- Offer additional help`,
  tools: [
    lookupOrder,
    processReturn,
    checkInventory
  ]
});
```

## Tools

```typescript
// src/tools/lookupOrder.ts
import { z } from 'zod';

export const lookupOrder = {
  name: 'lookup_order',
  description: 'Look up a customer order by order ID and retrieve order details including status, items, and tracking information',
  parameters: z.object({
    orderId: z.string().min(1).describe('The order ID to look up (e.g., ORD-12345)')
  }),
  execute: async ({ orderId }) => {
    // TODO: Connect to your order database
    console.log(`Looking up order: ${orderId}`);

    return {
      orderId,
      status: 'shipped',
      items: [
        { productId: 'PROD-001', name: 'Widget', quantity: 2, price: 29.99 }
      ],
      trackingNumber: '1Z999AA10123456784',
      orderDate: '2024-01-15',
      total: 59.98
    };
  }
};
```

```typescript
// src/tools/processReturn.ts
import { z } from 'zod';

export const processReturn = {
  name: 'process_return',
  description: 'Process a return request for an order, create return label, and update order status',
  parameters: z.object({
    orderId: z.string().describe('The order ID to process return for'),
    reason: z.string().describe('Reason for return'),
    items: z.array(z.string()).describe('List of product IDs to return')
  }),
  execute: async ({ orderId, reason, items }) => {
    // TODO: Connect to returns system
    console.log(`Processing return for order ${orderId}:`, { reason, items });

    return {
      success: true,
      returnLabel: 'RETURN-12345',
      refundAmount: 59.98,
      refundDate: '2024-01-20',
      instructions: 'Package items and attach return label. Drop off at any location.'
    };
  }
};
```

```typescript
// src/tools/checkInventory.ts
import { z } from 'zod';

export const checkInventory = {
  name: 'check_inventory',
  description: 'Check if a product is in stock and available for immediate shipping',
  parameters: z.object({
    productId: z.string().describe('Product ID to check')
  }),
  execute: async ({ productId }) => {
    // TODO: Connect to inventory system
    console.log(`Checking inventory for: ${productId}`);

    return {
      productId,
      inStock: true,
      quantity: 150,
      location: 'Warehouse West',
      restockDate: null
    };
  }
};
```

## Entry Point

```typescript
// src/index.ts
import { agents } from 'openai-agents-sdk';
import { customerSupportAgent } from './agents/customer-support';

async function main() {
  const message = process.argv[2] || 'Hello! How can I help you today?';

  const response = await agents.run({
    agent: customerSupportAgent,
    message
  });

  console.log(response.output);
}

main().catch(console.error);
```

## Usage

```bash
# Run the agent
npm start "Where is my order ORD-12345?"

# Check type safety
npm run typecheck
```

## Key Takeaways

1. **Tools require all four properties**: name, description, parameters (Zod schema), execute function
2. **Zod schemas provide validation** and documentation for the agent
3. **Tool names use snake_case** for consistency
4. **Clear agent instructions** guide behavior without needing to code every response
5. **Error handling** should be in tool execute functions
