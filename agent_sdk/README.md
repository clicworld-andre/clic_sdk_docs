# ClicBrain Agent SDK Documentation

**Version:** 8.0.0
**Last Updated:** January 2026

Welcome to the ClicBrain Agent SDK documentation. This SDK provides a comprehensive framework for building, deploying, and managing intelligent agents within the Clic ecosystem.

## Overview

The ClicBrain Agent SDK is built on **CAP (Clic Agent Protocol)**, a sophisticated multi-agent coordination system that enables:

- **Distributed Agent Execution** - Run agents locally or across distributed infrastructure
- **Multi-Agent Coordination** - Orchestrate multiple agents with conflict detection and resolution
- **Stateful Threads** - Maintain conversation and execution context across interactions
- **Human-in-the-Loop** - Built-in interrupt handling for approvals and clarifications
- **Knowledge Integration** - Seamless access to ClicBrain's knowledge base via RAG
- **Observability** - OpenTelemetry tracing, structured logging, and metrics

## Quick Start

### Installation

The Agent SDK is included in ClicBrain v4+. No additional installation required.

```typescript
import { CAPHub } from '@/cap/services/cap-hub';
import { AgentRegistry } from '@/cap/services/agent-registry';
import { createAgentWithTools } from '@/agent-sdk/agent-presets';
```

### Creating Your First Agent

```typescript
import { CreateAgentInput } from '@/cap/types/agent';

const myAgent: CreateAgentInput = {
  agent_id: 'my-company.assistant.v1',
  name: 'My Assistant Agent',
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
};

// Register the agent
const registry = new AgentRegistry();
const agent = await registry.register(myAgent);
```

## Documentation Structure

| Section | Description |
|---------|-------------|
| [Core Concepts](./core-concepts/) | CAP architecture, agents, threads, runs |
| [API Reference](./api-reference/) | Complete API documentation |
| [Step Handlers](./step-handlers/) | RAG, extraction, classification handlers |
| [Tools](./tools/) | Agent tools and presets |
| [Examples](./examples/) | Code examples and tutorials |
| [Guides](./guides/) | How-to guides and best practices |
| [Reference](./reference/) | Enums, schemas, and type definitions |

## Key Features

### 1. Agent Types

The SDK supports 24+ agent types across multiple Clic systems:

| System | Agent Types |
|--------|-------------|
| **CAP Core** | `llm`, `orchestrator`, `tool`, `hybrid` |
| **ClicBrain** | `coding`, `legal`, `compliance`, `document_processing`, `accounting`, `knowledge_manager` |
| **Clix Trading** | `trading`, `wallet`, `bot`, `bond_auction`, `letter_of_credit`, `commodities`, `analytics`, `master_orchestrator` |
| **Clic Banking** | `investment_advisor`, `transaction_advisor`, `customer_service` |
| **OneClicID** | `identity_verification`, `trust_rating` |

### 2. Knowledge Domains

Agents can access partitioned knowledge domains:

- `global` - Shared across all agents
- `trading` - Trading strategies and signals
- `banking` - Banking and finance
- `legal` - Legal documents and compliance
- `compliance` - Regulatory requirements
- `accounting` - Financial records
- `organization` - Company-specific knowledge
- `personal` - User-specific knowledge

### 3. Skill Actions

Agents can perform various actions:

```typescript
// Knowledge Operations
'query_clicbrain', 'search_qkos', 'create_qko', 'update_qko'

// External Calls
'call_mcp_tool', 'call_api', 'call_agent', 'parallel_agents'

// LLM Operations
'llm_generate', 'llm_analyze', 'llm_summarize'

// DSPy Actions
'dspy_rag', 'dspy_reasoning', 'dspy_classification', 'dspy_extraction'

// Control Flow
'wait', 'branch', 'loop'

// Human Interaction
'request_input', 'request_approval'
```

### 4. Interrupt Handling

Built-in support for human-in-the-loop scenarios:

```typescript
type InterruptType =
  | 'approval_required'
  | 'confirmation_required'
  | 'input_required'
  | 'clarification_required'
  | 'selection_required'
  | 'confidence_low'
  | 'conflict_detected'
  | 'error_occurred'
  | 'knowledge_gap'
  | 'high_risk_operation'
  | 'policy_violation'
  | 'anomaly_detected';
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CAP Hub                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Agent     │  │   Thread    │  │      Run Executor       │ │
│  │  Registry   │  │  Manager    │  │  (Local/Distributed)    │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │  Interrupt  │  │    CAP      │  │    Step Handler         │ │
│  │  Handler    │  │ Persistence │  │      Registry           │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
    ┌──────▼──────┐   ┌───────▼───────┐  ┌──────▼──────┐
    │   Agents    │   │  Step Handlers │  │    Tools    │
    │  (24+ types)│   │  (RAG, DSPy)  │  │  (9+ tools) │
    └─────────────┘   └───────────────┘  └─────────────┘
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/cap/agents` | GET | List all agents |
| `/api/cap/agents` | POST | Create new agent |
| `/api/cap/agents/:id` | GET | Get agent by ID |
| `/api/cap/agents/:id` | PUT | Update agent |
| `/api/cap/agents/:id` | DELETE | Delete agent |
| `/api/cap/agents/:id/health` | GET | Get agent health |
| `/api/cap/threads` | POST | Create thread |
| `/api/cap/threads/:id/runs` | POST | Execute run |

## Observability

The SDK includes built-in observability:

- **OpenTelemetry Tracing** - Distributed tracing across agent calls
- **Structured Logging** - JSON-formatted logs with context
- **Metrics Collection** - Latency, success rates, queue depths

```typescript
// Tracing is automatic for all agent operations
import { CAPTracer } from '@/cap/observability/cap-tracing';

// Custom spans
const span = CAPTracer.startSpan('my-operation');
try {
  // ... operation
} finally {
  span.end();
}
```

## Version History

| Version | Release | Key Features |
|---------|---------|--------------|
| 8.0.0 | Jan 2026 | Skills DSPy Integration, Test Infrastructure |
| 7.3.0 | Dec 2025 | CAP Phase 3 - Advanced Features & Optimization |
| 7.2.0 | Dec 2025 | Agent Versioning, Deployment Strategies |
| 7.1.0 | Dec 2025 | Distributed Mode, OpenTelemetry |
| 7.0.0 | Nov 2025 | Write-through Persistence, Hardened Hub |

## Support

- **Documentation**: This repository
- **Issues**: [GitHub Issues](https://github.com/clic-world/clicbrain/issues)
- **Email**: developer@clic.world

## License

Proprietary - Clic World Ltd. All rights reserved.
