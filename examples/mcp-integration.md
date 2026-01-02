# MCP (Model Context Protocol) Integration

The OpenAI Agents SDK for TypeScript supports integration with MCP servers, enabling agents to use tools hosted on external MCP servers. This document shows how to integrate MCP tools with your agents.

## What is MCP?

MCP (Model Context Protocol) is a protocol that allows AI agents to connect to external tools and data sources. MCP servers can host tools that agents can invoke, extending their capabilities.

## MCP Server Types

The OpenAI Agents SDK TypeScript supports three MCP server types:

1. **MCPServerStdio** - Connect to local MCP servers via stdin/stdout
2. **MCPServerStreamableHttp** - Connect to HTTP MCP servers with streaming support
3. **MCPServerSSE** - Connect to SSE (Server-Sent Events) MCP servers

## 1. Streamable HTTP MCP Server

Connect to a hosted MCP server via HTTP with streaming support.

```typescript
// src/mcp/http-mcp-server.ts
import { MCPServerStreamableHttp, Agent, run } from '@openai/agents';

async function main() {
  // Create a Streamable HTTP MCP server
  const mcpServer = new MCPServerStreamableHttp({
    url: 'https://gitmcp.io/openai/codex',
    name: 'GitMCP Documentation Server',
    // Optional: Add authentication headers
    requestInit: {
      headers: {
        'Authorization': `Bearer ${process.env.MCP_API_KEY}`
      }
    }
  });

  // Create agent with MCP server
  const agent = new Agent({
    name: 'GitMCP Assistant',
    instructions: 'Use the tools to respond to user requests.',
    mcpServers: [mcpServer],
  });

  try {
    // Connect to the MCP server
    await mcpServer.connect();

    // Run the agent
    const result = await run(agent, 'Which language is this repo written in?');
    console.log(result.finalOutput);
  } finally {
    // Always close the connection when done
    await mcpServer.close();
  }
}

main().catch(console.error);
```

**Key Points:**
- Always call `connect()` before using the server
- Always call `close()` when done (use try-finally)
- Pass `mcpServers` array directly to the Agent constructor
- Tools from MCP servers are automatically loaded and available to the agent

## 2. SSE (Server-Sent Events) MCP Server

Connect to an SSE-based MCP server for real-time updates.

```typescript
// src/mcp/sse-mcp-server.ts
import { MCPServerSSE, Agent, run } from '@openai/agents';

async function main() {
  // Create an SSE MCP server
  const mcpServer = new MCPServerSSE({
    url: 'https://api.example.com/mcp/sse',
    name: 'Analytics MCP Server',
    // Optional: Add authentication
    requestInit: {
      headers: {
        'Authorization': `Bearer ${process.env.MCP_API_KEY}`
      }
    }
  });

  const agent = new Agent({
    name: 'Analytics Assistant',
    instructions: 'You analyze data using the available MCP tools.',
    mcpServers: [mcpServer],
  });

  try {
    await mcpServer.connect();
    const result = await run(agent, 'Generate a report on the latest data');
    console.log(result.finalOutput);
  } finally {
    await mcpServer.close();
  }
}

main().catch(console.error);
```

## 3. stdio MCP Server (Local)

Connect to a local MCP server via stdin/stdout communication.

```typescript
// src/mcp/stdio-mcp-server.ts
import { MCPServerStdio, Agent, run } from '@openai/agents';

async function main() {
  // Create a stdio MCP server
  const mcpServer = new MCPServerStdio({
    // Use fullCommand or command + args
    fullCommand: 'npx -y @modelcontextprotocol/server-filesystem ./sample_files',
    // Or use command + args separately:
    // command: 'npx',
    // args: ['-y', '@modelcontextprotocol/server-filesystem', './sample_files'],

    name: 'Filesystem MCP Server',
    // Optional: Set working directory
    cwd: process.cwd(),
    // Optional: Set environment variables
    env: {
      ...process.env,
      CUSTOM_VAR: 'value'
    }
  });

  const agent = new Agent({
    name: 'Filesystem Assistant',
    instructions: 'You help users work with files using the available MCP tools.',
    mcpServers: [mcpServer],
  });

  try {
    await mcpServer.connect();
    const result = await run(agent, 'List all files in the current directory');
    console.log(result.finalOutput);
  } finally {
    await mcpServer.close();
  }
}

main().catch(console.error);
```

