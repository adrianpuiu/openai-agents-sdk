# Parallelization Pattern

Running multiple agents in parallel can significantly improve performance and enable use cases like:
- Getting multiple perspectives on the same input
- Redundancy and reliability
- Comparing different approaches
- Aggregating results from multiple sources

## Basic Parallel Execution

Run multiple agents simultaneously and collect all results:

```typescript
import { Agent, run } from '@openai/agents';

async function main() {
  // Create multiple translator agents
  const translator1 = new Agent({
    name: 'Translator 1',
    instructions: 'You translate English to Spanish. Be creative with your translation.',
  });

  const translator2 = new Agent({
    name: 'Translator 2',
    instructions: 'You translate English to Spanish. Be formal and precise.',
  });

  const translator3 = new Agent({
    name: 'Translator 3',
    instructions: 'You translate English to Spanish. Use colloquial language.',
  });

  const message = 'Hello, how are you today?';

  // Run all translators in parallel
  const [result1, result2, result3] = await Promise.all([
    run(translator1, message),
    run(translator2, message),
    run(translator3, message),
  ]);

  console.log('Translation 1:', result1.finalOutput);
  console.log('Translation 2:', result2.finalOutput);
  console.log('Translation 3:', result3.finalOutput);
}

main().catch(console.error);
```

## Parallel with Aggregation

Run agents in parallel and use another agent to pick the best result:

```typescript
import { Agent, run, extractAllTextOutput } from '@openai/agents';

async function main() {
  const spanishAgent = new Agent({
    name: 'Spanish Translator',
    instructions: 'You translate English to Spanish.',
  });

  const pickerAgent = new Agent({
    name: 'Translation Picker',
    instructions: 'You pick the best Spanish translation from the given options.',
  });

  const message = 'The quick brown fox jumps over the lazy dog.';

  // Run multiple translations in parallel
  const [res1, res2, res3] = await Promise.all([
    run(spanishAgent, message),
    run(spanishAgent, message),
    run(spanishAgent, message),
  ]);

  // Extract outputs
  const outputs = [
    extractAllTextOutput(res1.newItems),
    extractAllTextOutput(res2.newItems),
    extractAllTextOutput(res3.newItems),
  ];

  console.log('All translations:');
  outputs.forEach((output, i) => {
    console.log(`${i + 1}. ${output}`);
  });

  // Use picker agent to choose best translation
  const translations = outputs.join('\n\n');
  const bestResult = await run(
    pickerAgent,
    `Input: ${message}\n\nTranslations:\n${translations}\n\nWhich is the best translation? Respond with only the translation.`,
  );

  console.log('\nBest translation:', bestResult.finalOutput);
}

main().catch(console.error);
```

## Parallel Multi-Agent Orchestration

Coordinate multiple specialized agents working in parallel:

```typescript
import { Agent, run } from '@openai/agents';

async function main() {
  // Specialized research agents
  const newsResearcher = new Agent({
    name: 'News Researcher',
    instructions: 'You search for and summarize recent news articles on the given topic.',
    tools: [webSearchTool()],
  });

  const academicResearcher = new Agent({
    name: 'Academic Researcher',
    instructions: 'You search for and summarize academic papers on the given topic.',
    tools: [academicSearchTool()],
  });

  const socialAnalyst = new Agent({
    name: 'Social Analyst',
    instructions: 'You analyze social media sentiment on the given topic.',
    tools: [socialMediaTool()],
  });

  const topic = 'artificial intelligence in healthcare';

  // Run all research in parallel
  const [newsResults, academicResults, socialResults] = await Promise.all([
    run(newsResearcher, topic),
    run(academicResearcher, topic),
    run(socialAnalyst, topic),
  ]);

  // Synthesize results
  const synthesizer = new Agent({
    name: 'Research Synthesizer',
    instructions: 'You synthesize research from multiple sources into a comprehensive report.',
  });

  const synthesis = await run(
    synthesizer,
    `Create a comprehensive report on "${topic}" using these research findings:

NEWS RESEARCH:
${newsResults.finalOutput}

ACADEMIC RESEARCH:
${academicResults.finalOutput}

SOCIAL SENTIMENT:
${socialResults.finalOutput}

Provide a well-structured synthesis with key findings, trends, and insights.`,
  );

  console.log(synthesis.finalOutput);
}

main().catch(console.error);
```

## Parallel with Error Handling

Handle failures gracefully when running agents in parallel:

