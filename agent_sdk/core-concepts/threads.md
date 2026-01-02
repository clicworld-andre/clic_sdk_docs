# Threads

Threads provide stateful conversation management for agent interactions. They maintain context across multiple runs and enable continuous, coherent conversations.

## Thread Schema

```typescript
interface Thread {
  id: string;              // UUID
  agent_id: string;        // Associated agent
  status: ThreadStatus;    // active, paused, closed, archived
  messages: Message[];     // Conversation history
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

## Thread Status

| Status | Description |
|--------|-------------|
| `active` | Thread is active and accepting runs |
| `paused` | Thread is temporarily paused |
| `closed` | Thread is closed, no new runs |
| `archived` | Thread is archived for retention |

## Messages

### Message Schema

```typescript
interface Message {
  id: string;
  thread_id: string;
  role: 'user' | 'assistant' | 'system' | 'tool';
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

### Message Roles

| Role | Description |
|------|-------------|
| `user` | User input |
| `assistant` | Agent response |
| `system` | System instructions |
| `tool` | Tool call results |

## Creating Threads

```typescript
import { ThreadManager } from '@/cap/services/thread-manager';

const threadManager = new ThreadManager();

// Create a simple thread
const thread = await threadManager.create({
  agent_id: 'my-agent',
});

// Create a thread with metadata
const threadWithMeta = await threadManager.create({
  agent_id: 'my-agent',
  metadata: {
    user_id: 'user-123',
    session_id: 'session-456',
    channel: 'web',
    language: 'en',
  },
});

// Create a thread with initial system message
const threadWithSystem = await threadManager.create({
  agent_id: 'my-agent',
  initialMessages: [
    {
      role: 'system',
      content: 'You are a helpful trading assistant.',
    },
  ],
});
```

## Managing Messages

### Adding Messages

```typescript
// Add user message
await threadManager.addMessage(thread.id, {
  role: 'user',
  content: 'What is the current BTC price?',
});

// Add assistant message
await threadManager.addMessage(thread.id, {
  role: 'assistant',
  content: 'The current BTC price is $45,000.',
  metadata: {
    model: 'claude-3-opus',
    tokens: { input: 50, output: 20 },
  },
});

// Add tool result
await threadManager.addMessage(thread.id, {
  role: 'tool',
  content: JSON.stringify({ price: 45000, currency: 'USD' }),
  metadata: {
    tool_call_id: 'call_123',
    tool_name: 'get_price',
  },
});
```

### Retrieving Messages

```typescript
// Get all messages
const messages = await threadManager.getMessages(thread.id);

// Get messages with pagination
const recentMessages = await threadManager.getMessages(thread.id, {
  limit: 10,
  offset: 0,
  order: 'desc',
});

// Get messages by role
const userMessages = await threadManager.getMessages(thread.id, {
  role: 'user',
});
```

## Thread Operations

### Update Thread

```typescript
// Update metadata
await threadManager.update(thread.id, {
  metadata: {
    ...thread.metadata,
    escalated: true,
    escalated_at: new Date(),
  },
});

// Pause thread
await threadManager.updateStatus(thread.id, 'paused');

// Resume thread
await threadManager.updateStatus(thread.id, 'active');
```

### Close Thread

```typescript
// Close the thread
await threadManager.close(thread.id);

// Close with summary
await threadManager.close(thread.id, {
  summary: 'User inquiry about BTC price resolved.',
  resolution: 'completed',
});
```

### Archive Thread

```typescript
// Archive for retention
await threadManager.archive(thread.id);

// Archive with retention policy
await threadManager.archive(thread.id, {
  retentionDays: 365,
  reason: 'compliance',
});
```

## Thread Context

### Context Window Management

```typescript
// Get context for LLM
const context = await threadManager.getContext(thread.id, {
  maxTokens: 8000,
  strategy: 'recent', // or 'summary', 'hybrid'
});

// Summarize older messages
await threadManager.summarize(thread.id, {
  threshold: 50, // Summarize when > 50 messages
  keepRecent: 10, // Keep last 10 messages intact
});
```

### Context Strategies

| Strategy | Description |
|----------|-------------|
| `recent` | Include most recent messages up to token limit |
| `summary` | Summarize older messages, include recent verbatim |
| `hybrid` | Combine summary with key messages |

## Thread with Runs

### Creating Runs on Threads

```typescript
import { RunExecutor } from '@/cap/services/run-executor';

const executor = new RunExecutor();

// Execute a run on a thread
const run = await executor.execute({
  agent_id: 'my-agent',
  thread_id: thread.id,
  input: {
    messages: [
      { role: 'user', content: 'Analyze my portfolio' },
    ],
  },
});

// Messages are automatically added to thread
```

### Thread Continuation

```typescript
// Continue conversation
const thread = await threadManager.get(threadId);

// Add new user message
await threadManager.addMessage(thread.id, {
  role: 'user',
  content: 'Tell me more about the risks',
});

// Execute new run
const followUp = await executor.execute({
  agent_id: thread.agent_id,
  thread_id: thread.id,
  input: {
    messages: [], // Will use thread history
  },
});
```

## Thread Events

```typescript
// Subscribe to thread events
threadManager.on('thread:created', (thread) => {
  console.log('Thread created:', thread.id);
});

threadManager.on('thread:message', ({ threadId, message }) => {
  console.log(`New message in ${threadId}:`, message.content);
});

threadManager.on('thread:closed', (thread) => {
  console.log('Thread closed:', thread.id);
  // Save transcript, send feedback request, etc.
});
```

## Thread Persistence

Threads are automatically persisted to the database when persistence is enabled:

```typescript
// Threads are persisted automatically
const thread = await threadManager.create({ agent_id: 'my-agent' });

// Retrieve persisted thread
const retrieved = await threadManager.get(thread.id);

// List threads for user
const userThreads = await threadManager.list({
  metadata: { user_id: 'user-123' },
  status: 'active',
});
```

## Thread Memory Types

```typescript
type AgentMemoryType =
  | 'conversation_turn'    // User/assistant exchange
  | 'tool_result'          // Tool execution result
  | 'intermediate_result'  // Intermediate computation
  | 'context_summary'      // Summarized context
  | 'decision_point';      // Key decision made
```

## Best Practices

### 1. Use Metadata for Context

```typescript
await threadManager.create({
  agent_id: 'my-agent',
  metadata: {
    user_id: 'user-123',
    session_id: 'session-456',
    channel: 'web',
    intent: 'portfolio-analysis',
  },
});
```

### 2. Manage Context Window

```typescript
// Summarize long conversations
if (messages.length > 50) {
  await threadManager.summarize(thread.id, {
    keepRecent: 10,
  });
}
```

### 3. Close Threads Properly

```typescript
// Always close threads when done
await threadManager.close(thread.id, {
  summary: 'Conversation summary',
  resolution: 'completed',
});
```

### 4. Handle Thread Limits

```typescript
// Check thread count per user
const userThreads = await threadManager.list({
  metadata: { user_id: userId },
  status: 'active',
});

if (userThreads.length >= MAX_ACTIVE_THREADS) {
  // Close oldest thread or notify user
}
```
