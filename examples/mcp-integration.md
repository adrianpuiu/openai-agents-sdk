# MCP (Model Context Protocol) Integration

The OpenAI Agents SDK for TypeScript supports integration with MCP servers, enabling agents to use tools hosted on external MCP servers. This document shows how to integrate MCP tools with your agents.

## What is MCP?

MCP (Model Context Protocol) is a protocol that allows AI agents to connect to external tools and data sources. MCP servers can host tools that agents can invoke, extending their capabilities.

## MCP Integration Patterns

The OpenAI Agents SDK TypeScript supports several MCP integration patterns:

1. **Hosted MCP Tools** - Connect to HTTP MCP servers
2. **Streamable HTTP MCP Servers** - Use SSE for streaming responses
3. **stdio MCP Servers** - Connect to local MCP servers via stdio
4. **Tool Filtering** - Selectively use tools from MCP servers

## 1. Hosted MCP Tools (HTTP)

Connect to a hosted MCP server via HTTP.

```typescript
// src/mcp/http-mcp-client.ts
import { mcp } from '@openai/agents';

// Configure HTTP MCP server connection
const httpMcpServer = mcp.createServer({
  type: 'http',
  url: 'https://api.example.com/mcp',
  headers: {
    'Authorization': `Bearer ${process.env.MCP_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

// List available tools from the MCP server
const tools = await httpMcpServer.listTools();
console.log('Available MCP tools:', tools);

// Create a wrapper tool for an MCP tool
import { tool } from '@openai/agents';
import { z } from 'zod';

export const mcpWeatherTool = tool({
  description: 'Get weather information from MCP server',
  parameters: z.object({
    location: z.string().describe('City name or location (e.g., San Francisco, CA)'),
    units: z.enum(['celsius', 'fahrenheit']).describe('Temperature units')
  }),
  execute: async ({ location, units }) => {
    // Call the MCP server
    const result = await httpMcpServer.callTool('get_weather', {
      location,
      units
    });

    return JSON.stringify({
      success: true,
      data: result
    }, null, 2);
  }
});
```

## 2. Streamable HTTP MCP Servers (SSE)

Use Server-Sent Events (SSE) for streaming responses from MCP servers.

```typescript
// src/mcp/sse-mcp-client.ts
import { mcp } from '@openai/agents';

// Configure SSE MCP server connection
const sseMcpServer = mcp.createServer({
  type: 'sse',
  url: 'https://api.example.com/mcp/stream',
  headers: {
    'Authorization': `Bearer ${process.env.MCP_API_KEY}`
  }
});

// Create a streaming tool
import { tool } from '@openai/agents';
import { z } from 'zod';

export const mcpStreamingTool = tool({
  description: 'Stream data analysis from MCP server',
  parameters: z.object({
    dataSource: z.string().describe('Data source identifier'),
    analysisType: z.string().describe('Type of analysis to perform')
  }),
  execute: async ({ dataSource, analysisType }) => {
    const results = [];

    // Stream results from MCP server
    for await (const chunk of sseMcpServer.streamTool('analyze_data', {
      source: dataSource,
      type: analysisType
    })) {
      results.push(chunk);
      process.stdout.write(`Received: ${chunk}\n`);
    }

    return JSON.stringify({
      success: true,
      chunks: results.length,
      finalResult: results[results.length - 1]
    }, null, 2);
  }
});
```

## 3. stdio MCP Servers (Local)

Connect to local MCP servers via stdin/stdout.

```typescript
// src/mcp/stdio-mcp-client.ts
import { mcp } from '@openai/agents';
import { spawn } from 'child_process';

// Start local MCP server process
const mcpProcess = spawn('python', ['mcp_server.py'], {
  cwd: './servers'
});

// Configure stdio MCP server connection
const stdioMcpServer = mcp.createServer({
  type: 'stdio',
  process: mcpProcess
});

// Create a tool using the stdio MCP server
import { tool } from '@openai/agents';
import { z } from 'zod';

export const mcpLocalTool = tool({
  description: 'Process data using local MCP server',
  parameters: z.object({
    data: z.string().describe('Data to process'),
    algorithm: z.string().describe('Processing algorithm to use')
  }),
  execute: async ({ data, algorithm }) => {
    const result = await stdioMcpServer.callTool('process_data', {
      input: data,
      method: algorithm
    });

    return JSON.stringify({
      success: true,
      processed: result
    }, null, 2);
  }
});
```

## 4. Tool Filtering

Selectively use tools from MCP servers.

```typescript
// src/mcp/filtered-mcp.ts
import { mcp } from '@openai/agents';

const mcpServer = mcp.createServer({
  type: 'http',
  url: 'https://api.example.com/mcp'
});

// Filter tools by name prefix
const allowedPrefixes = ['weather_', 'finance_'];

const filteredTools = (await mcpServer.listTools()).filter(tool =>
  allowedPrefixes.some(prefix => tool.name.startsWith(prefix))
);

console.log('Filtered tools:', filteredTools);

// Create wrapper tools only for allowed tools
import { tool } from '@openai/agents';
import { z } from 'zod';

export const createMcpTool = (mcpTool: any) => {
  return tool({
    description: mcpTool.description,
    parameters: z.object({
      // Dynamically create schema from MCP tool definition
      ...Object.fromEntries(
        Object.entries(mcpTool.inputSchema.properties || {}).map(([key, prop]: [string, any]) => [
          key,
          z.string().describe(prop.description || key)
        ])
      )
    }),
    execute: async (params) => {
      const result = await mcpServer.callTool(mcpTool.name, params);
      return JSON.stringify(result, null, 2);
    }
  });
};

// Create tools only for filtered tools
export const mcpTools = filteredTools.map(createMcpTool);
```

## 5. Complete Agent with MCP Integration

```typescript
// src/agents/mcp-agent.ts
import { Agent } from '@openai/agents';
import { mcpWeatherTool, mcpFinanceTool, mcpStreamingTool } from '../mcp';

export const mcpAgent = new Agent({
  name: 'mcp-integrated',
  instructions: `You are an AI assistant with access to external MCP tools.

You can:
- Get weather information from the weather MCP server
- Query financial data from the finance MCP server
- Stream data analysis from the analytics MCP server

Always explain which MCP tool you're using before calling it.`,
  tools: [
    mcpWeatherTool,
    mcpFinanceTool,
    mcpStreamingTool
  ]
});
```

## 6. MCP Server Configuration Management

```typescript
// src/mcp/config.ts
interface McpServerConfig {
  type: 'http' | 'sse' | 'stdio';
  url?: string;
  apiKey?: string;
  command?: string;
  args?: string[];
}

// Configuration for multiple MCP servers
const mcpConfigs: Record<string, McpServerConfig> = {
  weather: {
    type: 'http',
    url: process.env.WEATHER_MCP_URL || 'https://weather.example.com/mcp',
    apiKey: process.env.WEATHER_MCP_API_KEY
  },
  analytics: {
    type: 'sse',
    url: process.env.ANALYTICS_MCP_URL || 'https://analytics.example.com/mcp',
    apiKey: process.env.ANALYTICS_MCP_API_KEY
  },
  local: {
    type: 'stdio',
    command: 'python',
    args: ['local_mcp_server.py']
  }
};

// Create MCP servers from configuration
import { mcp } from '@openai/agents';

export const createMcpServer = (name: string, config: McpServerConfig) => {
  switch (config.type) {
    case 'http':
      return mcp.createServer({
        type: 'http',
        url: config.url!,
        headers: config.apiKey ? {
          'Authorization': `Bearer ${config.apiKey}`
        } : {}
      });

    case 'sse':
      return mcp.createServer({
        type: 'sse',
        url: config.url!,
        headers: config.apiKey ? {
          'Authorization': `Bearer ${config.apiKey}`
        } : {}
      });

    case 'stdio':
      return mcp.createServer({
        type: 'stdio',
        process: spawn(config.command!, config.args!, {
          cwd: './servers'
        })
      });

    default:
      throw new Error(`Unknown MCP type: ${config.type}`);
  }
};

export const mcpServers = Object.fromEntries(
  Object.entries(mcpConfigs).map(([name, config]) => [name, createMcpServer(name, config)])
);
```

## Environment Setup

```bash
# .env.example
# MCP Server URLs
WEATHER_MCP_URL=https://weather.example.com/mcp
ANALYTICS_MCP_URL=https://analytics.example.com/mcp

# MCP API Keys
WEATHER_MCP_API_KEY=your-weather-mcp-api-key
ANALYTICS_MCP_API_KEY=your-analytics-mcp-api-key
```

## Common MCP Use Cases

| Use Case | MCP Server Type | Example |
|----------|----------------|---------|
| External API access | HTTP/SSE | Weather, financial data, news |
| Long-running operations | SSE | Data processing, ETL jobs |
| Local computation | stdio | File operations, local databases |
| Real-time streaming | SSE | Live data feeds, monitoring |

## Troubleshooting MCP Integration

### Connection Issues

```typescript
// Test MCP server connectivity
async function testMcpConnection(server: any) {
  try {
    const tools = await server.listTools();
    console.log('MCP server connected. Available tools:', tools);
    return true;
  } catch (error) {
    console.error('MCP server connection failed:', error);
    return false;
  }
}
```

### Tool Execution Failures

```typescript
// Add error handling to MCP tool calls
execute: async (params) => {
  try {
    const result = await mcpServer.callTool('tool_name', params);
    return JSON.stringify({ success: true, data: result }, null, 2);
  } catch (error) {
    return JSON.stringify({
      success: false,
      error: error instanceof Error ? error.message : 'MCP tool execution failed'
    }, null, 2);
  }
}
```

## Best Practices

1. **Authentication**: Always use API keys for HTTP/SSE MCP servers
2. **Timeout handling**: Implement timeouts for long-running MCP operations
3. **Error handling**: Wrap all MCP calls in try-catch blocks
4. **Tool filtering**: Only expose necessary tools from MCP servers
5. **Monitoring**: Log MCP server calls for debugging
6. **Caching**: Cache frequently-used MCP responses when appropriate

## Key Takeaways

- MCP enables agents to use external tools and services
- HTTP/SSE for remote MCP servers, stdio for local servers
- Always wrap MCP tools with the `tool()` helper
- Filter tools to only expose what's needed
- Implement proper error handling for MCP calls
