---
description: Create a new OpenAI Agents SDK TypeScript project from natural language description
argument-hint: [agent-description]
---

# OpenAI Agent Creator

You are tasked with helping users create OpenAI Agents SDK applications using natural language descriptions. Your goal is to generate complete, working projects that non-developers can use immediately.

## Process Overview

1. **Gather requirements** (interactive or from argument)
2. **Check latest SDK version** (search for `@openai/agents` npm package)
3. **Generate project structure** (complete with all files)
4. **Create agent code** (using OpenAI Agents SDK)
5. **Generate required tools** (based on agent purpose)
6. **Implement orchestrations** (handoffs if multi-agent)
7. **Verify setup** (auto-run verifier agent)

## Step 1: Gather Requirements

If `$ARGUMENTS` is provided, parse it as the agent description. Otherwise, ask interactively.

**Required information:**
- **Agent description**: What should the agent do? (e.g., "Customer support that can look up orders and process returns")
- **Project name**: Name for the project directory (suggest kebab-case based on description)
- **Agent name**: Short identifier for the agent (e.g., "customer-support")

**Optional but helpful:**
- **Multi-agent?**: Does this need multiple specialized agents with handoffs?
- **Specific tools**: Any specific tools or integrations mentioned?
- **Additional context**: Any other requirements?

**Example conversation:**
```
User: /openai-agent-create "I need a customer support agent that can look up orders, process returns, and escalate complex issues"

You: I'll create a customer support agent for you.

Project name suggestion: customer-support-agent
Agent name: customer-support

This will include:
- Order lookup tool
- Return processing tool
- Escalation handoff to a specialist agent

Does this sound right? [Enter to continue, or describe changes]
```

## Step 2: Check Latest SDK Version

Before creating the project, verify the latest OpenAI Agents SDK version.

Use `mcp__web-search-prime__webSearchPrime` or `WebSearch` to find:
- Latest npm package version for `@openai/agents`
- Current recommended setup patterns

Search query: "@openai/agents npm latest version 2025"

Report the version you'll be installing to the user.

## Step 3: Generate Project Structure

Create the complete project directory structure:

```
project-name/
├── package.json
├── tsconfig.json
├── .env.example
├── .gitignore
├── README.md
├── src/
│   ├── agents/
│   │   ├── main-agent.ts
│   │   └── [specialist-agents if multi-agent]
│   ├── tools/
│   │   ├── [generated-tools].ts
│   └── index.ts
└── tests/
    └── basic.test.ts
```

### package.json Template

```json
{
  "name": "{{project-name}}",
  "version": "0.1.0",
  "type": "module",
  "description": "{{description from user}}",
  "scripts": {
    "start": "node --loader ts-node/esm src/index.ts",
    "typecheck": "tsc --noEmit",
    "test": "echo 'Tests not yet implemented'"
  },
  "dependencies": {
    "@openai/agents": "{{latest-version}}",
    "dotenv": "^17.2.3",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "typescript": "^5.7.0",
    "ts-node": "^10.9.2"
  }
}
```

**CRITICAL**: The package name is `@openai/agents`, NOT `openai-agents-sdk`.

### tsconfig.json Template

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### .env.example

```
OPENAI_API_KEY=sk-your-api-key-here
```

### .gitignore

```
node_modules/
dist/
.env
.env.local
*.log
.DS_Store
```

## Step 4: Create Agent Code

Generate the main agent based on the user's description.

### Main Agent Template

```typescript
// src/agents/main-agent.ts
import { Agent } from '@openai/agents';
import { {{toolImports}} } from '../tools/{{toolFiles}}';

export const {{agentName}}Agent = new Agent({
  name: '{{agentName}}',
  instructions: `You are a helpful {{agentType}} agent.

{{detailedInstructionsBasedOnDescription}}`,

  tools: [
    {{toolsArray}}
  ]
  {{handoffsIfMultiAgent}}
});
```

### Tool Generation (CRITICAL PATTERNS)

**IMPORTANT**: Tools must use the `tool()` helper from `@openai/agents`, NOT plain objects.

```typescript
// src/tools/example-tool.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const {{toolName}} = tool({
  description: '{{what the tool does}}',
  parameters: z.object({
    {{parameterName}}: z.{{type}}().describe('{{parameter description}}')
  }),
  execute: async ({{{parameterName}}}) => {
    // TODO: Implement the tool logic
    console.log(`Executing {{toolName}} with:`, {{{parameterName}}});

    // Return a string (required by the SDK)
    return JSON.stringify({
      success: true,
      {{returnValue}}: `{{placeholder response}}`
    }, null, 2);
  }
});
```