**stdIO Server Options:**
- `fullCommand`: Complete command string (easier)
- `command` + `args`: Separate command and arguments (more control)
- `env`: Environment variables for the server process
- `cwd`: Working directory for the server process
- `timeout`: Connection timeout in seconds

## 4. Multiple MCP Servers

Use multiple MCP servers in a single agent.

```typescript
// src/mcp/multi-mcp-servers.ts
import { MCPServerStreamableHttp, MCPServerStdio, MCPServerSSE, Agent, run } from '@openai/agents';

async function main() {
  // Create multiple MCP servers
  const gitServer = new MCPServerStreamableHttp({
    url: 'https://gitmcp.io/openai/codex',
    name: 'Git MCP',
  });

  const filesystemServer = new MCPServerStdio({
    fullCommand: 'npx -y @modelcontextprotocol/server-filesystem ./data',
    name: 'Filesystem MCP',
  });

  const analyticsServer = new MCPServerSSE({
    url: 'https://analytics.example.com/mcp',
    name: 'Analytics MCP',
  });

  const agent = new Agent({
    name: 'Multi-Source Assistant',
    instructions: `You can access multiple data sources:
    - Git repositories via Git MCP
    - Local files via Filesystem MCP
    - Analytics data via Analytics MCP`,
    mcpServers: [gitServer, filesystemServer, analyticsServer],
  });

  try {
    // Connect all servers
    await Promise.all([
      gitServer.connect(),
      filesystemServer.connect(),
      analyticsServer.connect(),
    ]);

    const result = await run(agent, 'Analyze the git repo and create a report');
    console.log(result.finalOutput);
  } finally {
    // Close all servers
    await Promise.all([
      gitServer.close(),
      filesystemServer.close(),
      analyticsServer.close(),
    ]);
  }
}

main().catch(console.error);
```

## 5. Tool Filtering

Selectively use tools from MCP servers using static or callable filters.

### Static Tool Filter

```typescript
// src/mcp/static-filter.ts
import { MCPServerStreamableHttp, Agent, run } from '@openai/agents';

async function main() {
  const mcpServer = new MCPServerStreamableHttp({
    url: 'https://api.example.com/mcp',
    name: 'Filtered MCP Server',
    // Use static filter to allow only specific tools
    toolFilter: {
      // Allow only tools with these names
      allowedToolNames: ['get_weather', 'get_forecast'],
      // Or block specific tools
      blockedToolNames: ['delete_data', 'admin_function'],
    },
  });

  const agent = new Agent({
    name: 'Filtered Assistant',
    instructions: 'Use only the allowed tools to help users.',
    mcpServers: [mcpServer],
  });

  try {
    await mcpServer.connect();
    const result = await run(agent, 'What is the weather?');
    console.log(result.finalOutput);
  } finally {
    await mcpServer.close();
  }
}

main().catch(console.error);
```

### Callable Tool Filter (Dynamic)

