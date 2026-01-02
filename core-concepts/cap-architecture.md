# CAP Architecture

**CAP (Clic Agent Protocol)** is the foundational architecture for building and orchestrating intelligent agents in the ClicBrain ecosystem.

## Overview

CAP provides a standardized way to:
- Define and register agents
- Manage stateful threads
- Execute agent runs with step handling
- Handle human-in-the-loop interrupts
- Persist state across restarts

## Components

### CAP Hub

The central coordination point for all CAP services.

```typescript
import { CAPHub } from '@/cap/services/cap-hub';

// Initialize the hub
const hub = new CAPHub({
  persistence: {
    enabled: true,
    connectionString: process.env.DATABASE_URL,
  },
  distributed: {
    enabled: false, // Enable for BullMQ distributed mode
    redis: process.env.REDIS_URL,
  },
});

await hub.initialize();
```

**Responsibilities:**
- Orchestrates all CAP services
- Manages agent lifecycle
- Coordinates thread execution
- Handles graceful shutdown
- Database persistence

### Agent Registry

Manages agent registration, discovery, and health monitoring.

```typescript
import { AgentRegistry } from '@/cap/services/agent-registry';

const registry = new AgentRegistry();

// Register an agent
const agent = await registry.register({
  agent_id: 'my-agent',
  name: 'My Agent',
  version: '1.0.0',
  system: 'clicbrain',
  type: 'llm',
  // ...
});

// Discover agents
const agents = await registry.discover({
  system: 'clicbrain',
  type: 'llm',
  status: 'active',
});

// Health check
const health = await registry.getHealthStatus('my-agent');
```

**Features:**
- Write-through persistence (v7.0.0)
- EventEmitter-based lifecycle events
- Automatic health checks (30-second intervals)
- Max 100 agents per system (configurable)

### Thread Manager

Manages stateful conversation threads.

```typescript
import { ThreadManager } from '@/cap/services/thread-manager';

const threadManager = new ThreadManager();

// Create a thread
const thread = await threadManager.create({
  agent_id: 'my-agent',
  metadata: {
    user_id: 'user-123',
    session_id: 'session-456',
  },
});

// Add messages
await threadManager.addMessage(thread.id, {
  role: 'user',
  content: 'Hello, agent!',
});

// Get thread history
const messages = await threadManager.getMessages(thread.id);
```

### Run Executor

Executes agent runs with full lifecycle management.

```typescript
import { RunExecutor } from '@/cap/services/run-executor';

const executor = new RunExecutor({
  mode: 'local', // or 'distributed' for BullMQ
});

// Execute a run
const run = await executor.execute({
  agent_id: 'my-agent',
  thread_id: thread.id,
  input: {
    messages: [{ role: 'user', content: 'Help me analyze this data' }],
  },
});

// Stream results
for await (const event of run.stream()) {
  console.log(event);
}
```

**Execution Modes:**

| Mode | Description | Use Case |
|------|-------------|----------|
| `local` | In-process execution | Development, single-node |
| `distributed` | BullMQ queue-based | Production, multi-node |

### Interrupt Handler

Handles human-in-the-loop scenarios.

```typescript
import { InterruptHandler } from '@/cap/services/interrupt-handler';

const interruptHandler = new InterruptHandler();

// Create an interrupt
const interrupt = await interruptHandler.create({
  run_id: run.id,
  type: 'approval_required',
  priority: 'high',
  message: 'Approve this action?',
  options: ['Approve', 'Reject'],
  timeout_ms: 300000, // 5 minutes
});

// Resolve the interrupt
await interruptHandler.resolve(interrupt.id, {
  response: 'Approve',
  resolved_by: 'user-123',
});
```

### CAP Persistence

Manages database persistence for all CAP entities.

