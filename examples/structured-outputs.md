# Structured Outputs Pattern

Structured outputs allow agents to return data in a specific format (JSON schema) instead of plain text. This is essential for:
- API integrations
- Data processing pipelines
- Type-safe applications
- Predictable agent responses

## JSON Schema Output Type

Define the exact structure you want the agent to return:

```typescript
import { Agent, run, JsonSchemaDefinition } from '@openai/agents';

// Define the output schema
const WeatherSchema: JsonSchemaDefinition = {
  type: 'json_schema',
  name: 'Weather',
  strict: true,
  schema: {
    type: 'object',
    properties: {
      city: { type: 'string' },
      temperature: { type: 'number' },
      conditions: { type: 'string' },
      humidity: { type: 'number' },
    },
    required: ['city', 'temperature', 'conditions', 'humidity'],
    additionalProperties: false,
  },
};

async function main() {
  const weatherAgent = new Agent({
    name: 'Weather Reporter',
    instructions: 'You provide weather information in a structured format.',
    outputType: WeatherSchema,
  });

  const result = await run(weatherAgent, 'What is the weather in Tokyo?');

  // Result is guaranteed to match the schema
  console.log(result.finalOutput);
  // Output: { city: 'Tokyo', temperature: 22, conditions: 'Sunny', humidity: 60 }
}

main().catch(console.error);
```

## Using Zod for Schema Definition

Use Zod for type-safe schema definitions:

```typescript
import { Agent, run } from '@openai/agents';
import { z } from 'zod';

// Define schema with Zod
const WeatherSchema = z.object({
  city: z.string().describe('City name'),
  temperature: z.number().describe('Temperature in Celsius'),
  conditions: z.string().describe('Weather conditions'),
  humidity: z.number().describe('Humidity percentage'),
  windSpeed: z.number().optional().describe('Wind speed in km/h'),
});

async function main() {
  const weatherAgent = new Agent({
    name: 'Weather Reporter',
    instructions: 'You provide detailed weather information.',
    outputType: WeatherSchema,
  });

  const result = await run(weatherAgent, 'Weather in Paris?');

  // TypeScript knows the type
  const weather = result.finalOutput;
  console.log(`Temperature in ${weather.city}: ${weather.temperature}°C`);
  console.log(`Conditions: ${weather.conditions}`);
  console.log(`Humidity: ${weather.humidity}%`);
}

main().catch(console.error);
```

## Complex Nested Schemas

Define complex nested structures:

```typescript
import { Agent, run } from '@openai/agents';
import { z } from 'zod';

// Define nested schema
const ProductReviewSchema = z.object({
  product: z.object({
    name: z.string(),
    category: z.string(),
    price: z.number(),
  }),
  reviews: z.array(z.object({
    author: z.string(),
    rating: z.number().min(1).max(5),
    comment: z.string(),
    date: z.string(),
  })),
  summary: z.object({
    averageRating: z.number(),
    totalReviews: z.number(),
    sentiment: z.enum(['positive', 'neutral', 'negative']),
  }),
});

async function main() {
  const reviewAgent = new Agent({
    name: 'Review Analyzer',
    instructions: 'You analyze product reviews and extract structured data.',
    outputType: ProductReviewSchema,
  });

  const result = await run(
    reviewAgent,
    'Analyze reviews for "Wireless Bluetooth Headphones" priced at $79.99. Reviews: "Great sound" (5 stars), "Good battery" (4 stars), "Comfortable" (5 stars)'
  );

  console.log('Product:', result.finalOutput.product.name);
  console.log('Average Rating:', result.finalOutput.summary.averageRating);
  console.log('Sentiment:', result.finalOutput.summary.sentiment);
}

main().catch(console.error);
```

## Structured Output with Tools

Combine structured outputs with tool calls:

```typescript
import { Agent, run, tool } from '@openai/agents';
import { z } from 'zod';

const DatabaseSchema = z.object({
  query: z.string(),
  parameters: z.array(z.string()),
  reason: z.string(),
});

const queryPlanner = new Agent({
  name: 'Query Planner',
  instructions: 'You plan database queries based on user requests.',
  outputType: DatabaseSchema,
});

const executeQueryTool = tool({
  name: 'execute_query',
  description: 'Execute a database query',
  parameters: z.object({
    query: z.string(),
    parameters: z.array(z.string()),
  }),
  execute: async ({ query, parameters }) => {
    // Database execution logic
    return `Executed: ${query} with params ${parameters.join(', ')}`;
  },
});

const queryExecutor = new Agent({
  name: 'Query Executor',
  instructions: 'You execute database queries using the available tool.',
  tools: [executeQueryTool],
});

async function main() {
  const userRequest = 'Find all users who signed up this week';

  // Plan the query
  const plan = await run(queryPlanner, userRequest);
  console.log('Query Plan:', plan.finalOutput);

  // Execute the query
  const result = await run(
    queryExecutor,
    `Execute this query: ${plan.finalOutput.query} with params: ${plan.finalOutput.parameters.join(', ')}`
  );

  console.log('Result:', result.finalOutput);
}

main().catch(console.error);
```