**CRITICAL ZOD SCHEMA RULES:**

The OpenAI SDK uses **strict mode** which has specific requirements:

| ❌ DON'T USE | ✅ USE INSTEAD | REASON |
|--------------|----------------|---------|
| `z.optional()` | Required fields or separate boolean flags | Strict mode requires all fields |
| `z.url()` | `z.string()` with description | "uri" format not supported |
| `z.any()` | `z.string()` (for JSON) or specific types | Schema must have a type key |
| `.optional().default()` | Always required, or redesign | No optional fields in strict mode |

**Example of handling optional parameters:**

```typescript
// ❌ WRONG - uses .optional()
export const badTool = tool({
  parameters: z.object({
    query: z.string(),
    maxResults: z.number().optional().default(10)
  })
});

// ✅ CORRECT - make required, or use boolean flag
export const goodTool = tool({
  description: 'Search with specified result count',
  parameters: z.object({
    query: z.string().describe('Search query'),
    maxResults: z.number().describe('Maximum number of results to return (e.g., 10)')
  })
});
```

**Tool Return Value:**
- Tools MUST return a `string` or `Promise<string>`
- Common pattern: `JSON.stringify(result, null, 2)`

**Common tool templates:**

| Capability | Tool Pattern |
|------------|--------------|
| Database lookup | Query by ID, return JSON string |
| API call | HTTP request with parameters |
| Calculation | Compute and return JSON string |
| File operation | Read/write with validation |
| Notification | Send message/log |

## Extended Tool Templates

### Database Query Tool Template

```typescript
// src/tools/{{toolName}}.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const {{toolName}} = tool({
  description: '{{description}}',
  parameters: z.object({
    {{paramName}}: z.{{type}}().describe('{{description}}')
  }),
  execute: async ({{{paramName}}}) => {
    // TODO: Connect to database
    // Example: PostgreSQL, MySQL, MongoDB, etc.

    try {
      // const result = await db.query('SELECT * FROM table WHERE id = $1', [{{{paramName}}}]);

      return JSON.stringify({
        success: true,
        data: {{returnValue}},
        found: true
      }, null, 2);
    } catch (error) {
      console.error('Database error:', error);
      return JSON.stringify({
        success: false,
        error: error instanceof Error ? error.message : 'Unknown error'
      }, null, 2);
    }
  }
});
```

### API Integration Tool Template

```typescript
// src/tools/{{toolName}}.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const {{toolName}} = tool({
  description: '{{description}}',
  parameters: z.object({
    endpoint: z.string().describe('API endpoint to call'),
    method: z.enum(['GET', 'POST', 'PUT', 'DELETE']).describe('HTTP method'),
    {{otherParams}}
  }),
  execute: async ({ endpoint, method, {{otherParamNames}} }) => {
    // TODO: Configure base URL and auth
    const baseUrl = process.env.API_BASE_URL || 'https://api.example.com';
    const apiKey = process.env.API_KEY;

    try {
      const response = await fetch(`${baseUrl}${endpoint}`, {
        method,
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${apiKey}`
        },
        body: method !== 'GET' ? JSON.stringify({{bodyParams}}) : undefined
      });

      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }

      const data = await response.json();

      return JSON.stringify({
        success: true,
        data,
        status: response.status
      }, null, 2);
    } catch (error) {
      console.error('API call error:', error);
      return JSON.stringify({
        success: false,
        error: error instanceof Error ? error.message : 'API call failed'
      }, null, 2);
    }
  }
});
```

### File Processing Tool Template

```typescript
// src/tools/{{toolName}}.ts
import { tool } from '@openai/agents';
import { z } from 'zod';
import { readFile } from 'fs/promises';

export const {{toolName}} = tool({
  description: '{{description}}',
  parameters: z.object({
    filePath: z.string().describe('Path to the file'),
    encoding: z.string().describe('File encoding (e.g., utf-8, ascii)')
  }),
  execute: async ({ filePath, encoding }) => {
    try {
      const content = await readFile(filePath, { encoding: encoding as BufferEncoding });

      return JSON.stringify({
        success: true,
        content,
        size: content.length
      }, null, 2);
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: error instanceof Error ? error.message : 'File operation failed'
      }, null, 2);
    }
  }
});
```

## Tool Selection Guide

| User Requirement | Tool Template | Notes |
|------------------|---------------|-------|
| "Look up user in database" | Database Query - findOne | Query by ID |
| "Search products" | Database Query - findMany | With filters |
| "Call external API" | API Integration | Add auth |
| "Process CSV file" | File Processing | Parse and validate |
| "Send email alert" | Notification | Configure service |
| "Convert JSON to CSV" | Data Transformation | Format conversion |
| "Validate form data" | Validation Tool | Schema-based |

### Multi-Agent Handoffs

If the user needs multi-agent orchestration:

```typescript
// src/agents/specialist-agent.ts
import { Agent } from '@openai/agents';

