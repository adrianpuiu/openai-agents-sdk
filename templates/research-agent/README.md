# Research Agent Template

A template for creating research and data analysis agents using OpenAI Agents SDK.

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
import { Agent } from 'openai-agents-sdk';
import { searchWeb, summarizeText, extractData } from '../tools';

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
import { z } from 'zod';

export const searchWeb = {
  name: 'search_web',
  description: 'Search the web for information on a given topic',
  parameters: z.object({
    query: z.string().describe('Search query'),
    numResults: z.number().optional().describe('Number of results to return')
  }),
  execute: async ({ query, numResults = 5 }) => {
    // TODO: Implement web search
    // Options: Google Search API, Bing Search API, or custom crawler

    return {
      results: [],
      query,
      count: 0
    };
  }
};
```

### Summarize Text Tool

```typescript
// src/tools/summarizeText.ts
import { z } from 'zod';

export const summarizeText = {
  name: 'summarize_text',
  description: 'Summarize a long text into key points',
  parameters: z.object({
    text: z.string().describe('Text to summarize'),
    maxLength: z.number().optional().describe('Maximum summary length')
  }),
  execute: async ({ text, maxLength = 200 }) => {
    // TODO: Implement summarization
    // Options: Use the LLM itself, or a dedicated summarization service

    return {
      summary: '',
      originalLength: text.length,
      summaryLength: 0
    };
  }
};
```

### Extract Data Tool

```typescript
// src/tools/extractData.ts
import { z } from 'zod';

export const extractData = {
  name: 'extract_data',
  description: 'Extract structured data from unstructured text',
  parameters: z.object({
    text: z.string().describe('Text to extract from'),
    schema: z.record(z.string()).describe('Schema defining what to extract')
  }),
  execute: async ({ text, schema }) => {
    // TODO: Implement data extraction
    // Use the LLM to extract structured data based on schema

    return {
      extracted: {},
      confidence: 0
    };
  }
};
```

## Usage

```bash
npm start "Research the latest developments in quantum computing"
```

## Customization

Replace the template variables:
- `{{RESEARCH_DOMAIN}}` - Your area of research
- `{{CUSTOM_INSTRUCTIONS}}` - Specific research guidelines
