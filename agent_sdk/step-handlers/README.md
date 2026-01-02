# Step Handlers

Step handlers are modular execution units that process specific types of operations within CAP runs. They encapsulate DSPy operations and provide structured input/output types, error handling, and observability.

## Overview

The step handler system provides:

- **Modular Execution** - Encapsulated logic for each operation type
- **Intelligent Routing** - Automatic handler selection based on input patterns
- **Observability** - Built-in OpenTelemetry tracing and metrics
- **Error Handling** - Result-based error handling with retry support
- **Human-in-the-Loop** - Interrupt handling for low-confidence responses

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Step Handler Registry                     │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   Routing   │  │  Capability  │  │  Pattern Matching │  │
│  │   Engine    │  │  Matching    │  │                   │  │
│  └──────┬──────┘  └──────┬───────┘  └─────────┬─────────┘  │
└─────────┼────────────────┼──────────────────────┼───────────┘
          │                │                      │
          ▼                ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Base Step Handler                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  Lifecycle  │  │   Tracing    │  │  Thread/Interrupt │  │
│  │  Management │  │  Integration │  │     Handling      │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                  Concrete Handlers                           │
│  ┌──────┐  ┌──────────┐  ┌────────────┐  ┌───────────┐    │
│  │ RAG  │  │Reasoning │  │Classification│ │ Extraction│    │
│  └──────┘  └──────────┘  └────────────┘  └───────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## Operation Types

| Operation | Description | Input Pattern |
|-----------|-------------|---------------|
| `rag` | Retrieval-Augmented Generation | query/question + context_ids |
| `reasoning` | Chain-of-thought reasoning | question + optional context |
| `classification` | Text classification | text + categories |
| `extraction` | Structured data extraction | text + schema |
| `generic` | General LLM operations | message/request |
| `tool_call` | Tool execution | tool_name + args |
| `agent_invocation` | Sub-agent calls | agent_id + input |

## Base Step Handler

All step handlers extend `BaseStepHandler`, which provides:

```typescript
import { BaseStepHandler } from '@/cap/step-handlers/base-step-handler';

class MyCustomHandler extends BaseStepHandler<MyInput, MyOutput> {
  constructor() {
    super({
      name: 'my-custom-handler',
      version: '1.0.0',
      operationType: 'generic',
      description: 'Custom operation handler',
      requiredCapabilities: ['my-capability'],
      priority: 100,
    });
  }

  canHandle(input: BaseStepInput, agent: CAPAgent): boolean {
    return input.operation === 'my-operation';
  }

  protected async executeStep(
    context: StepExecutionContext,
    input: MyInput
  ): Promise<Result<MyOutput, StepError>> {
    // Your implementation here
    return ok(output);
  }
}
```

### Handler Metadata

```typescript
interface StepHandlerMetadata {
  // Handler name for logging and identification
  name: string;

  // Semantic version
  version: string;

  // Operation type this handler processes
  operationType: StepOperationType;

  // Description for documentation
  description: string;

  // Required agent capabilities (tools or actions)
  requiredCapabilities?: string[];

  // Priority for routing (higher = preferred)
  priority: number;
}
```

### Execution Context

Handlers receive a context with services and callbacks:

```typescript
interface StepExecutionContext {
  // Current run
  run: CAPRun;

  // Executing agent
  agent: CAPAgent;

  // Thread context if available
  threadContext?: {
    summary?: string;
    messages: Array<{ role: string; content: string }>;
    truncated: boolean;
  };

  // Cancellation controller
  abortController: AbortController;

  // Available services
  services: {
    dspyClient: DSPyClient;
    threadManager: ThreadManagerService;
    interruptHandler: InterruptHandlerService;
  };

  // Step lifecycle callbacks
  addStep: (input: AddStepInput) => Promise<RunStep>;
  completeStep: (stepId: string, result: CompleteStepInput) => Promise<boolean>;
  updateTokenUsage: (tokens: TokenUsage) => void;
}
```

---

## RAG Step Handler

Handles Retrieval-Augmented Generation operations.

### Input

```typescript
interface RAGStepInput {
  operation?: 'rag';

  // Query or question to answer
  query?: string;
  question?: string;

  // Optional context IDs for filtering
  context_ids?: string[];

  // Number of results to retrieve (default: 5)
  top_k?: number;

  // Minimum confidence threshold
  min_confidence?: number;
}
```

### Output

```typescript
interface RAGStepOutput {
  operation: 'rag';
  agent_id: string;

  // Generated answer
  answer: string;

  // Confidence score (0-1)
  confidence: number;

  // Reasoning explanation
  reasoning: string;

  // Source documents
  sources: Array<{
    id: string;
    title: string;
    description: string;
    content: string;
    metadata: Record<string, unknown>;
  }>;

  // Whether human review is recommended
  requires_review: boolean;
}
```