```typescript
// src/mcp/callable-filter.ts
import { MCPServerStreamableHttp, Agent, run } from '@openai/agents';

async function main() {
  const mcpServer = new MCPServerStreamableHttp({
    url: 'https://api.example.com/mcp',
    name: 'Dynamic Filter MCP Server',
    // Use callable filter for dynamic tool selection
    toolFilter: async ({ runContext, agent, serverName, tool }) => {
      // Only allow 'read_' tools during business hours
      const hour = new Date().getHours();
      const isBusinessHours = hour >= 9 && hour < 17;

      if (tool.name.startsWith('read_')) {
        return true; // Always allow reading
      }

      if (tool.name.startsWith('write_') || tool.name.startsWith('delete_')) {
        return isBusinessHours; // Only allow writing during business hours
      }

      return false; // Block everything else
    },
  });

  const agent = new Agent({
    name: 'Time-Restricted Assistant',
    instructions: 'Help users with available tools based on current time.',
    mcpServers: [mcpServer],
  });

  try {
    await mcpServer.connect();
    const result = await run(agent, 'Update the configuration file');
    console.log(result.finalOutput);
  } finally {
    await mcpServer.close();
  }
}

main().catch(console.error);
```

## 6. MCP Server Configuration Management

```typescript
// src/mcp/config.ts
import { MCPServerStreamableHttp, MCPServerStdio, MCPServerSSE } from '@openai/agents';

interface McpServerConfig {
  type: 'stdio' | 'streamableHttp' | 'sse';
  url?: string;
  fullCommand?: string;
  command?: string;
  args?: string[];
  apiKey?: string;
  name?: string;
  toolFilter?: any;
}

// Configuration for multiple MCP servers
const mcpConfigs: Record<string, McpServerConfig> = {
  git: {
    type: 'streamableHttp',
    url: 'https://gitmcp.io/openai/codex',
    name: 'Git MCP',
  },
  filesystem: {
    type: 'stdio',
    fullCommand: 'npx -y @modelcontextprotocol/server-filesystem ./data',
    name: 'Filesystem MCP',
  },
  analytics: {
    type: 'sse',
    url: 'https://analytics.example.com/mcp',
    apiKey: process.env.ANALYTICS_API_KEY,
    name: 'Analytics MCP',
    toolFilter: {
      allowedToolNames: ['query', 'report'],
    },
  },
};

// Create MCP servers from configuration
export function createMcpServer(config: McpServerConfig) {
  switch (config.type) {
    case 'stdio':
      return new MCPServerStdio({
        fullCommand: config.fullCommand!,
        name: config.name,
      });

    case 'streamableHttp':
      return new MCPServerStreamableHttp({
        url: config.url!,
        name: config.name,
        requestInit: config.apiKey ? {
          headers: {
            'Authorization': `Bearer ${config.apiKey}`,
          },
        } : undefined,
        toolFilter: config.toolFilter,
      });

    case 'sse':
      return new MCPServerSSE({
        url: config.url!,
        name: config.name,
        requestInit: config.apiKey ? {
          headers: {
            'Authorization': `Bearer ${config.apiKey}`,
          },
        } : undefined,
        toolFilter: config.toolFilter,
      });

    default:
      throw new Error(`Unknown MCP type: ${(config as any).type}`);
  }
}

export const mcpServers = Object.entries(mcpConfigs).map(([name, config]) =>
  createMcpServer({ ...config, name: name })
);
```

## 7. Using MCP Servers in an Agent

```typescript
// src/index.ts
import 'dotenv/config';
import { Agent, run } from '@openai/agents';
import { mcpServers } from './mcp/config.js';

async function main() {
  const agent = new Agent({
    name: 'MCP-Powered Assistant',
    instructions: `You are a helpful assistant with access to multiple MCP servers.

You can:
- Access git repositories
- Work with local files
- Query analytics data

Always explain which MCP server you're using before performing actions.`,
    mcpServers: mcpServers,
  });

  // Connect all servers
  for (const server of mcpServers) {
    await server.connect();
  }

  try {
    const result = await run(agent, process.argv[2] || 'Hello!');
    console.log(result.finalOutput);
  } finally {
    // Close all servers
    for (const server of mcpServers) {
      await server.close();
    }
  }
}

