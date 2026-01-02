# Built-in Tools

The OpenAI Agents SDK includes several built-in tools that you can use directly in your agents without implementing them yourself.

## Available Built-in Tools

| Tool | Description | Use Case |
|------|-------------|----------|
| `webSearchTool` | Search the web for current information | News, facts, current events |
| `fileSearchTool` | Search and retrieve document contents | Document analysis, RAG |
| `computerTool` | Automate browser interactions | Web scraping, automation |

## Web Search Tool

Search the web for up-to-date information:

```typescript
import { Agent, run, webSearchTool } from '@openai/agents';

async function main() {
  const agent = new Agent({
    name: 'Research Assistant',
    instructions: 'You search the web and summarize findings.',
    tools: [
      webSearchTool({
        // Optional: Specify user location for localized results
        userLocation: {
          type: 'approximate',
          city: 'San Francisco',
        },
        // Optional: Add search context
        searchContext: 'technology news',
      }),
    ],
  });

  const result = await run(
    agent,
    "Search for the latest AI news and give me a summary"
  );

  console.log(result.finalOutput);
}

main().catch(console.error);
```

### Web Search with Location

```typescript
const agent = new Agent({
  name: 'Local Business Finder',
  instructions: 'You find local businesses and services.',
  tools: [
    webSearchTool({
      userLocation: {
        type: 'approximate',
        city: 'New York',
        region: 'NY',
        country: 'US',
      },
    }),
  ],
});
```

### Web Search Options

```typescript
webSearchTool({
  // User location for localized results
  userLocation: {
    type: 'approximate' | 'precise',
    city?: string,
    region?: string,
    country?: string,
    coordinates?: {
      latitude: number,
      longitude: number,
    },
  },

  // Search context/domain
  searchContext?: string,
});
```

## File Search Tool

Search through documents using vector search:

```typescript
import { Agent, run, fileSearchTool } from '@openai/agents';

async function main() {
  const agent = new Agent({
    name: 'Document Analyst',
    instructions: 'You search through documents and answer questions.',
    tools: [
      fileSearchTool({
        // Vector store ID from OpenAI
        vectorStoreIds: ['vs_abc123'],
        // Maximum number of results to return
        maxNumResults: 5,
      }),
    ],
  });

  const result = await run(
    agent,
    'What does the document say about our Q3 revenue?'
  );

  console.log(result.finalOutput);
}

main().catch(console.error);
```

### File Search with Multiple Stores

```typescript
const agent = new Agent({
  name: 'Knowledge Base Search',
  instructions: 'You search through our knowledge base.',
  tools: [
    fileSearchTool({
      vectorStoreIds: [
        'vs_hr_documents',
        'vs_policies',
        'vs_procedures',
      ],
      maxNumResults: 10,
    }),
  ],
});
```

### File Search Options

```typescript
fileSearchTool({
  // Vector store IDs to search
  vectorStoreIds: string[],

  // Maximum number of results
  maxNumResults?: number,
});
```

## Computer Use Tool (Browser Automation)

Automate web browser interactions using Playwright:

```typescript
import { Agent, run, computerTool } from '@openai/agents';

async function main() {
  const agent = new Agent({
    name: 'Web Browser',
    instructions: 'You browse the web and find information.',
    tools: [
      computerTool({
        // Display dimensions
        displaySize: {
          width: 1920,
          height: 1080,
        },
        // Environment (linux, windows, mac)
        environment: 'linux',
      }),
    ],
  });

  const result = await run(
    agent,
    'Go to example.com, find the contact page, and tell me the email address'
  );

  console.log(result.finalOutput);
}

main().catch(console.error);
```

### Computer Tool Options

```typescript
computerTool({
  // Display size for screenshots
  displaySize: {
    width: number,
    height: number,
  },

  // Operating environment
  environment?: 'linux' | 'windows' | 'mac',
});
```

## Combining Built-in Tools

Use multiple built-in tools together:

