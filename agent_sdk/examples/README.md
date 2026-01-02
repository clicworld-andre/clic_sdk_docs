# Examples & Tutorials

This section provides practical examples and step-by-step tutorials for using the ClicBrain Agent SDK.

## Quick Start Examples

### 1. Create and Register a Simple Agent

```typescript
import { AgentRegistry } from '@/cap/services/agent-registry';
import { CreateAgentInput } from '@/cap/types/agent';

async function createSimpleAgent() {
  const registry = new AgentRegistry();

  const agentConfig: CreateAgentInput = {
    agent_id: 'my-company.assistant.v1',
    name: 'My Assistant Agent',
    version: '1.0.0',
    description: 'A helpful assistant for general queries',
    system: 'clicbrain',
    type: 'llm',
    capabilities: {
      domains: ['global'],
      actions: ['llm_generate', 'query_clicbrain'],
      tools: ['search_knowledge'],
      parallel_tool_calls: true,
    },
    cap_extensions: {
      supports_threads: true,
      supports_interrupts: true,
      supports_streaming: true,
      max_concurrent_runs: 10,
      default_timeout_ms: 300000,
    },
  };

  const agent = await registry.register(agentConfig);
  console.log('Agent registered:', agent.id);
  return agent;
}
```

### 2. Execute a Simple Run

```typescript
import { RunExecutor } from '@/cap/services/run-executor';

async function executeSimpleRun(agentId: string) {
  const executor = new RunExecutor();

  const run = await executor.execute({
    agent_id: agentId,
    input: {
      messages: [
        { role: 'user', content: 'What are the trading hours for BTC?' },
      ],
    },
    options: {
      timeout_ms: 60000,
      stream: false,
    },
  });

  console.log('Run completed:', run.status);
  console.log('Response:', run.output?.response);
  return run;
}
```

### 3. Create a Thread with Conversation History

```typescript
import { ThreadManager } from '@/cap/services/thread-manager';
import { RunExecutor } from '@/cap/services/run-executor';

async function conversationExample(agentId: string) {
  const threadManager = new ThreadManager();
  const executor = new RunExecutor();

  // Create thread
  const thread = await threadManager.create({
    agent_id: agentId,
    metadata: {
      user_id: 'user-123',
      session_id: 'session-456',
    },
    initialMessages: [
      { role: 'system', content: 'You are a helpful trading assistant.' },
    ],
  });

  // First message
  await threadManager.addMessage(thread.id, {
    role: 'user',
    content: 'What is the current BTC price?',
  });

  const run1 = await executor.execute({
    agent_id: agentId,
    thread_id: thread.id,
    input: { messages: [] }, // Uses thread history
  });

  // Follow-up message
  await threadManager.addMessage(thread.id, {
    role: 'user',
    content: 'What about ETH?',
  });

  const run2 = await executor.execute({
    agent_id: agentId,
    thread_id: thread.id,
    input: { messages: [] },
  });

  // Close thread
  await threadManager.close(thread.id, {
    summary: 'User inquired about crypto prices',
    resolution: 'completed',
  });

  return { thread, run1, run2 };
}
```

---

## Domain-Specific Examples

### Trading Agent Example

```typescript
import { createAgentWithTools } from '@/agent-sdk/agent-presets';
import { AgentRegistry } from '@/cap/services/agent-registry';
import { RunExecutor } from '@/cap/services/run-executor';

async function tradingAgentExample() {
  // 1. Setup agent with trading preset
  const { client, executor: toolExecutor, preset } = createAgentWithTools('trading');

  // 2. Register with CAP
  const registry = new AgentRegistry();
  const agent = await registry.register({
    agent_id: 'clix.trading.v1',
    name: 'CLIX Trading Agent',
    version: '1.0.0',
    system: 'clix',
    type: 'trading',
    capabilities: {
      domains: ['trading', 'analytics'],
      actions: ['llm_generate', 'query_clicbrain', 'llm_analyze'],
      tools: preset.recommendedTools,
    },
  });

  // 3. Execute trading queries
  const runExecutor = new RunExecutor();

  // Get trading signals
  const signals = await toolExecutor.getTradingSignals({
    trading_pair: 'BTC/USDT',
    signal_type: 'all',
    timeframe: '4h',
  });

  console.log('Trading signals:', signals.knowledge);

  // Execute analysis run
  const run = await runExecutor.execute({
    agent_id: agent.agent_id,
    input: {
      messages: [
        { role: 'user', content: 'Analyze BTC/USDT for potential entry points' },
      ],
      context: {
        signals: signals.knowledge,
      },
    },
    options: {
      stream: true,
    },
  });

  // Stream results
  for await (const event of run.stream()) {
    if (event.type === 'token') {
      process.stdout.write(event.token);
    }
  }

  return run;
}
```

### Legal Agent Example

