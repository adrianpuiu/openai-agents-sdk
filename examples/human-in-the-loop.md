# Human-in-the-Loop Approval Pattern

The Human-in-the-Loop pattern allows agents to request human approval before executing certain tools or actions. This is essential for:
- Sensitive operations (deleting data, making purchases)
- High-stakes decisions (financial transactions)
- Safety-critical actions
- Compliance and audit trails

## Basic Approval Pattern

```typescript
import { Agent, run, tool } from '@openai/agents';
import { z } from 'zod';
import readline from 'node:readline/promises';

// Tool with approval requirement
const deleteFileTool = tool({
  name: 'delete_file',
  description: 'Delete a file from the filesystem',
  parameters: z.object({
    path: z.string().describe('File path to delete'),
  }),
  // Require approval for this tool
  needsApproval: async (_ctx, { path }) => {
    // Only require approval for certain paths
    return path.includes('important') || path.includes('/etc/');
  },
  execute: async ({ path }) => {
    // File deletion logic here
    return `Deleted file: ${path}`;
  },
});

const agent = new Agent({
  name: 'File Manager',
  instructions: 'You help manage files. Always get approval before deleting important files.',
  tools: [deleteFileTool],
});

// Approval confirmation function
async function confirm(question: string): Promise<boolean> {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  const answer = await rl.question(`${question} (y/n): `);
  const normalizedAnswer = answer.toLowerCase();
  rl.close();
  return normalizedAnswer === 'y' || normalizedAnswer === 'yes';
}

// Main execution with approval handling
async function main() {
  const result = await run(agent, 'Delete the important_data.txt file');

  // Check for interruptions (approval requests)
  if (result.interruptions && result.interruptions.length > 0) {
    for (const interruption of result.interruptions) {
      const question = `Agent ${interruption.agent.name} wants to use ${interruption.name} with args: ${interruption.arguments}. Approve?`;

      if (await confirm(question)) {
        // Approve the tool execution
        result.state.approve(interruption);
      } else {
        // Reject the tool execution
        result.state.reject(interruption);
      }
    }

    // Resume execution after handling approvals
    const finalResult = await run(agent, result.state);
    console.log(finalResult.finalOutput);
  } else {
    console.log(result.finalOutput);
  }
}

main().catch(console.error);
```

## Agent-as-Tool with Approval

You can also require approval when using agents as tools:

```typescript
import { Agent, run, tool } from '@openai/agents';
import { z } from 'zod';

const specialistAgent = new Agent({
  name: 'Specialist',
  instructions: 'You handle specialized tasks.',
  handoffDescription: 'Use for specialized work',
});

const mainAgent = new Agent({
  name: 'Manager',
  instructions: 'You coordinate work and delegate to specialists.',
  tools: [
    // Expose agent as a tool with approval
    specialistAgent.asTool({
      toolName: 'consult_specialist',
      toolDescription: 'Consult the specialist agent',
      // Require approval when input mentions sensitive topics
      needsApproval: async (_ctx, { input }) => {
        return input.toLowerCase().includes('confidential') ||
               input.toLowerCase().includes('secret');
      },
    }),
  ],
});
```

## Persistent Approval State

For long-running approval workflows, you can save and restore the agent state:

```typescript
import fs from 'node:fs/promises';
import { RunState } from '@openai/agents';

async function main() {
  let result = await run(agent, 'Perform sensitive operation');

  // Save state for later review
  await fs.writeFile('state.json', JSON.stringify(result.state), 'utf-8');

  // Later, restore and continue
  const storedState = await fs.readFile('state.json', 'utf-8');
  const state = await RunState.fromString(agent, storedState);

  // Handle approvals and resume
  for (const interruption of result.interruptions) {
    const approved = await getApprovalFromHuman(interruption);
    if (approved) {
      state.approve(interruption);
    } else {
      state.reject(interruption);
    }
  }

  // Resume with updated state
  const finalResult = await run(agent, state);
  console.log(finalResult.finalOutput);
}
```

## Approval Use Cases

| Use Case | Approval Trigger | Example |
|----------|-----------------|---------|
| File Operations | Sensitive paths | Deleting `/etc/` files |
| Financial Transactions | Amount threshold | Transfers over $10,000 |
| Data Deletion | Record count | Deleting >100 records |
| API Calls | Rate limiting | External API calls |
| Agent Handoffs | Context sensitivity | Sharing confidential data |
| System Changes | Production environments | Changes to prod systems |

## Best Practices

1. **Clear Approval Messages**: Always explain what action will be taken
2. **Context Information**: Include relevant details in approval requests
3. **State Persistence**: Save state for audit trails
4. **Conditional Approval**: Not all actions need approval (use `needsApproval` wisely)
5. **User Experience**: Make approval requests easy to understand
6. **Logging**: Log all approval decisions for compliance

## Key Takeaways

- Use `needsApproval` in tool definitions to require human confirmation
- Check `result.interruptions` for pending approvals
- Use `state.approve()` or `state.reject()` to handle interruptions
- Agent-as-tool can also require approval
- State can be saved/loaded for asynchronous approval workflows