```typescript
import { Agent, run, webSearchTool, fileSearchTool, computerTool } from '@openai/agents';

const researcherAgent = new Agent({
  name: 'Comprehensive Researcher',
  instructions: `You are a research assistant with multiple capabilities:
  - Search the web for current information
  - Search through internal documents
  - Browse websites for specific details

  Use the appropriate tool based on the request.`,
  tools: [
    webSearchTool({
      userLocation: { type: 'approximate', city: 'San Francisco' },
    }),
    fileSearchTool({
      vectorStoreIds: ['vs_internal_docs'],
      maxNumResults: 5,
    }),
    computerTool({
      displaySize: { width: 1920, height: 1080 },
    }),
  ],
});

async function main() {
  const queries = [
    'What are the latest trends in AI?',  // Uses web search
    'What do our internal documents say about Q4 planning?',  // Uses file search
    'Go to our competitor\'s site and find their pricing',  // Uses computer tool
  ];

  for (const query of queries) {
    const result = await run(researcherAgent, query);
    console.log(`\nQuery: ${query}`);
    console.log(`Answer: ${result.finalOutput}\n`);
  }
}

main().catch(console.error);
```

## Best Practices

### Web Search
- Provide location for localized results
- Use specific search queries
- Summarize multiple sources

### File Search
- Organize documents into relevant vector stores
- Use appropriate `maxNumResults` (5-10 is usually good)
- Create targeted vector stores per domain

### Computer Tool
- Provide clear navigation instructions
- Allow time for page loads
- Handle dynamic content gracefully

## Use Cases

### Web Search Use Cases
| Scenario | Example |
|----------|---------|
| Current Events | "Latest news about AI" |
| Fact Checking | "Verify this claim" |
| Research | "Information about quantum computing" |
| Comparisons | "Compare iPhone vs Android" |
| Local Info | "Restaurants near me" |

### File Search Use Cases
| Scenario | Example |
|----------|---------|
| RAG (Retrieval-Augmented Generation) | "What does our policy say about..." |
| Document Analysis | "Summarize the contract" |
| Knowledge Base | "How do I handle customer refunds?" |
| Compliance | "What are the safety requirements?" |

### Computer Tool Use Cases
| Scenario | Example |
|----------|---------|
| Web Scraping | "Extract product prices from this site" |
| Form Filling | "Fill out this contact form" |
| Navigation | "Find the contact link on this page" |
| Testing | "Test our signup flow" |
| Automation | "Check if our site is accessible" |

## Tool Selection Guide

```
User Request Type:
├── Current/Real-time Info → webSearchTool
├── Internal Knowledge → fileSearchTool
├── Web Interaction → computerTool
└── Mixed Requirements → Use multiple tools
```

## Example: Multi-Tool Research Workflow

```typescript
import { Agent, run, webSearchTool, fileSearchTool } from '@openai/agents';

async function comprehensiveResearch(topic: string) {
  // Step 1: Web search for current information
  const webResearcher = new Agent({
    name: 'Web Researcher',
    instructions: 'Search the web for current information.',
    tools: [webSearchTool()],
  });

  const webResults = await run(webResearcher, `Current state of ${topic}`);

  // Step 2: Document search for internal context
  const docResearcher = new Agent({
    name: 'Document Researcher',
    instructions: 'Search internal documents for relevant information.',
    tools: [fileSearchTool({ vectorStoreIds: ['vs_docs'] })],
  });

  const docResults = await run(docResearcher, `Internal knowledge about ${topic}`);

  // Step 3: Synthesize findings
  const synthesizer = new Agent({
    name: 'Research Synthesizer',
    instructions: 'Combine external and internal research into a comprehensive report.',
  });

  const report = await run(
    synthesizer,
    `Create a report on ${topic} using:

EXTERNAL RESEARCH:
${webResults.finalOutput}

INTERNAL KNOWLEDGE:
${docResults.finalOutput}

Provide a comprehensive summary with insights.`
  );

  return report.finalOutput;
}

// Usage
comprehensiveResearch('machine learning in healthcare').then(console.log);
```

## Key Takeaways

- Built-in tools save implementation time
- `webSearchTool` for current web information
- `fileSearchTool` for document/knowledge base search
- `computerTool` for browser automation
- Can combine multiple tools in one agent
- Each tool has specific configuration options
- Choose the right tool based on the use case