## Multiple Structured Outputs

Agents that return different types based on context:

```typescript
import { Agent, run } from '@openai/agents';
import { z } from 'zod';

const SuccessResponse = z.object({
  status: z.literal('success'),
  data: z.string(),
});

const ErrorResponse = z.object({
  status: z.literal('error'),
  error: z.string(),
  code: z.number(),
});

const ResultSchema = z.discriminatedUnion('status', [
  SuccessResponse,
  ErrorResponse,
]);

const processorAgent = new Agent({
  name: 'Processor',
  instructions: 'You process requests and return structured results.',
  outputType: ResultSchema,
});

async function main() {
  const requests = [
    'Process this valid data',
    'Process this invalid data that will cause an error',
  ];

  for (const request of requests) {
    const result = await run(processorAgent, request);

    if (result.finalOutput.status === 'success') {
      console.log('✓ Success:', result.finalOutput.data);
    } else {
      console.log('✗ Error:', result.finalOutput.error, `(Code: ${result.finalOutput.code})`);
    }
  }
}

main().catch(console.error);
```

## Structured Output for Data Extraction

Extract structured data from unstructured text:

```typescript
import { Agent, run } from '@openai/agents';
import { z } from 'zod';

const InvoiceSchema = z.object({
  invoiceNumber: z.string(),
  date: z.string(),
  vendor: z.object({
    name: z.string(),
    address: z.string(),
  }),
  items: z.array(z.object({
    description: z.string(),
    quantity: z.number(),
    unitPrice: z.number(),
    total: z.number(),
  })),
  subtotal: z.number(),
  tax: z.number(),
  total: z.number(),
});

const extractionAgent = new Agent({
  name: 'Invoice Extractor',
  instructions: 'You extract invoice data from text into a structured format.',
  outputType: InvoiceSchema,
});

async function main() {
  const invoiceText = `
    INVOICE #12345
    Date: 2025-01-15

    From: ABC Supplies
    Address: 123 Main St, City, State 12345

    Item              Qty   Price    Total
    ----------------------------------------
    Widget A          5     $10.00   $50.00
    Widget B          3     $25.00   $75.00

    Subtotal:                         $125.00
    Tax (10%):                        $12.50
    Total:                            $137.50
  `;

  const result = await run(extractionAgent, invoiceText);

  console.log('Invoice:', result.finalOutput.invoiceNumber);
  console.log('Vendor:', result.finalOutput.vendor.name);
  console.log('Items:', result.finalOutput.items.length);
  console.log('Total:', result.finalOutput.total);
}

main().catch(console.error);
```

## Structured Output for Validation

Validate content against criteria:

```typescript
import { Agent, run } from '@openai/agents';
import { z } from 'zod';

const ValidationSchema = z.object({
  isValid: z.boolean(),
  score: z.number().min(0).max(100),
  issues: z.array(z.object({
    severity: z.enum(['error', 'warning', 'info']),
    message: z.string(),
    location: z.string().optional(),
  })),
  suggestions: z.array(z.string()),
});

const validatorAgent = new Agent({
  name: 'Content Validator',
  instructions: 'You validate content against quality standards.',
  outputType: ValidationSchema,
});

async function main() {
  const content = 'This is some content that needs validation.';

  const result = await run(validatorAgent, content);

  console.log('Valid:', result.finalOutput.isValid);
  console.log('Score:', result.finalOutput.score);

  result.finalOutput.issues.forEach(issue => {
    console.log(`[${issue.severity.toUpperCase()}] ${issue.message}`);
  });

  if (result.finalOutput.suggestions.length > 0) {
    console.log('Suggestions:');
    result.finalOutput.suggestions.forEach(s => console.log(`  - ${s}`));
  }
}

main().catch(console.error);
```

## Structured Output Use Cases

| Use Case | Description | Example Schema |
|----------|-------------|----------------|
| Data Extraction | Pull structured data from documents | Invoice, resume, form data |
| API Responses | Generate API-compatible responses | REST/GraphQL response types |
| Validation | Check content against criteria | Quality scores, issue lists |
| Classification | Categorize content | Sentiment, topic, intent |
| Configuration | Generate config files | YAML, JSON, TOML |
| Code Generation | Generate structured code | AST, function signatures |
| Reports | Generate structured reports | Metrics, KPIs, analytics |

## Best Practices

1. **Be Specific**: Define clear, specific schemas
2. **Use Descriptions**: Add `.describe()` to Zod fields for better results
3. **Handle Optional Fields**: Use `.optional()` for non-required fields
4. **Validate Schemas**: Test schemas with various inputs
5. **Error Handling**: Handle cases where agent can't match schema
6. **Type Safety**: Use TypeScript to infer types from Zod schemas
7. **Strict Mode**: Use `strict: true` for better schema compliance

## Key Takeaways

- Use `outputType` to define structured output schema
- Zod schemas provide type-safe definitions
- Structured outputs are guaranteed to match the schema
- Combine with tools for powerful workflows
- Essential for integrations and data pipelines
- Use `.describe()` for better agent understanding
