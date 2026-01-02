# Workflow Automation Agent Template

A template for creating workflow automation agents using OpenAI Agents SDK.

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
import { Agent } from 'openai-agents-sdk';
import { createRecord, updateRecord, notifyUser } from '../tools';

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
import { z } from 'zod';

export const createRecord = {
  name: 'create_record',
  description: 'Create a new record in the system',
  parameters: z.object({
    recordType: z.string().describe('Type of record to create'),
    data: z.record(z.any()).describe('Record data')
  }),
  execute: async ({ recordType, data }) => {
    // TODO: Connect to your database or API
    console.log(`Creating ${recordType} record:`, data);

    return {
      success: true,
      recordId: `REC-${Date.now()}`,
      createdAt: new Date().toISOString()
    };
  }
};
```

### Update Record Tool

```typescript
// src/tools/updateRecord.ts
import { z } from 'zod';

export const updateRecord = {
  name: 'update_record',
  description: 'Update an existing record in the system',
  parameters: z.object({
    recordId: z.string().describe('ID of record to update'),
    updates: z.record(z.any()).describe('Fields to update')
  }),
  execute: async ({ recordId, updates }) => {
    // TODO: Connect to your database or API
    console.log(`Updating record ${recordId}:`, updates);

    return {
      success: true,
      recordId,
      updatedAt: new Date().toISOString(),
      fieldsUpdated: Object.keys(updates)
    };
  }
};
```

### Notify User Tool

```typescript
// src/tools/notifyUser.ts
import { z } from 'zod';

export const notifyUser = {
  name: 'notify_user',
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

    return {
      sent: true,
      notificationId: `NOTIF-${Date.now()}`,
      channel,
      recipient,
      sentAt: new Date().toISOString()
    };
  }
};
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
const step1 = await createRecord({ recordType: 'request', data: initialData });
const step2 = await updateRecord({ recordId: step1.recordId, updates: { status: 'pending' } });
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