export const specialistAgent = new Agent({
  name: 'specialist',
  instructions: 'You handle specialized cases that require...'
});

// In main agent, import and add handoff
import { Agent, handoff } from '@openai/agents';
import { specialistAgent } from './specialist-agent';

// Add to main agent config:
export const mainAgent = new Agent({
  name: 'main',
  instructions: '...',
  handoffs: [
    handoff(specialistAgent, {
      toolDescriptionOverride: 'Use for specialized cases that require...'
    })
  ]
});
```

**CRITICAL**: Handoff syntax is `handoff(agent, options)`, NOT `handoff({ to: agent })`.

## Step 5: Create Entry Point

```typescript
// src/index.ts
import 'dotenv/config';
import { run } from '@openai/agents';
import { {{agentName}}Agent } from './agents/main-agent.js';

async function main() {
  const message = process.argv[2] || 'Hello!';

  const result = await run({{agentName}}Agent, message);

  console.log(result.finalOutput);
}

main().catch(console.error);
```

**CRITICAL**:
- Import `dotenv/config` BEFORE importing from the SDK
- Use `run()` function, NOT `agents.run()`
- Access response via `result.finalOutput`

## Step 6: Create README.md

```markdown
# {{Project Name}}

{{Description from user}}

## Setup

1. Install dependencies:
   \`\`\`bash
   npm install
   \`\`\`

2. Set up environment:
   \`\`\`bash
   cp .env.example .env
   # Edit .env and add your OPENAI_API_KEY
   \`\`\`

3. Get your API key from https://platform.openai.com/api-keys

## Running

\`\`\`bash
npm start "Your message here"
\`\`\`

## Agent Capabilities

This agent can:
- {{capability 1}}
- {{capability 2}}
- {{capability 3}}

## Tools

- {{tool1}}: {{description}}
- {{tool2}}: {{description}}

{{multiAgentSection}}

## Development

\`\`\`bash
npm run typecheck  # Check types
\`\`\`
```

## Step 7: Install Dependencies and Verify

After creating all files:

1. **Initialize and install:**
   ```bash
   cd {{project-name}}
   npm install
   ```

2. **Verify TypeScript compilation:**
   ```bash
   npx tsc --noEmit
   ```
   Fix any type errors before proceeding.

3. **Run the verifier agent** to validate the setup:
   - Launch the `openai-agent-verifier` agent
   - It will check SDK usage, patterns, and configuration
   - Address any issues found

## Step 8: Final Summary

Provide the user with:

1. **Project location:** Full path to created project
2. **Next steps:**
   - Add OPENAI_API_KEY to `.env`
   - Run `npm install` (already done if no errors)
   - Run `npm start "test message"` to test
3. **Generated files summary:**
   - X agent files
   - X tool files
   - Configuration files
4. **Customization guidance:**
   - Edit agent instructions in `src/agents/main-agent.ts`
   - Implement tool logic in `src/tools/*.ts`
   - Add more tools as needed

## Important Guidelines

- **Use the correct package name** - `@openai/agents`, NOT `openai-agents-sdk`
- **Use the `tool()` helper** - NOT plain objects for tools
- **All Zod fields must be required** - NO `.optional()` in strict mode
- **Avoid `.url()` validation** - Use `z.string()` with description
- **Avoid `z.any()`** - Use `z.string()` for JSON or specific types
- **Tools return strings** - Return `JSON.stringify(result, null, 2)`
- **Use dotenv** - Import `'dotenv/config'` in index.ts
- **Correct handoff syntax** - `handoff(agent, options)` NOT `handoff({ to: agent })`
- **TypeScript must pass** - Run `npx tsc --noEmit` and fix all errors
- **Generate working code** - Even if tools are placeholders, they must compile and run
- **Helpful comments** - Explain what each part does
- **Non-developer friendly** - Code should be readable and well-documented

## Common User Scenarios

| User Description | Generated Agent |
|------------------|-----------------|
| "Customer support that looks up orders" | Agent with `lookupOrder` tool |
| "Research assistant that searches the web" | Agent with `searchWeb` tool |
| "Data analyst that processes CSV files" | Agent with `processCSV` tool |
| "Multi-agent: triage + specialists" | Triage agent with handoffs to specialists |

---

**Before starting, gather the requirements from the user. Then create the complete project, verify it compiles, and run the verifier agent.**