```typescript
import { CAPPersistence } from '@/cap/services/cap-persistence';

const persistence = new CAPPersistence({
  connectionString: process.env.DATABASE_URL,
});

await persistence.connect();

// Persistence is automatic for:
// - Agent registration/updates
// - Thread creation/messages
// - Run state checkpoints
// - Interrupt lifecycle
```

## Event System

CAP uses EventEmitter for coordination:

```typescript
// Agent events
registry.on('agent:registered', (agent) => console.log('Registered:', agent.name));
registry.on('agent:updated', (agent) => console.log('Updated:', agent.name));
registry.on('agent:removed', (agentId) => console.log('Removed:', agentId));
registry.on('agent:health_changed', ({ agentId, health }) => {
  console.log(`Health changed: ${agentId} -> ${health.status}`);
});

// Run events
executor.on('run:started', (run) => console.log('Started:', run.id));
executor.on('run:completed', (run) => console.log('Completed:', run.id));
executor.on('run:failed', (run, error) => console.log('Failed:', error));
executor.on('run:interrupted', (run, interrupt) => console.log('Interrupted:', interrupt.type));
```

## Configuration

### Environment Variables

```bash
# Database
DATABASE_URL=postgresql://user:pass@localhost:5435/clicbrain

# Redis (for distributed mode)
REDIS_URL=redis://localhost:6379

# CAP Settings
CAP_MAX_AGENTS=100
CAP_HEALTH_CHECK_INTERVAL_MS=30000
CAP_DEFAULT_TIMEOUT_MS=300000
CAP_DISTRIBUTED_MODE=false
```

### Programmatic Configuration

```typescript
const hub = new CAPHub({
  // Persistence settings
  persistence: {
    enabled: true,
    connectionString: process.env.DATABASE_URL,
    schema: 'public',
  },

  // Distributed settings
  distributed: {
    enabled: process.env.CAP_DISTRIBUTED_MODE === 'true',
    redis: process.env.REDIS_URL,
    queues: {
      runs: 'cap:runs',
      interrupts: 'cap:interrupts',
    },
  },

  // Health check settings
  healthCheck: {
    intervalMs: 30000,
    timeoutMs: 5000,
    unhealthyThreshold: 3,
  },

  // Run settings
  runs: {
    defaultTimeoutMs: 300000,
    maxConcurrentPerAgent: 10,
    checkpointIntervalMs: 10000,
  },
});
```

## Lifecycle

### Startup

```typescript
// 1. Initialize hub
await hub.initialize();

// 2. Load persisted agents
// (automatic with persistence enabled)

// 3. Start health checks
// (automatic)

// 4. Ready to accept requests
console.log('CAP Hub ready');
```

### Shutdown

```typescript
// Graceful shutdown
process.on('SIGTERM', async () => {
  // 1. Stop accepting new runs
  await hub.drain();

  // 2. Wait for active runs to complete
  await hub.waitForCompletion({ timeoutMs: 30000 });

  // 3. Persist final state
  await hub.checkpoint();

  // 4. Close connections
  await hub.shutdown();
});
```

## Best Practices

### 1. Always Enable Persistence in Production

```typescript
const hub = new CAPHub({
  persistence: { enabled: true },
});
```

### 2. Use Distributed Mode for Scale

```typescript
const hub = new CAPHub({
  distributed: {
    enabled: true,
    redis: process.env.REDIS_URL,
  },
});
```

### 3. Set Appropriate Timeouts

```typescript
const agent = {
  cap_extensions: {
    default_timeout_ms: 60000, // 1 minute for fast operations
    max_concurrent_runs: 5,    // Limit concurrent runs
  },
};
```

### 4. Handle Interrupts Gracefully

```typescript
run.on('interrupt', async (interrupt) => {
  // Notify user via appropriate channel
  await notifyUser(interrupt);

  // Set reasonable timeout
  setTimeout(() => {
    if (interrupt.status === 'pending') {
      interruptHandler.expire(interrupt.id);
    }
  }, interrupt.timeout_ms);
});
```
