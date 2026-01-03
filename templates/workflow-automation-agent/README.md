# Workflow Automation Agent Template

A template for creating workflow automation agents using OpenAI Agents SDK (`@openai/agents`).

## Structure

```
workflow-automation-agent/
├── src/
│   ├── agents/
│   │   └── workflow-agent.ts       # Main workflow agent
│   ├── tools/
│   │   ├── createRecord.ts          # Create new records
│   │   ├── updateRecord.ts          # Update existing records
│   │   └── notifyUser.ts            # Send notifications
│   └── index.ts                     # Entry point
├── package.json
├── tsconfig.json
└── README.md
```

## Agent Template

```typescript
// src/agents/workflow-agent.ts
import { Agent } from '@openai/agents';
import { createRecord, updateRecord, notifyUser } from '../tools/index.js';

export const workflowAgent = new Agent({
  name: 'workflow-agent',
  instructions: `You are a workflow automation agent for {{WORKFLOW_TYPE}}.

{{CUSTOM_INSTRUCTIONS}}

Available actions:
- Create new records in the system
- Update existing records with new information
- Send notifications to relevant parties

Workflow rules:
{{WORKFLOW_RULES}}

Always confirm before taking destructive actions like deletions.
Log all actions taken for audit purposes.`,
  tools: [createRecord, updateRecord, notifyUser]
});
```

## Tool Templates

### Create Record Tool

```typescript
// src/tools/createRecord.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const createRecord = tool({
  description: 'Create a new record in the system',
  parameters: z.object({
    recordType: z.string().describe('Type of record to create'),
    data: z.string().describe('JSON string containing record data')
  }),
  execute: async ({ recordType, data }) => {
    // TODO: Connect to your database or API
    console.log(`Creating ${recordType} record:`, data);

    return JSON.stringify({
      success: true,
      recordId: `REC-${Date.now()}`,
      createdAt: new Date().toISOString()
    }, null, 2);
  }
});
```

### Update Record Tool

```typescript
// src/tools/updateRecord.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const updateRecord = tool({
  description: 'Update an existing record in the system',
  parameters: z.object({
    recordId: z.string().describe('ID of record to update'),
    updates: z.string().describe('JSON string containing fields to update')
  }),
  execute: async ({ recordId, updates }) => {
    // TODO: Connect to your database or API
    console.log(`Updating record ${recordId}:`, updates);

    return JSON.stringify({
      success: true,
      recordId,
      updatedAt: new Date().toISOString(),
      fieldsUpdated: []
    }, null, 2);
  }
});
```

### Notify User Tool

```typescript
// src/tools/notifyUser.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const notifyUser = tool({
  description: 'Send a notification to a user via email, Slack, or other channel',
  parameters: z.object({
    recipient: z.string().describe('User ID or email address'),
    channel: z.enum(['email', 'slack', 'sms']).describe('Notification channel'),
    subject: z.string().describe('Notification subject'),
    message: z.string().describe('Notification message')
  }),
  execute: async ({ recipient, channel, subject, message }) => {
    // TODO: Integrate with notification service
    // Options: SendGrid, Slack Webhook, Twilio
    console.log(`Sending ${channel} notification to ${recipient}`);

    return JSON.stringify({
      sent: true,
      notificationId: `NOTIF-${Date.now()}`,
      channel,
      recipient,
      sentAt: new Date().toISOString()
    }, null, 2);
  }
});
```

## Entry Point Template

```typescript
// src/index.ts
import 'dotenv/config';
import { run } from '@openai/agents';
import { workflowAgent } from './agents/workflow-agent.js';

async function main() {
  const message = process.argv.slice(2).join(' ');

  const streamResult = await run(workflowAgent, message, {
    stream: true,
    maxTurns: 10
  });

  for await (const event of streamResult) {
    if (event.type === 'raw_model_stream_event') {
      if (event.data?.type === 'output_text_delta' && typeof event.data.delta === 'string') {
        process.stdout.write(event.data.delta);
      }
    }
  }
}

main().catch(console.error);
```

## Usage

```bash
npm start "Create a new task for John to review the Q4 report"
```

## Customization

Replace the template variables:
- `{{WORKFLOW_TYPE}}` - Type of workflow (e.g., "task management", "approval process")
- `{{CUSTOM_INSTRUCTIONS}}` - Specific workflow guidelines
- `{{WORKFLOW_RULES}}` - Business rules for the workflow

## Common Workflow Patterns

### Approval Workflow
```typescript
// Agent checks for approval before executing
if (requiresApproval(action)) {
  await notifyUser({ recipient: manager, channel: 'email', subject: 'Approval needed', message: actionDetails });
}
```

### Multi-Step Workflow
```typescript
// Agent executes steps in sequence
const step1 = await createRecord({ recordType: 'request', data: JSON.stringify(initialData) });
const step2 = await updateRecord({ recordId: step1.recordId, updates: JSON.stringify({ status: 'pending' }) });
const step3 = await notifyUser({ recipient: requester, channel: 'email', subject: 'Request received', message: 'Your request is pending approval' });
```

### Conditional Workflow
```typescript
// Agent branches based on conditions
if (data.priority === 'urgent') {
  await notifyUser({ recipient: escalationTeam, channel: 'slack', subject: 'Urgent request', message: data.summary });
} else {
  await notifyUser({ recipient: regularTeam, channel: 'email', subject: 'New request', message: data.summary });
}
```

## Key Patterns

- Use `tool()` helper for all tools (not plain objects)
- Import from `@openai/agents` (not `openai-agents-sdk`)
- Return `JSON.stringify(result, null, 2)` from tools
- Use `.js` extensions for imports in ES modules
- All Zod parameters must be required (no `.optional()`, no `z.any()`)
- Use `z.string()` for JSON data instead of `z.record(z.any())`