### Usage

```typescript
import { RAGStepHandler } from '@/cap/step-handlers/handlers/rag-step-handler';

const handler = new RAGStepHandler({
  reviewThreshold: 0.7, // Flag for review if confidence < 0.7
});

// Execute via registry
const result = await handler.execute(context, {
  query: 'What are the requirements for company registration?',
  context_ids: ['legal-domain'],
  top_k: 5,
});

if (result.ok) {
  console.log('Answer:', result.value.answer);
  console.log('Confidence:', result.value.confidence);
  console.log('Sources:', result.value.sources);
}
```

---

## Reasoning Step Handler

Handles chain-of-thought reasoning operations.

### Input

```typescript
interface ReasoningStepInput {
  operation?: 'reasoning';

  // Question to reason about
  question?: string;
  query?: string;

  // Additional context
  context?: string;

  // Maximum reasoning steps
  max_steps?: number;
}
```

### Output

```typescript
interface ReasoningStepOutput {
  operation: 'reasoning';
  agent_id: string;

  // Final answer
  answer: string;

  // Confidence score
  confidence: number;

  // Chain of reasoning steps
  reasoning_chain: string[];
}
```

### Usage

```typescript
import { ReasoningStepHandler } from '@/cap/step-handlers/handlers/reasoning-step-handler';

const handler = new ReasoningStepHandler();

const result = await handler.execute(context, {
  question: 'Should I use DCA or Grid strategy for BTC trading?',
  context: 'User has moderate risk tolerance and 6-month horizon.',
  max_steps: 5,
});
```

---

## Classification Step Handler

Handles text classification operations.

### Input

```typescript
interface ClassificationStepInput {
  operation?: 'classification';

  // Text to classify
  text: string;

  // Available categories
  categories: string[];

  // Allow multiple labels (default: false)
  multi_label?: boolean;
}
```

### Output

```typescript
interface ClassificationStepOutput {
  operation: 'classification';
  agent_id: string;

  // Predicted labels
  labels: string[];

  // Confidence scores per label
  scores: Record<string, number>;

  // Classification reasoning
  reasoning: string;
}
```

### Usage

```typescript
import { ClassificationStepHandler } from '@/cap/step-handlers/handlers/classification-step-handler';

const handler = new ClassificationStepHandler();

const result = await handler.execute(context, {
  text: 'The defendant failed to disclose material information...',
  categories: ['fraud', 'negligence', 'breach_of_contract', 'compliance'],
  multi_label: true,
});

if (result.ok) {
  console.log('Labels:', result.value.labels);
  console.log('Scores:', result.value.scores);
}
```

---

## Extraction Step Handler

Handles structured data extraction operations.

### Input

```typescript
interface ExtractionStepInput {
  operation?: 'extraction';

  // Text to extract from
  text: string;

  // Extraction schema
  schema: Record<string, unknown>;

  // Require all fields (default: false)
  strict?: boolean;
}
```

### Output

```typescript
interface ExtractionStepOutput {
  operation: 'extraction';
  agent_id: string;

  // Extracted data
  extracted: Record<string, unknown>;

  // Extraction confidence
  confidence: number;

  // Fields that couldn't be extracted
  missing_fields: string[];
}
```

### Usage

```typescript
import { ExtractionStepHandler } from '@/cap/step-handlers/handlers/extraction-step-handler';

const handler = new ExtractionStepHandler();

const result = await handler.execute(context, {
  text: 'Contract between ABC Corp and XYZ Ltd dated January 15, 2026...',
  schema: {
    parties: { type: 'array', items: { type: 'string' } },
    effective_date: { type: 'string', format: 'date' },
    termination_date: { type: 'string', format: 'date' },
    contract_value: { type: 'number' },
    governing_law: { type: 'string' },
  },
  strict: false,
});
```

---

## Step Handler Registry

The registry manages handler registration and routing.

### Registration

```typescript
import { StepHandlerRegistry } from '@/cap/step-handlers/step-handler-registry';

const registry = new StepHandlerRegistry({
  minConfidence: 0.5,
  useCapabilityRouting: true,
  usePatternMatching: true,
  defaultOperation: 'generic',
});

// Register handlers
registry.register(new RAGStepHandler());
registry.register(new ReasoningStepHandler());
registry.register(new ClassificationStepHandler());
registry.register(new ExtractionStepHandler());
```

### Routing

```typescript
// Route to best handler
const routing = registry.route({
  input: { query: 'What are the AML requirements?' },
  agent: myAgent,
});

if (routing) {
  console.log('Selected handler:', routing.handler.metadata.name);
  console.log('Confidence:', routing.confidence);
  console.log('Reason:', routing.reason);

  // Execute
  const result = await routing.handler.execute(context, input);
}
```

### Input Pattern Detection

The registry automatically detects operation types from input patterns:

