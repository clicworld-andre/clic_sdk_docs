# Agents

Agents are the core building blocks of the ClicBrain Agent SDK. Each agent is a self-contained unit with defined capabilities, lifecycle management, and health monitoring.

## Agent Schema

```typescript
interface CAPAgent {
  // Identity
  id: string;              // Database UUID
  agent_id: string;        // Unique identifier (e.g., 'clicbrain.legal.v1')
  name: string;            // Human-readable name
  version: string;         // Semantic version (e.g., '1.0.0')
  description?: string;    // Agent description

  // Classification
  system: ClicSystem;      // Which Clic system
  type: AgentType;         // Agent type/role
  ownership: 'system' | 'personal';

  // Capabilities
  capabilities: AgentCapabilities;
  input_schema?: object;   // JSON Schema for input validation
  output_schema?: object;  // JSON Schema for output validation

  // CAP Extensions
  cap_extensions: CAPExtensions;
  endpoints?: AgentEndpoints;

  // Status
  status: 'active' | 'inactive' | 'deprecated' | 'maintenance';
  lifecycle_state: AgentLifecycleState;

  // Health
  last_heartbeat?: Date;
  health_status?: AgentHealthStatus;

  // Metadata
  created_at: Date;
  updated_at: Date;
  created_by?: string;
}
```

## Agent Types

### CAP Core Types

| Type | Description | Use Case |
|------|-------------|----------|
| `llm` | LLM-powered agent | General AI tasks, conversation |
| `orchestrator` | Multi-agent coordinator | Complex workflows |
| `tool` | Tool-based agent | API integrations, actions |
| `hybrid` | Combination | LLM + tools |

### ClicBrain Types

| Type | Description | Domain |
|------|-------------|--------|
| `coding` | Code generation and analysis | Development |
| `legal` | Legal document processing | Legal |
| `compliance` | Regulatory compliance | Compliance |
| `document_processing` | Document extraction | Documents |
| `accounting` | Financial processing | Finance |
| `knowledge_manager` | Knowledge curation | Knowledge |

### Clix Trading Types

| Type | Description | Domain |
|------|-------------|--------|
| `trading` | Trading operations | Trading |
| `wallet` | Wallet management | Finance |
| `bot` | Trading bots | Automation |
| `bond_auction` | Bond auctions | Fixed income |
| `letter_of_credit` | LC processing | Trade finance |
| `commodities` | Commodity trading | Commodities |
| `analytics` | Analytics and reporting | Analytics |
| `master_orchestrator` | Platform orchestration | Coordination |

## Agent Capabilities

```typescript
interface AgentCapabilities {
  // Knowledge domains agent can access
  domains: KnowledgeDomain[];

  // Actions agent can perform
  actions: SkillAction[];

  // MCP tools agent can use
  tools: string[];

  // Other agents this agent can invoke
  invoke_agents: string[];

  // Maximum context window size
  max_context_tokens?: number;

  // Supports parallel tool execution
  parallel_tool_calls: boolean;
}
```

### Knowledge Domains

```typescript
type KnowledgeDomain =
  | 'global'       // Shared across all agents
  | 'trading'      // Trading strategies, signals
  | 'banking'      // Banking, finance
  | 'legal'        // Legal documents, compliance
  | 'compliance'   // Regulatory requirements
  | 'accounting'   // Financial records
  | 'organization' // Company-specific
  | 'personal';    // User-specific
```

### Skill Actions

```typescript
type SkillAction =
  // Knowledge operations
  | 'query_clicbrain'
  | 'search_qkos'
  | 'create_qko'
  | 'update_qko'

  // External calls
  | 'call_mcp_tool'
  | 'call_api'
  | 'call_agent'
  | 'parallel_agents'

  // LLM operations
  | 'llm_generate'
  | 'llm_analyze'
  | 'llm_summarize'

  // DSPy actions
  | 'dspy_rag'
  | 'dspy_reasoning'
  | 'dspy_classification'
  | 'dspy_extraction'

  // Control flow
  | 'wait'
  | 'branch'
  | 'loop'

  // Human interaction
  | 'request_input'
  | 'request_approval';
```