```typescript
import { Agent, run } from '@openai/agents';

async function runWithRetry<T>(
  agent: Agent<any, any>,
  input: string,
  maxRetries = 3
): Promise<string | null> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const result = await run(agent, input);
      return result.finalOutput as string;
    } catch (error) {
      console.error(`Attempt ${attempt} failed for ${agent.name}:`, error);
      if (attempt === maxRetries) {
        return null;
      }
      // Exponential backoff
      await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempt) * 1000));
    }
  }
  return null;
}

async function main() {
  const agents = [
    new Agent({ name: 'Agent 1', instructions: 'You are agent 1.' }),
    new Agent({ name: 'Agent 2', instructions: 'You are agent 2.' }),
    new Agent({ name: 'Agent 3', instructions: 'You are agent 3.' }),
  ];

  const input = 'Process this request';

  // Run all with retry logic
  const results = await Promise.all(
    agents.map(agent => runWithRetry(agent, input))
  );

  // Filter out failures
  const successfulResults = results.filter(r => r !== null);

  console.log(`Successful: ${successfulResults.length}/${results.length}`);
  successfulResults.forEach((result, i) => {
    console.log(`Result ${i + 1}:`, result);
  });
}

main().catch(console.error);
```

## Parallel Batch Processing

Process multiple inputs in parallel:

```typescript
import { Agent, run } from '@openai/agents';

async function main() {
  const summarizer = new Agent({
    name: 'Summarizer',
    instructions: 'You create concise summaries of the given text.',
  });

  const articles = [
    'Long article text 1...',
    'Long article text 2...',
    'Long article text 3...',
    'Long article text 4...',
    'Long article text 5...',
  ];

  // Process all articles in parallel (batch processing)
  const summaries = await Promise.all(
    articles.map(article => run(summarizer, `Summarize: ${article}`))
  );

  summaries.forEach((summary, i) => {
    console.log(`Summary ${i + 1}:`, summary.finalOutput);
  });
}

main().catch(console.error);
```

## Parallel with Rate Limiting

Control concurrency to avoid rate limits:

```typescript
import { Agent, run } from '@openai/agents';

async function main() {
  const agent = new Agent({
    name: 'Processor',
    instructions: 'You process requests.',
  });

  const inputs = Array.from({ length: 100 }, (_, i) => `Request ${i + 1}`);

  // Process with concurrency limit of 5
  const concurrency = 5;
  const results: string[] = [];

  for (let i = 0; i < inputs.length; i += concurrency) {
    const batch = inputs.slice(i, i + concurrency);
    const batchResults = await Promise.all(
      batch.map(input => run(agent, input))
    );
    results.push(...batchResults.map(r => r.finalOutput as string));
    console.log(`Processed ${Math.min(i + concurrency, inputs.length)}/${inputs.length}`);
  }

  console.log('All results:', results);
}

main().catch(console.error);
```

## Parallel Comparison

Compare different approaches to the same problem:

```typescript
import { Agent, run } from '@openai/agents';

async function main() {
  const problem = 'Design a user authentication system';

  // Different architectural approaches
  const securityFocusedAgent = new Agent({
    name: 'Security Expert',
    instructions: 'You design systems with maximum security in mind.',
  });

  const uxFocusedAgent = new Agent({
    name: 'UX Expert',
    instructions: 'You design systems with optimal user experience.',
  });

  const performanceFocusedAgent = new Agent({
    name: 'Performance Expert',
    instructions: 'You design systems for maximum performance.',
  });

  // Get all perspectives in parallel
  const [securityDesign, uxDesign, performanceDesign] = await Promise.all([
    run(securityFocusedAgent, problem),
    run(uxFocusedAgent, problem),
    run(performanceFocusedAgent, problem),
  ]);

  // Synthesize balanced approach
  const architect = new Agent({
    name: 'System Architect',
    instructions: 'You create balanced system designs considering all perspectives.',
  });

  const balancedDesign = await run(
    architect,
    `Create a balanced authentication system design considering:

SECURITY PERSPECTIVE:
${securityDesign.finalOutput}

UX PERSPECTIVE:
${uxDesign.finalOutput}

PERFORMANCE PERSPECTIVE:
${performanceDesign.finalOutput}

Provide a design that balances all concerns.`
  );

  console.log(balancedDesign.finalOutput);
}

main().catch(console.error);
```

## Parallel Use Cases

| Use Case | Description | Example |
|----------|-------------|---------|
| Redundancy | Run same task multiple times for reliability | Multiple translations |
| Diversity | Get different perspectives | Different expert opinions |
| Aggregation | Combine results from multiple sources | Research synthesis |
| Batch Processing | Process multiple items simultaneously | Summarizing articles |
| Comparison | Compare different approaches | Architectural decisions |
| Speed | Reduce total execution time | Parallel API calls |

## Best Practices

1. **Error Handling**: Always handle failures in parallel execution
2. **Rate Limiting**: Control concurrency to avoid overwhelming APIs
3. **Resource Management**: Be mindful of memory/CPU usage
4. **Result Aggregation**: Have a strategy for combining results
5. **Timeouts**: Set timeouts for individual agent runs
6. **Idempotency**: Ensure operations can be safely retried
7. **Monitoring**: Track performance and success rates

## Key Takeaways

- Use `Promise.all()` for parallel execution
- Parallel execution can significantly improve performance
- Handle errors gracefully when running in parallel
- Use aggregation agents to combine parallel results
- Control concurrency to avoid rate limits
- Parallel execution is great for redundancy, diversity, and batch processing