```typescript
import { AgentRegistry } from '@/cap/services/agent-registry';
import { RunExecutor } from '@/cap/services/run-executor';
import { RAGStepHandler } from '@/cap/step-handlers/handlers/rag-step-handler';

async function legalAgentExample() {
  const registry = new AgentRegistry();
  const executor = new RunExecutor();

  // Register legal agent
  const agent = await registry.register({
    agent_id: 'clicbrain.legal.v1',
    name: 'Legal Research Agent',
    version: '1.0.0',
    system: 'clicbrain',
    type: 'legal',
    capabilities: {
      domains: ['legal', 'compliance'],
      actions: ['llm_generate', 'query_clicbrain', 'dspy_rag'],
      tools: ['search_knowledge'],
    },
    cap_extensions: {
      supports_interrupts: true, // For human review of legal advice
    },
  });

  // Execute legal research
  const run = await executor.execute({
    agent_id: agent.agent_id,
    input: {
      messages: [
        {
          role: 'user',
          content: 'What are the data protection requirements under Uganda law?',
        },
      ],
    },
    options: {
      timeout_ms: 120000,
    },
  });

  // Handle potential interrupt for review
  if (run.status === 'interrupted') {
    console.log('Legal response requires human review');
    // In real app, show to human reviewer
  }

  return run;
}
```

### Document Processing Example

```typescript
import { AgentRegistry } from '@/cap/services/agent-registry';
import { RunExecutor } from '@/cap/services/run-executor';

async function documentProcessingExample() {
  const registry = new AgentRegistry();
  const executor = new RunExecutor();

  // Register document processing agent
  const agent = await registry.register({
    agent_id: 'clicbrain.doc-processor.v1',
    name: 'Document Processor',
    version: '1.0.0',
    system: 'clicbrain',
    type: 'document_processing',
    capabilities: {
      domains: ['legal', 'compliance'],
      actions: ['dspy_extraction', 'dspy_classification'],
    },
  });

  // Extract contract details
  const extractionRun = await executor.execute({
    agent_id: agent.agent_id,
    input: {
      operation: 'extraction',
      text: `
        SERVICE AGREEMENT

        This Agreement is entered into as of January 15, 2026 between:
        ABC Corporation Ltd ("Client") and XYZ Services Inc ("Provider")

        Term: 24 months commencing February 1, 2026
        Value: $150,000 per annum
        Governing Law: Laws of Uganda

        Services: Software development and maintenance
      `,
      schema: {
        parties: { type: 'array' },
        effective_date: { type: 'string' },
        term_months: { type: 'number' },
        annual_value: { type: 'number' },
        governing_law: { type: 'string' },
        service_type: { type: 'string' },
      },
    },
  });

  console.log('Extracted data:', extractionRun.output?.extracted);

  // Classify document type
  const classificationRun = await executor.execute({
    agent_id: agent.agent_id,
    input: {
      operation: 'classification',
      text: extractionRun.input.text,
      categories: [
        'service_agreement',
        'employment_contract',
        'nda',
        'lease_agreement',
        'sale_contract',
      ],
    },
  });

  console.log('Document type:', classificationRun.output?.labels);

  return { extractionRun, classificationRun };
}
```

---

## Streaming Example

```typescript
import { RunExecutor } from '@/cap/services/run-executor';

async function streamingExample(agentId: string) {
  const executor = new RunExecutor();

  const run = await executor.execute({
    agent_id: agentId,
    input: {
      messages: [
        { role: 'user', content: 'Generate a detailed market analysis report' },
      ],
    },
    options: {
      stream: true,
      timeout_ms: 300000,
    },
  });

  // Process stream events
  for await (const event of run.stream()) {
    switch (event.type) {
      case 'step:started':
        console.log('\n[Step Started]', event.step.name);
        break;

      case 'step:completed':
        console.log('\n[Step Completed]', event.step.name);
        break;

      case 'token':
        process.stdout.write(event.token);
        break;

      case 'tool:calling':
        console.log('\n[Calling Tool]', event.tool.name);
        break;

      case 'tool:result':
        console.log('\n[Tool Result]', event.tool.name);
        break;

      case 'interrupt':
        console.log('\n[Interrupt]', event.interrupt.type);
        // Handle interrupt
        break;

      case 'completed':
        console.log('\n\n[Run Completed]');
        break;

      case 'error':
        console.error('\n[Error]', event.error);
        break;
    }
  }

  return run;
}
```

---

## Interrupt Handling Example

