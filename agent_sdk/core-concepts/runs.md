# Runs

Runs represent individual execution instances of an agent. Each run processes input, executes steps, and produces output within a defined lifecycle.

## Run Schema

```typescript
interface Run {
  id: string;
  agent_id: string;
  thread_id?: string;
  status: RunStatus;
  input: RunInput;
  output?: RunOutput;
  steps: RunStep[];
  error?: string;
  metadata: RunMetadata;
  created_at: Date;
  started_at?: Date;
  completed_at?: Date;
}
```

## Run Status

```
┌─────────┐
│ pending │  Run created, waiting to start
└────┬────┘
     │
     ▼
┌─────────┐
│ queued  │  In distributed queue (distributed mode)
└────┬────┘
     │
     ▼
┌─────────┐     ┌─────────────┐
│ running │────►│  streaming  │  Generating streaming output
└────┬────┘     └──────┬──────┘
     │                 │
     ▼                 │
┌─────────────┐        │
│ interrupted │◄───────┘  Waiting for human input
└─────────────┘
     │
     ▼
┌───────────┐
│ completed │  Successfully finished
└───────────┘

Error states: failed, cancelled, timeout
```

| Status | Description |
|--------|-------------|
| `pending` | Created, not started |
| `queued` | In queue (distributed mode) |
| `running` | Actively processing |
| `streaming` | Streaming output |
| `interrupted` | Waiting for human input |
| `completed` | Successfully finished |
| `failed` | Failed with error |
| `cancelled` | Manually cancelled |
| `timeout` | Exceeded timeout |

## Run Steps

### Step Types

```typescript
type RunStepType =
  | 'llm_call'           // LLM inference
  | 'tool_call'          // Tool execution
  | 'agent_call'         // Sub-agent invocation
  | 'decision'           // Decision point
  | 'skill_execution'    // Skill execution
  | 'knowledge_query'    // RAG query
  | 'parallel_execution'; // Parallel operations
```

### Step Schema

```typescript
interface RunStep {
  id: string;
  run_id: string;
  type: RunStepType;
  name: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  input: unknown;
  output?: unknown;
  error?: string;
  duration_ms?: number;
  started_at?: Date;
  completed_at?: Date;
}
```

## Creating Runs

### Basic Run

```typescript
import { RunExecutor } from '@/cap/services/run-executor';

const executor = new RunExecutor();

const run = await executor.execute({
  agent_id: 'my-agent',
  input: {
    messages: [
      { role: 'user', content: 'Hello, agent!' },
    ],
  },
});
```

### Run with Thread

```typescript
const run = await executor.execute({
  agent_id: 'my-agent',
  thread_id: 'thread-123',
  input: {
    messages: [
      { role: 'user', content: 'Continue our conversation' },
    ],
  },
});
```

### Run with Options

```typescript
const run = await executor.execute({
  agent_id: 'my-agent',
  input: {
    messages: [{ role: 'user', content: 'Analyze this data' }],
    context: {
      user_id: 'user-123',
      portfolio_id: 'portfolio-456',
    },
  },
  options: {
    timeout_ms: 60000,
    stream: true,
    checkpoint_interval_ms: 5000,
  },
});
```

## Streaming Runs

```typescript
// Execute with streaming
const run = await executor.execute({
  agent_id: 'my-agent',
  input: { messages: [{ role: 'user', content: 'Generate a report' }] },
  options: { stream: true },
});

// Process stream events
for await (const event of run.stream()) {
  switch (event.type) {
    case 'step:started':
      console.log('Step started:', event.step.name);
      break;
    case 'step:completed':
      console.log('Step completed:', event.step.name);
      break;
    case 'token':
      process.stdout.write(event.token);
      break;
    case 'interrupt':
      console.log('Interrupt:', event.interrupt.type);
      break;
    case 'completed':
      console.log('Run completed');
      break;
    case 'error':
      console.error('Error:', event.error);
      break;
  }
}
```

### Stream Events

| Event | Description |
|-------|-------------|
| `step:started` | Step execution started |
| `step:completed` | Step execution completed |
| `token` | Token generated (LLM streaming) |
| `tool:calling` | Tool being called |
| `tool:result` | Tool returned result |
| `interrupt` | Interrupt triggered |
| `completed` | Run completed |
| `error` | Error occurred |

## Run Execution Modes

### Local Mode