main().catch(console.error);
```

## 8. Caching MCP Tools

Improve performance by caching the tool list from MCP servers.

```typescript
// src/mcp/cached-mcp.ts
import { MCPServerStreamableHttp, Agent, run } from '@openai/agents';

async function main() {
  const mcpServer = new MCPServerStreamableHttp({
    url: 'https://api.example.com/mcp',
    name: 'Cached MCP Server',
    // Enable tool list caching
    cacheToolsList: true,
    // Optional: Set custom timeout
    clientSessionTimeoutSeconds: 300,
  });

  const agent = new Agent({
    name: 'Cached Assistant',
    instructions: 'Use the cached MCP tools for faster responses.',
    mcpServers: [mcpServer],
  });

  try {
    await mcpServer.connect();

    // First call fetches and caches tools
    const result1 = await run(agent, 'List available tools');

    // Subsequent calls use cached tools
    const result2 = await run(agent, 'Use a tool');

    console.log(result1.finalOutput, result2.finalOutput);
  } finally {
    await mcpServer.close();
  }
}

main().catch(console.error);
```

## Environment Setup

```bash
# .env.example
OPENAI_API_KEY=sk-your-openai-api-key

# MCP Server URLs (if needed)
GIT_MCP_URL=https://gitmcp.io/openai/codex
ANALYTICS_MCP_URL=https://analytics.example.com/mcp

# MCP API Keys (if needed)
ANALYTICS_MCP_API_KEY=your-analytics-mcp-api-key
```

## Common MCP Use Cases

| Use Case | MCP Server Type | Example |
|----------|----------------|---------|
| Git repository access | StreamableHttp | `gitmcp.io` |
| Local file operations | stdio | `@modelcontextprotocol/server-filesystem` |
| Real-time analytics | SSE | Custom analytics server |
| Database queries | StreamableHttp | Custom database MCP server |
| API integrations | StreamableHttp | Third-party API wrappers |

## Troubleshooting MCP Integration

### Connection Issues

```typescript
// Test MCP server connectivity
async function testMcpConnection(server: MCPServerStreamableHttp) {
  try {
    await server.connect();
    const tools = await server.listTools();
    console.log('MCP server connected. Available tools:', tools.map(t => t.name));
    return true;
  } catch (error) {
    console.error('MCP server connection failed:', error);
    return false;
  } finally {
    await server.close();
  }
}
```

### Tool Execution Failures

Add error handling in agent instructions:

```typescript
const agent = new Agent({
  name: 'Robust Assistant',
  instructions: `If an MCP tool fails:
  1. Explain what went wrong
  2. Suggest alternative approaches
  3. Never retry failed tools more than once`,
  mcpServers: [mcpServer],
});
```

## Best Practices

1. **Always use try-finally**: Ensure MCP servers are properly closed
2. **Cache when appropriate**: Use `cacheToolsList: true` for stable tool lists
3. **Use tool filtering**: Only expose necessary tools from MCP servers
4. **Handle errors gracefully**: Add error handling in agent instructions
5. **Monitor connections**: Log MCP server connection status
6. **Set timeouts**: Use `clientSessionTimeoutSeconds` for long-running operations

## Key Differences from Python SDK

| Feature | Python SDK | TypeScript SDK |
|---------|------------|----------------|
| Server Import | `from openai.agents import mcp` | `import { MCPServerStreamableHttp } from '@openai/agents'` |
| Server Creation | `mcp.createServer()` | `new MCPServerStreamableHttp()` |
| Agent Integration | `mcp_servers=[...]` | `mcpServers=[...]` |
| Tool Loading | Automatic | Automatic |
| Connection | Manual | Manual (`connect()`/`close()`) |

## Key Takeaways

- MCP extends agent capabilities with external tools
- Three server types: stdio, StreamableHttp, SSE
- Pass `mcpServers` array directly to Agent constructor
- Always `connect()` before use and `close()` when done
- Use tool filtering to control which tools are available
- Enable caching for better performance
