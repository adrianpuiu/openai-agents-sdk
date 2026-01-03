# Research Agent Template

A template for creating research and data analysis agents using OpenAI Agents SDK (`@openai/agents`).

## Structure

```
research-agent/
├── src/
│   ├── agents/
│   │   └── research-agent.ts    # Main research agent
│   ├── tools/
│   │   ├── searchWeb.ts          # Web search tool
│   │   ├── summarizeText.ts      # Text summarization
│   │   └── extractData.ts        # Data extraction
│   └── index.ts                  # Entry point
├── package.json
├── tsconfig.json
└── README.md
```

## Agent Template

```typescript
// src/agents/research-agent.ts
import { Agent } from '@openai/agents';
import { searchWeb, summarizeText, extractData } from '../tools/index.js';

export const researchAgent = new Agent({
  name: 'research-agent',
  instructions: `You are a research agent specializing in {{RESEARCH_DOMAIN}}.

{{CUSTOM_INSTRUCTIONS}}

Your capabilities:
- Search the web for current information
- Summarize long documents and articles
- Extract structured data from text

Research process:
1. Understand the research question
2. Search for relevant sources
3. Analyze and summarize findings
4. Provide citations when possible

Always verify information from multiple sources when possible.`,
  tools: [searchWeb, summarizeText, extractData]
});
```

## Tool Templates

### Web Search Tool

```typescript
// src/tools/searchWeb.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const searchWeb = tool({
  description: 'Search the web for information on a given topic',
  parameters: z.object({
    query: z.string().describe('Search query')
  }),
  execute: async ({ query }) => {
    // TODO: Implement web search
    // Options: Tavily API, Google Search API, Bing Search API

    return JSON.stringify({
      results: [],
      query,
      count: 0
    }, null, 2);
  }
});
```

### Summarize Text Tool

```typescript
// src/tools/summarizeText.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const summarizeText = tool({
  description: 'Summarize a long text into key points',
  parameters: z.object({
    text: z.string().describe('Text to summarize'),
    maxLength: z.number().describe('Maximum summary length in characters')
  }),
  execute: async ({ text, maxLength }) => {
    // TODO: Implement summarization
    // Options: Use the LLM itself, or a dedicated summarization service

    return JSON.stringify({
      summary: '',
      originalLength: text.length,
      summaryLength: 0
    }, null, 2);
  }
});
```

### Extract Data Tool

```typescript
// src/tools/extractData.ts
import { tool } from '@openai/agents';
import { z } from 'zod';

export const extractData = tool({
  description: 'Extract structured data from unstructured text',
  parameters: z.object({
    text: z.string().describe('Text to extract from'),
    schema: z.string().describe('JSON schema defining what to extract')
  }),
  execute: async ({ text, schema }) => {
    // TODO: Implement data extraction
    // Use the LLM to extract structured data based on schema

    return JSON.stringify({
      extracted: {},
      confidence: 0
    }, null, 2);
  }
});
```

## Entry Point Template

```typescript
// src/index.ts
import 'dotenv/config';
import { run } from '@openai/agents';
import { researchAgent } from './agents/research-agent.js';

async function main() {
  const message = process.argv.slice(2).join(' ');

  const streamResult = await run(researchAgent, message, {
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
npm start "Research the latest developments in quantum computing"
```

## Customization

Replace the template variables:
- `{{RESEARCH_DOMAIN}}` - Your area of research
- `{{CUSTOM_INSTRUCTIONS}}` - Specific research guidelines

## Key Patterns

- Use `tool()` helper for all tools (not plain objects)
- Import from `@openai/agents` (not `openai-agents-sdk`)
- Return `JSON.stringify(result, null, 2)` from tools
- Use `.js` extensions for imports in ES modules
- All Zod parameters must be required (no `.optional()`)