```typescript
import { RunExecutor } from '@/cap/services/run-executor';
import { InterruptHandler } from '@/cap/services/interrupt-handler';

async function interruptExample(agentId: string) {
  const executor = new RunExecutor();
  const interruptHandler = new InterruptHandler();

  // Execute a run that may require approval
  const run = await executor.execute({
    agent_id: agentId,
    input: {
      messages: [
        { role: 'user', content: 'Execute a high-value trade for 10 BTC' },
      ],
      context: {
        trade_value_usd: 450000,
      },
    },
  });

  // Check for interrupts
  if (run.status === 'interrupted') {
    const pendingInterrupts = await interruptHandler.listPending({
      run_id: run.id,
    });

    for (const interrupt of pendingInterrupts) {
      console.log('Interrupt:', interrupt.type);
      console.log('Message:', interrupt.payload.message);

      // In a real app, present to user and get response
      const userApproval = await getUserApproval(interrupt);

      // Resolve the interrupt
      await interruptHandler.resolve(interrupt.id, {
        response: userApproval ? 'approved' : 'rejected',
        resolved_by: 'user-123',
        metadata: {
          decision_reason: userApproval
            ? 'User approved the trade'
            : 'User rejected due to risk concerns',
        },
      });
    }

    // Resume the run
    const resumedRun = await executor.resume(run.id);
    return resumedRun;
  }

  return run;
}

async function getUserApproval(interrupt: any): Promise<boolean> {
  // Simulate user approval (in real app, this would be a UI interaction)
  return true;
}
```

---

## Multi-Agent Orchestration Example

```typescript
import { AgentRegistry } from '@/cap/services/agent-registry';
import { RunExecutor } from '@/cap/services/run-executor';
import { CAPHub } from '@/cap/services/cap-hub';

async function multiAgentExample() {
  const registry = new AgentRegistry();
  const executor = new RunExecutor();

  // Register orchestrator agent
  const orchestrator = await registry.register({
    agent_id: 'clix.orchestrator.v1',
    name: 'CLIX Master Orchestrator',
    version: '1.0.0',
    system: 'clix',
    type: 'master_orchestrator',
    capabilities: {
      domains: ['global'],
      actions: ['call_agent', 'parallel_agents'],
    },
  });

  // Register specialized agents
  const tradingAgent = await registry.register({
    agent_id: 'clix.trading.v1',
    name: 'Trading Agent',
    version: '1.0.0',
    system: 'clix',
    type: 'trading',
    capabilities: {
      domains: ['trading'],
      actions: ['llm_generate', 'llm_analyze'],
    },
  });

  const analyticsAgent = await registry.register({
    agent_id: 'clix.analytics.v1',
    name: 'Analytics Agent',
    version: '1.0.0',
    system: 'clix',
    type: 'analytics',
    capabilities: {
      domains: ['analytics'],
      actions: ['llm_generate', 'llm_summarize'],
    },
  });

  // Execute orchestrated run
  const run = await executor.execute({
    agent_id: orchestrator.agent_id,
    input: {
      messages: [
        {
          role: 'user',
          content: 'Analyze my portfolio and suggest optimizations',
        },
      ],
      context: {
        portfolio_id: 'portfolio-123',
        user_id: 'user-456',
      },
    },
    options: {
      // Orchestrator will route to specialized agents
      enable_agent_calls: true,
    },
  });

  // Check steps to see which agents were called
  for (const step of run.steps) {
    if (step.type === 'agent_call') {
      console.log('Called agent:', step.called_agent_id);
      console.log('Result:', step.output_summary);
    }
  }

  return run;
}
```

---

## Distributed Execution Example

```typescript
import { RunExecutor } from '@/cap/services/run-executor';

async function distributedExample() {
  // Configure for distributed mode
  const executor = new RunExecutor({
    mode: 'distributed',
    redis: process.env.REDIS_URL,
    queue: 'cap:runs',
    concurrency: 10,
  });

  // Submit run (will be queued)
  const run = await executor.execute({
    agent_id: 'my-agent',
    input: {
      messages: [{ role: 'user', content: 'Process this large dataset' }],
    },
  });

  console.log('Run queued:', run.id);
  console.log('Status:', run.status); // 'queued'

  // Poll for completion
  let status = await executor.getStatus(run.id);
  while (status.status !== 'completed' && status.status !== 'failed') {
    await new Promise((resolve) => setTimeout(resolve, 1000));
    status = await executor.getStatus(run.id);
    console.log('Status:', status.status);
  }

  // Get final result
  const result = await executor.get(run.id);
  return result;
}
```

---

## Custom Step Handler Example

