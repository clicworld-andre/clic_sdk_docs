# Reference

Complete reference documentation for CAP types, enums, and schemas.

## Table of Contents

1. [Enums](#enums)
2. [Agent Types](#agent-types)
3. [Run Types](#run-types)
4. [Thread Types](#thread-types)
5. [Interrupt Types](#interrupt-types)
6. [Skill Types](#skill-types)
7. [Error Codes](#error-codes)

---

## Enums

### ClicSystem

Systems in the Clic ecosystem:

| Value | Description |
|-------|-------------|
| `clicbrain` | ClicBrain knowledge platform |
| `clix_trading` | CLIX Trading platform |
| `clic_banking` | Clic Banking services |
| `clic_voice` | Voice interface system |
| `clicme` | Personal assistant |
| `oneclicid` | Identity verification |
| `clic-brain` | ClicBrain (hyphen variant) |
| `clic-law` | Legal services |
| `clic-finance` | Financial services |
| `clic-me` | Personal (hyphen variant) |
| `clic-id` | Identity (hyphen variant) |
| `clix` | Trading (hyphen variant) |

### AgentType

Types of agents:

| Value | System | Description |
|-------|--------|-------------|
| `llm` | CAP Core | LLM-powered agent |
| `orchestrator` | CAP Core | Multi-agent coordinator |
| `tool` | CAP Core | Tool-based agent |
| `hybrid` | CAP Core | Combination of types |
| `coding` | ClicBrain | Code generation/analysis |
| `legal` | ClicBrain | Legal research/analysis |
| `compliance` | ClicBrain | Compliance checking |
| `document_processing` | ClicBrain | Document extraction |
| `accounting` | ClicBrain | Financial processing |
| `knowledge_manager` | ClicBrain | Knowledge management |
| `wallet` | Clix Trading | Wallet operations |
| `trading` | Clix Trading | Trading operations |
| `bot` | Clix Trading | Trading bot management |
| `bond_auction` | Clix Trading | Bond auction operations |
| `letter_of_credit` | Clix Trading | LC processing |
| `commodities` | Clix Trading | Commodity trading |
| `analytics` | Clix Trading | Analytics/reporting |
| `master_orchestrator` | Clix Trading | Multi-agent orchestration |
| `investment_advisor` | Clic Banking | Investment advice |
| `transaction_advisor` | Clic Banking | Transaction guidance |
| `customer_service` | Clic Banking | Customer support |
| `voice_interface` | Clic Voice | Voice interactions |
| `personal_assistant` | Clicme | Personal assistance |
| `identity_verification` | OneClicID | Identity verification |
| `trust_rating` | OneClicID | Trust scoring |

### AgentStatus

Agent operational status:

| Value | Description |
|-------|-------------|
| `active` | Agent is operational |
| `inactive` | Agent is disabled |
| `deprecated` | Agent is deprecated |
| `maintenance` | Agent is under maintenance |

### AgentLifecycleState

Runtime state machine:

| Value | Description |
|-------|-------------|
| `registered` | Newly registered |
| `initializing` | Starting up |
| `ready` | Ready to accept runs |
| `idle` | No active runs |
| `running` | Executing a run |
| `waiting` | Waiting for input |
| `interrupted` | Paused for human input |
| `draining` | Completing active runs |
| `stopped` | Stopped |
| `error` | Error state |
| `failed` | Failed permanently |
| `maintenance` | Under maintenance |

### RunStatus

Run execution lifecycle:

| Value | Description |
|-------|-------------|
| `pending` | Created, not started |
| `queued` | In queue (distributed mode) |
| `running` | Actively processing |
| `streaming` | Streaming output |
| `interrupted` | Waiting for human input |
| `completed` | Successfully finished |
| `failed` | Failed with error |
| `cancelled` | Manually cancelled |
| `timeout` | Exceeded timeout |

### RunStepType

Types of execution steps:

| Value | Description |
|-------|-------------|
| `llm_call` | LLM inference |
| `tool_call` | Tool execution |
| `agent_call` | Sub-agent invocation |
| `decision` | Decision point |
| `skill_execution` | Skill execution |
| `knowledge_query` | RAG query |
| `parallel_execution` | Parallel operations |

### InterruptType

Reasons for human-in-the-loop:

| Value | Description |
|-------|-------------|
| `approval_required` | Needs approval to proceed |
| `confirmation_required` | Needs confirmation |
| `input_required` | Needs additional input |
| `clarification_required` | Needs clarification |
| `selection_required` | Needs selection from options |
| `confidence_low` | Low confidence in result |
| `conflict_detected` | Detected conflicting information |
| `error_occurred` | Error needs resolution |
| `knowledge_gap` | Missing knowledge |
| `high_risk_operation` | High-risk operation |
| `policy_violation` | Policy violation detected |
| `anomaly_detected` | Anomaly detected |

### InterruptStatus

Interrupt lifecycle:

| Value | Description |
|-------|-------------|
| `pending` | Waiting for response |
| `notified` | User notified |
| `viewed` | User viewed |
| `resolved` | Response received |
| `expired` | Timeout expired |
| `cancelled` | Cancelled |

### InterruptPriority

| Value | Description |
|-------|-------------|
| `low` | Low priority |
| `normal` | Normal priority |
| `high` | High priority |
| `urgent` | Urgent - immediate attention |

### MessageRole

Thread message roles:

| Value | Description |
|-------|-------------|
| `user` | User input |
| `assistant` | Agent response |
| `system` | System instructions |
| `tool` | Tool call results |

### ThreadStatus

Thread lifecycle:

| Value | Description |
|-------|-------------|
| `active` | Thread is active |
| `paused` | Thread is paused |
| `closed` | Thread is closed |
| `archived` | Thread is archived |

### KnowledgeDomain

Knowledge partitions:

| Value | Description |
|-------|-------------|
| `global` | Shared across all agents |
| `trading` | Trading strategies and signals |
| `banking` | Banking and finance |
| `legal` | Legal documents and compliance |
| `compliance` | Regulatory requirements |
| `accounting` | Financial records |
| `organization` | Company-specific knowledge |
| `personal` | User-specific knowledge |

### SkillAction

Operations agents can perform:

| Value | Category | Description |
|-------|----------|-------------|
| `query_clicbrain` | Knowledge | Query knowledge base |
| `search_qkos` | Knowledge | Search QKO objects |
| `create_qko` | Knowledge | Create QKO |
| `update_qko` | Knowledge | Update QKO |
| `call_mcp_tool` | External | Call MCP tool |
| `call_api` | External | Call external API |
| `call_agent` | External | Call another agent |
| `parallel_agents` | External | Call agents in parallel |
| `llm_generate` | LLM | Generate text |
| `llm_analyze` | LLM | Analyze content |
| `llm_summarize` | LLM | Summarize content |
| `dspy_rag` | DSPy | RAG operation |
| `dspy_reasoning` | DSPy | Chain-of-thought |
| `dspy_classification` | DSPy | Text classification |
| `dspy_extraction` | DSPy | Data extraction |
| `wait` | Control | Wait for event |
| `branch` | Control | Conditional branch |
| `loop` | Control | Loop execution |
| `request_input` | Human | Request user input |
| `request_approval` | Human | Request approval |

### AgentMemoryType

Types of agent memory:

| Value | Description |
|-------|-------------|
| `conversation_turn` | User/assistant exchange |
| `tool_result` | Tool execution result |
| `intermediate_result` | Intermediate computation |
| `context_summary` | Summarized context |
| `decision_point` | Key decision made |

---

## Agent Types

### CAPAgent

```typescript
interface CAPAgent {
  // Identity
  id: string;               // UUID
  agent_id: string;         // Semantic ID (e.g., "clicbrain.legal.v1")
  name: string;             // Display name
  version: string;          // Semantic version
  description?: string;

  // Classification
  system: ClicSystem;       // Which Clic system
  type: AgentType;          // Agent type
  ownership: AgentOwnership; // system or personal

  // State
  status: AgentStatus;
  lifecycle_state: AgentLifecycleState;

  // Capabilities
  capabilities?: {
    domains: KnowledgeDomain[];
    actions: SkillAction[];
    tools: string[];
    parallel_tool_calls?: boolean;
  };

  // CAP Extensions
  cap_extensions?: {
    supports_threads: boolean;
    supports_interrupts: boolean;
    supports_streaming: boolean;
    max_concurrent_runs: number;
    default_timeout_ms: number;
  };

  // Health
  health_status?: {
    status: 'healthy' | 'degraded' | 'unhealthy';
    metrics?: {
      avg_latency_ms: number;
      success_rate: number;
      active_runs: number;
    };
  };

  // Timestamps
  created_at: Date;
  updated_at: Date;
}
```

### CreateAgentInput

```typescript
interface CreateAgentInput {
  agent_id: string;
  name: string;
  version: string;
  description?: string;
  system: ClicSystem;
  type: AgentType;
  ownership?: AgentOwnership;
  capabilities?: {
    domains?: KnowledgeDomain[];
    actions?: SkillAction[];
    tools?: string[];
    parallel_tool_calls?: boolean;
  };
  cap_extensions?: {
    supports_threads?: boolean;
    supports_interrupts?: boolean;
    supports_streaming?: boolean;
    max_concurrent_runs?: number;
    default_timeout_ms?: number;
  };
}
```

---

## Run Types

### CAPRun

```typescript
interface CAPRun {
  id: string;
  run_id: string;
  agent_id: string;
  thread_id?: string;
  status: RunStatus;

  input: {
    messages: Array<{
      role: MessageRole;
      content: string;
    }>;
    context?: Record<string, unknown>;
  };

  output?: {
    response?: string;
    data?: unknown;
    artifacts?: Array<{
      name: string;
      type: string;
      content: unknown;
    }>;
    usage?: {
      input_tokens: number;
      output_tokens: number;
      total_tokens: number;
    };
    metrics?: {
      total_duration_ms: number;
      llm_duration_ms: number;
      tool_duration_ms: number;
    };
  };

  steps: RunStep[];
  error?: string;

  created_at: Date;
  started_at?: Date;
  completed_at?: Date;
}
```

### RunStep

```typescript
interface RunStep {
  id: string;
  run_id: string;
  type: RunStepType;
  name: string;
  status: 'pending' | 'running' | 'completed' | 'failed';

  input_summary?: string;
  output_summary?: string;

  // For tool calls
  tool_name?: string;
  tool_input?: Record<string, unknown>;
  tool_output?: Record<string, unknown>;

  // For agent calls
  called_agent_id?: string;
  called_run_id?: string;

  error?: Record<string, unknown>;
  duration_ms?: number;
  started_at?: Date;
  completed_at?: Date;
}
```

### RunInput

```typescript
interface RunInput {
  agent_id: string;
  thread_id?: string;
  input: {
    messages: Array<{
      role: MessageRole;
      content: string;
    }>;
    context?: Record<string, unknown>;
  };
  options?: {
    timeout_ms?: number;
    stream?: boolean;
    checkpoint_interval_ms?: number;
  };
}
```

---

## Thread Types

### Thread

```typescript
interface Thread {
  id: string;
  agent_id: string;
  status: ThreadStatus;
  messages: Message[];
  metadata: {
    user_id?: string;
    session_id?: string;
    [key: string]: unknown;
  };
  created_at: Date;
  updated_at: Date;
  closed_at?: Date;
}
```

### Message

```typescript
interface Message {
  id: string;
  thread_id: string;
  role: MessageRole;
  content: string;
  metadata?: {
    tool_call_id?: string;
    tool_name?: string;
    model?: string;
    tokens?: {
      input: number;
      output: number;
    };
  };
  created_at: Date;
}
```

---

## Interrupt Types

### Interrupt

```typescript
interface Interrupt {
  id: string;
  thread_id: string;
  run_id: string;
  agent_id: string;
  type: InterruptType;
  status: InterruptStatus;
  priority: InterruptPriority;

  payload: {
    message: string;
    proposed_action?: string;
    options?: string[];
    details?: Record<string, unknown>;
  };

  response?: {
    value: string;
    resolved_by?: string;
    metadata?: Record<string, unknown>;
  };

  timeout_ms: number;
  created_at: Date;
  expires_at: Date;
  resolved_at?: Date;
}
```

---

## Skill Types

### Skill

```typescript
interface Skill {
  id: string;
  name: string;
  description: string;
  scope: SkillScope;
  category: SkillCategory;
  complexity: SkillComplexity;

  actions: SkillAction[];
  required_domains?: KnowledgeDomain[];

  input_schema: Record<string, unknown>;
  output_schema: Record<string, unknown>;

  created_at: Date;
  updated_at: Date;
}
```

---

## Error Codes

### CAP Error Codes

| Code | Description |
|------|-------------|
| `CAP_AGENT_NOT_FOUND` | Agent not found |
| `CAP_AGENT_NOT_READY` | Agent not ready |
| `CAP_AGENT_UNHEALTHY` | Agent is unhealthy |
| `CAP_THREAD_NOT_FOUND` | Thread not found |
| `CAP_THREAD_CLOSED` | Thread is closed |
| `CAP_RUN_NOT_FOUND` | Run not found |
| `CAP_RUN_CANCELLED` | Run was cancelled |
| `CAP_RUN_TIMEOUT` | Run timed out |
| `CAP_RUN_EXECUTION_FAILED` | Run execution failed |
| `CAP_INTERRUPT_NOT_FOUND` | Interrupt not found |
| `CAP_INTERRUPT_EXPIRED` | Interrupt expired |

### Validation Error Codes

| Code | Description |
|------|-------------|
| `VALID_REQUIRED_FIELD` | Required field missing |
| `VALID_INVALID_FORMAT` | Invalid format |
| `VALID_OUT_OF_RANGE` | Value out of range |

### Network Error Codes

| Code | Description |
|------|-------------|
| `NET_CONNECTION_FAILED` | Connection failed |
| `NET_REQUEST_TIMEOUT` | Request timed out |

### RAG Error Codes

| Code | Description |
|------|-------------|
| `RAG_QUERY_FAILED` | RAG query failed |
| `RAG_NO_RESULTS` | No results found |
| `RAG_LOW_CONFIDENCE` | Low confidence result |

### Timeout Error Codes

| Code | Description |
|------|-------------|
| `TIMEOUT_OPERATION` | Operation timed out |
| `TIMEOUT_LLM_CALL` | LLM call timed out |

---

## Type Utilities

### Result Type

```typescript
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

// Usage
function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}
```

### TokenUsage

```typescript
interface TokenUsage {
  input: number;
  output: number;
  total: number;
  estimatedCost: number;
}
```

### HealthStatus

```typescript
interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  components?: Record<string, {
    status: 'healthy' | 'unhealthy';
    latency?: number;
    error?: string;
  }>;
  metrics?: {
    avg_latency_ms: number;
    success_rate: number;
    active_runs: number;
    queued_runs: number;
  };
  issues?: string[];
  updated_at: Date;
}
```