| Pattern | Detected Operation | Confidence |
|---------|-------------------|------------|
| `text` + `categories` | classification | 0.95 |
| `text` + `schema` | extraction | 0.95 |
| `query/question` + `context_ids` | rag | 0.90 |
| `question` (no context_ids) | reasoning | 0.70 |
| `query` (no context_ids) | rag | 0.60 |
| `message/request` | generic | 0.50 |

### Singleton Access

```typescript
import { getStepHandlerRegistry } from '@/cap/step-handlers/step-handler-registry';

// Get singleton instance
const registry = getStepHandlerRegistry();

// Reset for testing
import { resetStepHandlerRegistry } from '@/cap/step-handlers/step-handler-registry';
resetStepHandlerRegistry();
```

---

## Error Handling

Step handlers use Result-based error handling:

```typescript
interface StepError {
  // Error code from taxonomy
  code: ErrorCodeType;

  // Human-readable message
  message: string;

  // Original error if wrapped
  cause?: Error;

  // Whether the error is retryable
  retryable: boolean;

  // Additional context
  context?: Record<string, unknown>;
}
```

### Common Error Codes

| Code | Description | Retryable |
|------|-------------|-----------|
| `CAP_RUN_CANCELLED` | Run was cancelled | No |
| `CAP_RUN_EXECUTION_FAILED` | Execution failed | No |
| `RAG_QUERY_FAILED` | RAG query failed | No |
| `NET_CONNECTION_FAILED` | DSPy connection failed | Yes |
| `TIMEOUT_OPERATION` | Operation timed out | Yes |
| `VALID_REQUIRED_FIELD` | Required field missing | No |

### Error Handling Example

```typescript
const result = await handler.execute(context, input);

if (!result.ok) {
  const error = result.error;

  if (error.retryable) {
    // Retry logic
    console.log('Retrying:', error.message);
  } else {
    // Handle permanent failure
    console.error('Fatal error:', error.code, error.message);
  }
}
```

---

## Token Usage Tracking

Handlers automatically track token usage:

```typescript
interface TokenUsage {
  input: number;
  output: number;
  total: number;
  estimatedCost: number;
}

// Tokens are estimated from response size
// ~4 characters per token
// Cost: $3/1M input, $15/1M output tokens
```

---

## Creating Custom Handlers

```typescript
import { BaseStepHandler } from '@/cap/step-handlers/base-step-handler';
import { ok, err } from '@/core/errors';

interface CustomInput extends BaseStepInput {
  operation?: 'custom';
  data: Record<string, unknown>;
}

interface CustomOutput extends BaseStepOutput {
  operation: 'custom';
  result: unknown;
}

class CustomStepHandler extends BaseStepHandler<CustomInput, CustomOutput> {
  constructor() {
    super({
      name: 'custom-handler',
      version: '1.0.0',
      operationType: 'generic', // Or register a new type
      description: 'Custom operation handler',
      priority: 50,
    });
  }

  canHandle(input: BaseStepInput, agent: CAPAgent): boolean {
    return (input as CustomInput).data !== undefined;
  }

  protected async executeStep(
    context: StepExecutionContext,
    input: CustomInput
  ): Promise<Result<CustomOutput, StepError>> {
    // Check cancellation
    this.checkCancellation(context);

    try {
      // Your custom logic
      const result = await this.processData(input.data);

      // Add to thread if needed
      await this.addThreadMessage(context, JSON.stringify(result));

      return ok({
        operation: 'custom',
        agent_id: context.agent.agent_id,
        result,
      });
    } catch (error) {
      return err(this.wrapError(error as Error));
    }
  }

  private async processData(data: Record<string, unknown>): Promise<unknown> {
    // Implementation
    return data;
  }
}
```

---

## Best Practices

### 1. Always Check Cancellation

```typescript
protected async executeStep(context, input) {
  this.checkCancellation(context);

  // After long operations
  const result = await longOperation();
  this.checkCancellation(context);

  return ok(result);
}
```

### 2. Use Result Pattern

```typescript
// Return ok() for success
return ok(output);

// Return err() for failures
return err(createStepError(
  ErrorCode.RAG_QUERY_FAILED,
  'Query failed',
  { retryable: false }
));
```

### 3. Add Thread Messages

```typescript
await this.addThreadMessage(
  context,
  'Response content',
  { confidence: 0.95, sources: ['doc-1'] }
);
```

### 4. Create Interrupts for Human Review

```typescript
if (response.confidence < 0.7) {
  await this.createInterrupt(context, {
    type: 'approval',
    message: 'Low confidence response requires review',
    details: { answer: response.answer, confidence: response.confidence },
    timeout_ms: 300000,
  });
}
```

### 5. Track Span Events

```typescript
this.addTraceEvent('knowledge_retrieved', {
  sources: sources.length,
  confidence: confidence,
});
```