```typescript
import { BaseStepHandler } from '@/cap/step-handlers/base-step-handler';
import { StepHandlerRegistry } from '@/cap/step-handlers/step-handler-registry';
import { ok, err } from '@/core/errors';

// Define custom input/output types
interface SentimentInput {
  operation?: 'sentiment';
  text: string;
}

interface SentimentOutput {
  operation: 'sentiment';
  agent_id: string;
  sentiment: 'positive' | 'negative' | 'neutral';
  confidence: number;
  reasoning: string;
}

// Create custom handler
class SentimentStepHandler extends BaseStepHandler<SentimentInput, SentimentOutput> {
  constructor() {
    super({
      name: 'sentiment-handler',
      version: '1.0.0',
      operationType: 'generic', // Or register a new type
      description: 'Analyzes text sentiment',
      priority: 80,
    });
  }

  canHandle(input: any, agent: any): boolean {
    return input.operation === 'sentiment' ||
           (input.text && !input.categories && !input.schema);
  }

  protected async executeStep(
    context: any,
    input: SentimentInput
  ) {
    const { services, agent } = context;

    // Use DSPy for sentiment analysis
    const response = await services.dspyClient.classify({
      text: input.text,
      categories: ['positive', 'negative', 'neutral'],
    });

    return ok({
      operation: 'sentiment' as const,
      agent_id: agent.agent_id,
      sentiment: response.labels[0] as 'positive' | 'negative' | 'neutral',
      confidence: response.scores[response.labels[0]],
      reasoning: response.reasoning,
    });
  }
}

// Register the handler
async function registerCustomHandler() {
  const registry = new StepHandlerRegistry();
  registry.register(new SentimentStepHandler());

  // Use the handler
  const routing = registry.route({
    input: { operation: 'sentiment', text: 'This product is amazing!' },
    agent: { agent_id: 'my-agent', capabilities: {} },
  });

  if (routing) {
    console.log('Selected handler:', routing.handler.metadata.name);
  }
}
```

---

## Testing Example

```typescript
import { AgentRegistry } from '@/cap/services/agent-registry';
import { RunExecutor } from '@/cap/services/run-executor';
import { ThreadManager } from '@/cap/services/thread-manager';

describe('Agent Integration Tests', () => {
  let registry: AgentRegistry;
  let executor: RunExecutor;
  let threadManager: ThreadManager;
  let testAgent: any;

  beforeAll(async () => {
    registry = new AgentRegistry();
    executor = new RunExecutor({ mode: 'local' });
    threadManager = new ThreadManager();

    // Register test agent
    testAgent = await registry.register({
      agent_id: 'test.agent.v1',
      name: 'Test Agent',
      version: '1.0.0',
      system: 'clicbrain',
      type: 'llm',
      capabilities: {
        domains: ['global'],
        actions: ['llm_generate'],
      },
    });
  });

  afterAll(async () => {
    // Cleanup
    await registry.delete(testAgent.id);
  });

  it('should execute a simple run', async () => {
    const run = await executor.execute({
      agent_id: testAgent.agent_id,
      input: {
        messages: [{ role: 'user', content: 'Hello' }],
      },
    });

    expect(run.status).toBe('completed');
    expect(run.output?.response).toBeDefined();
  });

  it('should maintain thread context', async () => {
    const thread = await threadManager.create({
      agent_id: testAgent.agent_id,
    });

    await threadManager.addMessage(thread.id, {
      role: 'user',
      content: 'My name is Alice',
    });

    const run1 = await executor.execute({
      agent_id: testAgent.agent_id,
      thread_id: thread.id,
      input: { messages: [] },
    });

    await threadManager.addMessage(thread.id, {
      role: 'user',
      content: 'What is my name?',
    });

    const run2 = await executor.execute({
      agent_id: testAgent.agent_id,
      thread_id: thread.id,
      input: { messages: [] },
    });

    // Agent should remember the name from context
    expect(run2.output?.response).toContain('Alice');

    await threadManager.close(thread.id);
  });

  it('should handle timeouts', async () => {
    await expect(
      executor.execute({
        agent_id: testAgent.agent_id,
        input: {
          messages: [{ role: 'user', content: 'Long operation' }],
        },
        options: {
          timeout_ms: 1, // Very short timeout
        },
      })
    ).rejects.toThrow(/timeout/i);
  });
});
```

---

## Best Practices Checklist

### Agent Registration
- [ ] Use semantic versioning for agent versions
- [ ] Define appropriate capabilities
- [ ] Set realistic timeout values
- [ ] Enable features based on use case (threads, interrupts, streaming)

### Run Execution
- [ ] Use streaming for long operations
- [ ] Handle all run statuses (completed, failed, interrupted, cancelled)
- [ ] Set appropriate timeouts
- [ ] Use checkpointing for long runs

### Thread Management
- [ ] Create threads for multi-turn conversations
- [ ] Add metadata for user/session tracking
- [ ] Summarize long conversations to manage context
- [ ] Close threads when conversations end

### Error Handling
- [ ] Check for retryable errors
- [ ] Implement retry logic with backoff
- [ ] Log errors with context
- [ ] Handle interrupts appropriately

### Testing
- [ ] Test happy paths
- [ ] Test error scenarios
- [ ] Test timeout handling
- [ ] Test interrupt flows
- [ ] Clean up test resources