## CAP Extensions

```typescript
interface CAPExtensions {
  // Feature support
  supports_threads: boolean;      // Stateful threads
  supports_interrupts: boolean;   // Human-in-the-loop
  supports_streaming: boolean;    // Streaming responses
  clicbrain_integration: boolean; // Knowledge access

  // Requirements
  required_skills: string[];      // Required skill IDs

  // Limits
  max_concurrent_runs: number;    // Max parallel runs
  default_timeout_ms: number;     // Default timeout

  // Approval
  requires_approval: boolean;     // Requires human approval
}
```

## Agent Lifecycle

```
┌──────────────┐
│  registered  │  Initial state after registration
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ initializing │  Agent is initializing resources
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    ready     │  Agent is ready but idle
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│     idle     │◄────│   running    │  Processing runs
└──────┬───────┘     └──────┬───────┘
       │                    │
       │             ┌──────▼───────┐
       │             │   waiting    │  Waiting for interrupt
       │             └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│   draining   │────►│   stopped    │  Graceful shutdown
└──────────────┘     └──────────────┘

Error states: error, failed, maintenance
```

### Lifecycle States

| State | Description |
|-------|-------------|
| `registered` | Agent registered but not initialized |
| `initializing` | Agent is starting up |
| `ready` | Agent ready to accept runs |
| `idle` | Agent has no active runs |
| `running` | Agent is processing runs |
| `waiting` | Agent waiting for interrupt response |
| `interrupted` | Agent run was interrupted |
| `draining` | Agent stopping, completing active runs |
| `stopped` | Agent fully stopped |
| `error` | Agent encountered recoverable error |
| `failed` | Agent failed and cannot recover |
| `maintenance` | Agent in maintenance mode |

## Creating Agents

### Basic Agent

```typescript
import { AgentRegistry } from '@/cap/services/agent-registry';

const registry = new AgentRegistry();

const agent = await registry.register({
  agent_id: 'my-company.assistant.v1',
  name: 'My Assistant',
  version: '1.0.0',
  description: 'A helpful assistant agent',
  system: 'clicbrain',
  type: 'llm',
  capabilities: {
    domains: ['global'],
    actions: ['llm_generate', 'query_clicbrain'],
    tools: [],
    parallel_tool_calls: true,
  },
  cap_extensions: {
    supports_threads: true,
    supports_interrupts: true,
    supports_streaming: true,
    max_concurrent_runs: 10,
    default_timeout_ms: 300000,
  },
});
```

### Agent with Tools

```typescript
const toolAgent = await registry.register({
  agent_id: 'my-company.analyst.v1',
  name: 'Data Analyst',
  version: '1.0.0',
  system: 'clicbrain',
  type: 'hybrid',
  capabilities: {
    domains: ['trading', 'analytics'],
    actions: [
      'llm_generate',
      'llm_analyze',
      'query_clicbrain',
      'call_mcp_tool',
    ],
    tools: [
      'search_knowledge',
      'get_trading_signals',
      'store_insight',
    ],
    invoke_agents: ['trading-agent'],
    parallel_tool_calls: true,
  },
  cap_extensions: {
    supports_threads: true,
    supports_interrupts: true,
    supports_streaming: true,
    required_skills: ['market-analysis', 'report-generation'],
    max_concurrent_runs: 5,
    default_timeout_ms: 600000,
    requires_approval: false,
  },
});
```

### Orchestrator Agent

```typescript
const orchestrator = await registry.register({
  agent_id: 'my-company.orchestrator.v1',
  name: 'Workflow Orchestrator',
  version: '1.0.0',
  system: 'clicbrain',
  type: 'orchestrator',
  capabilities: {
    domains: ['global', 'trading', 'analytics'],
    actions: [
      'call_agent',
      'parallel_agents',
      'branch',
      'loop',
      'wait',
    ],
    tools: [],
    invoke_agents: [
      'trading-agent',
      'analytics-agent',
      'reporting-agent',
    ],
    parallel_tool_calls: true,
  },
  cap_extensions: {
    supports_threads: true,
    supports_interrupts: true,
    supports_streaming: true,
    max_concurrent_runs: 3,
    default_timeout_ms: 1800000, // 30 minutes
  },
});
```