```typescript
const executor = new RunExecutor({
  mode: 'local',
});

// Runs execute in-process
const run = await executor.execute({ ... });
```

### Distributed Mode

```typescript
const executor = new RunExecutor({
  mode: 'distributed',
  redis: process.env.REDIS_URL,
  queue: 'cap:runs',
  concurrency: 10,
});

// Runs are queued and processed by workers
const run = await executor.execute({ ... });

// Check status
const status = await executor.getStatus(run.id);
```

## Step Handlers

Runs use step handlers to process different step types:

```typescript
import { StepHandlerRegistry } from '@/cap/step-handlers/step-handler-registry';

const registry = new StepHandlerRegistry();

// Built-in handlers
registry.register('rag', new RAGStepHandler());
registry.register('extraction', new ExtractionStepHandler());
registry.register('classification', new ClassificationStepHandler());
registry.register('reasoning', new ReasoningStepHandler());

// Custom handler
registry.register('my-handler', {
  async execute(step, context) {
    // Custom logic
    return { result: 'done' };
  },
});
```

## Checkpointing

Runs automatically checkpoint state for recovery:

```typescript
const run = await executor.execute({
  agent_id: 'my-agent',
  input: { ... },
  options: {
    checkpoint_interval_ms: 10000, // Checkpoint every 10s
  },
});

// Resume from checkpoint after failure
const resumed = await executor.resume(run.id);
```

## Run Cancellation

```typescript
// Cancel a running run
await executor.cancel(run.id, {
  reason: 'User requested',
});

// Check cancellation
const run = await executor.get(runId);
if (run.status === 'cancelled') {
  console.log('Run was cancelled');
}
```

## Run Timeout

```typescript
const run = await executor.execute({
  agent_id: 'my-agent',
  input: { ... },
  options: {
    timeout_ms: 60000, // 1 minute timeout
  },
});

// Handle timeout
run.on('timeout', () => {
  console.log('Run timed out');
});
```

## Run Events

```typescript
// Subscribe to run events
executor.on('run:started', (run) => {
  console.log('Run started:', run.id);
});

executor.on('run:completed', (run) => {
  console.log('Run completed:', run.id);
  console.log('Output:', run.output);
});

executor.on('run:failed', (run, error) => {
  console.error('Run failed:', run.id, error);
});

executor.on('run:interrupted', (run, interrupt) => {
  console.log('Run interrupted:', interrupt.type);
});
```

## Run Output

```typescript
interface RunOutput {
  // Final response
  response?: string;

  // Structured data
  data?: unknown;

  // Collected artifacts
  artifacts?: {
    name: string;
    type: string;
    content: unknown;
  }[];

  // Token usage
  usage?: {
    input_tokens: number;
    output_tokens: number;
    total_tokens: number;
  };

  // Execution metrics
  metrics?: {
    total_duration_ms: number;
    llm_duration_ms: number;
    tool_duration_ms: number;
  };
}
```

## Run Metadata

```typescript
interface RunMetadata {
  // Request context
  user_id?: string;
  session_id?: string;
  request_id?: string;

  // Execution context
  model?: string;
  temperature?: number;

  // Tracing
  trace_id?: string;
  span_id?: string;

  // Custom metadata
  [key: string]: unknown;
}
```

## Best Practices

### 1. Use Appropriate Timeouts

```typescript
// Quick operation
options: { timeout_ms: 30000 } // 30 seconds

// Complex analysis
options: { timeout_ms: 300000 } // 5 minutes

// Long-running workflow
options: { timeout_ms: 1800000 } // 30 minutes
```

### 2. Enable Streaming for Long Operations

```typescript
// Always stream for user-facing operations
const run = await executor.execute({
  ...config,
  options: { stream: true },
});
```

### 3. Handle Interrupts

```typescript
run.on('interrupt', async (interrupt) => {
  // Show UI to user
  const response = await showInterruptDialog(interrupt);

  // Resolve interrupt
  await interruptHandler.resolve(interrupt.id, { response });
});
```

### 4. Use Checkpointing for Long Runs

```typescript
options: {
  checkpoint_interval_ms: 30000, // Every 30 seconds
}
```

### 5. Monitor Run Metrics

```typescript
run.on('completed', (run) => {
  // Log metrics
  metrics.recordHistogram('run_duration', run.output.metrics.total_duration_ms);
  metrics.recordCounter('run_tokens', run.output.usage.total_tokens);
});
```