## Agent Health

### Health Status

```typescript
interface AgentHealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy' | 'unknown';

  components: {
    [name: string]: {
      status: 'healthy' | 'degraded' | 'unhealthy';
      message?: string;
      last_check?: Date;
    };
  };

  metrics?: {
    avg_latency_ms?: number;
    success_rate?: number;
    active_runs?: number;
    queued_runs?: number;
  };

  issues: string[];
  updated_at?: Date;
}
```

### Monitoring Health

```typescript
// Get current health
const health = await registry.getHealthStatus(agent.agent_id);

// Subscribe to health changes
registry.on('agent:health_changed', ({ agentId, health }) => {
  if (health.status === 'unhealthy') {
    alertOps(`Agent ${agentId} is unhealthy: ${health.issues.join(', ')}`);
  }
});

// Custom health check endpoint
app.get('/api/agents/:id/health', async (req, res) => {
  const health = await registry.getHealthStatus(req.params.id);
  res.status(health.status === 'healthy' ? 200 : 503).json(health);
});
```

## Agent Versioning

### Semantic Versioning

```typescript
interface AgentVersion {
  major: number;  // Breaking changes
  minor: number;  // New features, backward compatible
  patch: number;  // Bug fixes

  min_compatible?: string;  // Minimum compatible version
  deprecated_at?: Date;     // Deprecation date
  sunset_at?: Date;         // End-of-life date
  prerelease?: string;      // e.g., 'alpha', 'beta', 'rc.1'
  build?: string;           // Build metadata
}
```

### Version Helpers

```typescript
import {
  parseVersion,
  formatVersion,
  compareVersions,
  satisfiesMinVersion,
} from '@/cap/types/agent';

// Parse version string
const v = parseVersion('2.1.0-beta');
// { major: 2, minor: 1, patch: 0, prerelease: 'beta' }

// Format version object
const str = formatVersion(v);
// '2.1.0-beta'

// Compare versions
compareVersions('2.0.0', '1.9.9'); // 1 (greater)
compareVersions('1.0.0', '1.0.0'); // 0 (equal)
compareVersions('1.0.0', '2.0.0'); // -1 (less)

// Check compatibility
satisfiesMinVersion('2.1.0', '2.0.0'); // true
satisfiesMinVersion('1.9.0', '2.0.0'); // false
```

## Agent Discovery

```typescript
// Discover agents by criteria
const agents = await registry.discover({
  system: 'clicbrain',
  type: 'llm',
  status: 'active',
  domain: 'trading',
});

// Find agents with specific capability
const ragAgents = await registry.discover({
  action: 'dspy_rag',
});

// Find agents with specific tool
const analysisAgents = await registry.discover({
  has_tool: 'search_knowledge',
});
```

## Best Practices

### 1. Use Descriptive Agent IDs

```typescript
// Good: includes organization, purpose, version
'acme-corp.trading-assistant.v2'
'clicbrain.legal.contract-analyzer.v1'

// Bad: vague or too generic
'agent1'
'my-agent'
```

### 2. Define Clear Capabilities

```typescript
capabilities: {
  // Be specific about domains
  domains: ['trading', 'analytics'], // Not just ['global']

  // Only include actions the agent actually uses
  actions: ['llm_generate', 'query_clicbrain'],

  // List specific tools
  tools: ['search_knowledge', 'get_trading_signals'],
}
```

### 3. Set Appropriate Limits

```typescript
cap_extensions: {
  // Reasonable concurrent run limit
  max_concurrent_runs: 10,

  // Timeout based on expected operation time
  default_timeout_ms: 60000, // 1 minute for simple tasks

  // Enable features only if needed
  supports_interrupts: true,
}
```

### 4. Version Your Agents

```typescript
// Start with 1.0.0
version: '1.0.0',

// Increment for changes
// - Bug fix: 1.0.1
// - New feature: 1.1.0
// - Breaking change: 2.0.0
```
