# Agent Tools & Presets

The ClicBrain Agent SDK provides ready-to-use tools and pre-configured agent presets for common operations within the Clic ecosystem.

## Overview

The tools system provides:

- **RAG Tools** - Knowledge retrieval and storage operations
- **Agent Presets** - Pre-configured agents for each domain
- **LangChain Integration** - Compatible tool adapters
- **OpenAI Function Calling** - Standard function definitions

---

## RAG Tools

### Tool Definitions

Tools are defined in OpenAI Function Calling format:

```typescript
import { RAG_TOOL_DEFINITIONS } from '@/agent-sdk/agent-tools';
```

### Available Tools

#### search_knowledge

Search ClicBrain knowledge base for relevant information.

```typescript
{
  name: 'search_knowledge',
  description: 'Search ClicBrain knowledge base for relevant information',
  parameters: {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        description: 'The search query to find relevant knowledge',
      },
      domain: {
        type: 'string',
        enum: ['trading', 'wallet', 'bot', 'bond_auction', 'letter_of_credit', 'commodities', 'analytics'],
        description: 'Optional: specific domain to search in',
      },
      max_results: {
        type: 'number',
        description: 'Maximum number of results to return (default: 5)',
      },
    },
    required: ['query'],
  },
}
```

#### recall_user_context

Recall previous interactions and preferences for a user.

```typescript
{
  name: 'recall_user_context',
  description: 'Recall previous interactions, preferences, and context for a user',
  parameters: {
    type: 'object',
    properties: {
      user_id: {
        type: 'string',
        description: 'The user ID to recall context for',
      },
      context_type: {
        type: 'string',
        enum: ['preferences', 'history', 'portfolio', 'all'],
        description: 'Type of context to recall',
      },
    },
    required: ['user_id'],
  },
}
```

#### store_insight

Store learned insights for future reference.

```typescript
{
  name: 'store_insight',
  description: 'Store a learned insight or important information',
  parameters: {
    type: 'object',
    properties: {
      title: {
        type: 'string',
        description: 'Brief title for the insight',
      },
      content: {
        type: 'string',
        description: 'The insight content to store',
      },
      tags: {
        type: 'array',
        items: { type: 'string' },
        description: 'Tags for categorization',
      },
      user_id: {
        type: 'string',
        description: 'Optional: associate with specific user',
      },
    },
    required: ['title', 'content'],
  },
}
```

#### get_trading_signals

Retrieve trading signals and market insights.

```typescript
{
  name: 'get_trading_signals',
  description: 'Retrieve recent trading signals and market insights',
  parameters: {
    type: 'object',
    properties: {
      trading_pair: {
        type: 'string',
        description: 'Trading pair (e.g., BTC/USDT)',
      },
      signal_type: {
        type: 'string',
        enum: ['buy', 'sell', 'hold', 'all'],
        description: 'Type of signals to retrieve',
      },
      timeframe: {
        type: 'string',
        enum: ['1h', '4h', '1d', '1w'],
        description: 'Timeframe for signals',
      },
    },
    required: [],
  },
}
```

#### get_error_resolution

Find resolution steps for known errors.

```typescript
{
  name: 'get_error_resolution',
  description: 'Find resolution steps for known errors',
  parameters: {
    type: 'object',
    properties: {
      error_code: {
        type: 'string',
        description: 'The error code to look up',
      },
      error_message: {
        type: 'string',
        description: 'The error message for context',
      },
    },
    required: ['error_code'],
  },
}
```

#### get_strategy_performance

Get historical performance data for trading strategies.

```typescript
{
  name: 'get_strategy_performance',
  description: 'Retrieve historical performance for trading bot strategies',
  parameters: {
    type: 'object',
    properties: {
      strategy_type: {
        type: 'string',
        enum: ['grid', 'dca', 'arbitrage', 'momentum', 'mean_reversion'],
        description: 'Type of strategy',
      },
      trading_pair: {
        type: 'string',
        description: 'Trading pair',
      },
      min_roi: {
        type: 'number',
        description: 'Minimum ROI filter',
      },
    },
    required: [],
  },
}
```

#### get_bond_auction_history

Retrieve historical bond auction data.

```typescript
{
  name: 'get_bond_auction_history',
  description: 'Retrieve historical bond auction data including yields',
  parameters: {
    type: 'object',
    properties: {
      issuer: {
        type: 'string',
        description: 'Bond issuer (e.g., Bank of Uganda)',
      },
      maturity: {
        type: 'number',
        description: 'Bond maturity in years',
      },
      limit: {
        type: 'number',
        description: 'Number of auctions to retrieve',
      },
    },
    required: [],
  },
}
```

#### get_commodity_price

Get commodity prices with grade and region info.

```typescript
{
  name: 'get_commodity_price',
  description: 'Get current and historical commodity prices',
  parameters: {
    type: 'object',
    properties: {
      commodity: {
        type: 'string',
        description: 'Commodity name (e.g., coffee, gold)',
      },
      grade: {
        type: 'string',
        description: 'Quality grade',
      },
      region: {
        type: 'string',
        description: 'Geographic region',
      },
    },
    required: ['commodity'],
  },
}
```

#### get_ucp_rules

Retrieve UCP 600 rules for Letter of Credit processing.

```typescript
{
  name: 'get_ucp_rules',
  description: 'Retrieve UCP 600 rules and guidelines',
  parameters: {
    type: 'object',
    properties: {
      article: {
        type: 'number',
        description: 'UCP 600 article number',
      },
      topic: {
        type: 'string',
        description: 'Topic to search for',
      },
    },
    required: [],
  },
}
```

---

## RAG Tool Executor

The `RAGToolExecutor` class executes tools against the RAG client:

```typescript
import { RAGToolExecutor, RAG_TOOL_DEFINITIONS } from '@/agent-sdk/agent-tools';
import { RAGClient } from '@/agent-sdk/rag-client';

// Create client and executor
const client = new RAGClient({ agentId: 'my-agent' });
const executor = new RAGToolExecutor(client);

// Execute a tool
const result = await executor.execute('search_knowledge', {
  query: 'BTC trading patterns',
  domain: 'trading',
  max_results: 5,
});

console.log('Results:', result.knowledge);
console.log('Insights:', result.insights);
```

### Available Methods

```typescript
class RAGToolExecutor {
  // Execute any tool by name
  execute(toolName: string, args: Record<string, any>): Promise<any>;

  // Individual tool methods
  searchKnowledge(args: { query: string; domain?: RAGDomain; max_results?: number }): Promise<RAGQueryResult>;
  recallUserContext(args: { user_id: string; context_type?: string }): Promise<RAGQueryResult>;
  storeInsight(args: { title: string; content: string; tags?: string[]; user_id?: string }): Promise<{ success: boolean }>;
  getTradingSignals(args: { trading_pair?: string; signal_type?: string; timeframe?: string }): Promise<RAGQueryResult>;
  getErrorResolution(args: { error_code: string; error_message?: string }): Promise<RAGQueryResult>;
  getStrategyPerformance(args: { strategy_type?: string; trading_pair?: string; min_roi?: number }): Promise<RAGQueryResult>;
  getBondAuctionHistory(args: { issuer?: string; maturity?: number; limit?: number }): Promise<RAGQueryResult>;
  getCommodityPrice(args: { commodity: string; grade?: string; region?: string }): Promise<RAGQueryResult>;
  getUCPRules(args: { article?: number; topic?: string }): Promise<RAGQueryResult>;
}
```

---

## Agent Presets

Pre-configured agent setups for each domain:

### Available Presets

| Preset | Domain | Description |
|--------|--------|-------------|
| `trading` | trading | Trading operations and signals |
| `wallet` | wallet | Transaction and wallet management |
| `bot` | bot | Trading bot configuration |
| `bond` | bond_auction | Bond auction analysis |
| `lc` | letter_of_credit | LC document processing |
| `commodities` | commodities | Commodity pricing and info |
| `analytics` | analytics | Reporting and analysis |
| `master` | all | Multi-domain orchestration |

### Preset Structure

```typescript
interface AgentPreset {
  // Configuration
  config: {
    agentId: string;
    agentName: string;
    primaryDomain: RAGDomain;
    secondaryDomains?: RAGDomain[];
    confidenceThreshold: number;
    maxResults: number;
  };

  // Few-shot examples for RAG usage
  fewShotExamples: Array<{
    userQuery: string;
    ragQuery: string;
    usage: string;
  }>;

  // System prompt enhancement
  systemPromptAddition: string;

  // Recommended tools
  recommendedTools: string[];
}
```

### Trading Agent Preset

```typescript
import { TRADING_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'trading-agent',
  agentName: 'CLIX Trading Agent',
  primaryDomain: 'trading',
  secondaryDomains: ['analytics'],
  confidenceThreshold: 0.7,
  maxResults: 5,
}

// Recommended tools
['search_knowledge', 'get_trading_signals', 'store_insight']
```

### Wallet Agent Preset

```typescript
import { WALLET_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'wallet-agent',
  agentName: 'CLIX Wallet Agent',
  primaryDomain: 'wallet',
  secondaryDomains: ['analytics'],
  confidenceThreshold: 0.8,
  maxResults: 5,
}

// Recommended tools
['search_knowledge', 'get_error_resolution', 'recall_user_context']
```

### Bot Agent Preset

```typescript
import { BOT_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'bot-agent',
  agentName: 'CLIX Bot Agent',
  primaryDomain: 'bot',
  secondaryDomains: ['trading', 'analytics'],
  confidenceThreshold: 0.75,
  maxResults: 7,
}

// Recommended tools
['search_knowledge', 'get_strategy_performance', 'store_insight']
```

### Bond Agent Preset

```typescript
import { BOND_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'bond-agent',
  agentName: 'CLIX Bond Auction Agent',
  primaryDomain: 'bond_auction',
  secondaryDomains: ['analytics'],
  confidenceThreshold: 0.8,
  maxResults: 5,
}

// Recommended tools
['search_knowledge', 'get_bond_auction_history', 'store_insight']
```

### Letter of Credit Agent Preset

```typescript
import { LC_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'lc-agent',
  agentName: 'CLIX Letter of Credit Agent',
  primaryDomain: 'letter_of_credit',
  secondaryDomains: ['commodities'],
  confidenceThreshold: 0.85,
  maxResults: 5,
}

// Recommended tools
['search_knowledge', 'get_ucp_rules', 'get_error_resolution']
```

### Commodities Agent Preset

```typescript
import { COMMODITIES_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'commodities-agent',
  agentName: 'CLIX Commodities Agent',
  primaryDomain: 'commodities',
  secondaryDomains: ['analytics', 'trading'],
  confidenceThreshold: 0.75,
  maxResults: 5,
}

// Recommended tools
['search_knowledge', 'get_commodity_price', 'store_insight']
```

### Analytics Agent Preset

```typescript
import { ANALYTICS_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'analytics-agent',
  agentName: 'CLIX Analytics Agent',
  primaryDomain: 'analytics',
  secondaryDomains: ['trading', 'wallet', 'bot'],
  confidenceThreshold: 0.7,
  maxResults: 10,
}

// Recommended tools
['search_knowledge', 'recall_user_context', 'store_insight']
```

### Master Orchestrator Preset

```typescript
import { MASTER_AGENT_PRESET } from '@/agent-sdk/agent-presets';

// Configuration
{
  agentId: 'master-agent',
  agentName: 'CLIX Master Agent',
  primaryDomain: 'trading',
  secondaryDomains: ['wallet', 'bot', 'bond_auction', 'letter_of_credit', 'commodities', 'analytics'],
  confidenceThreshold: 0.6,
  maxResults: 5,
}

// Recommended tools
['search_knowledge', 'recall_user_context']
```

---

## Factory Functions

### Create Agent with Tools

```typescript
import { createAgentWithTools } from '@/agent-sdk/agent-presets';

// Create complete agent setup
const { client, executor, preset } = createAgentWithTools('trading', {
  // Optional config overrides
  confidenceThreshold: 0.8,
});

// Use the executor
const signals = await executor.getTradingSignals({
  trading_pair: 'BTC/USDT',
  signal_type: 'buy',
  timeframe: '1d',
});

// Access preset configuration
console.log('Agent:', preset.config.agentName);
console.log('System prompt addition:', preset.systemPromptAddition);
```

### Create RAG Client Only

```typescript
import { createAgentRAGClient } from '@/agent-sdk/agent-presets';

const client = createAgentRAGClient('wallet', {
  maxResults: 10,
});
```

### Get Tool Definitions for Agent

```typescript
import { getAgentToolDefinitions } from '@/agent-sdk/agent-presets';

const tools = getAgentToolDefinitions('trading');
// Returns array of OpenAI function calling definitions
```

### Get System Prompt Addition

```typescript
import { getAgentSystemPromptAddition } from '@/agent-sdk/agent-presets';

const systemPrompt = getAgentSystemPromptAddition('trading');
// Returns system prompt enhancement string
```

### Get Few-Shot Examples

```typescript
import { getAgentFewShotExamples } from '@/agent-sdk/agent-presets';

const examples = getAgentFewShotExamples('trading');
// Returns array of {userQuery, ragQuery, usage}
```

---

## LangChain Integration

Create LangChain-compatible tools:

```typescript
import { createLangChainTools } from '@/agent-sdk/agent-tools';
import { RAGClient } from '@/agent-sdk/rag-client';

const client = new RAGClient({ agentId: 'my-agent' });
const tools = createLangChainTools(client);

// Available tools:
// - search_clicbrain: Search knowledge base
// - recall_context: Recall user context (format: "user_id:context_type")
// - store_learning: Store insight (format: "title|content|tags")
```

---

## Helper Functions

### Format RAG Result for Agent

```typescript
import { formatRAGResultForAgent } from '@/agent-sdk/agent-tools';

const result = await executor.searchKnowledge({ query: 'BTC patterns' });
const formatted = formatRAGResultForAgent(result);

// Returns formatted markdown:
// ## Relevant Knowledge
// [context]
//
// ## Insights
// - insight 1
// - insight 2
//
// ## Suggested Actions
// - action 1
```

### Extract Key Facts

```typescript
import { extractKeyFacts } from '@/agent-sdk/agent-tools';

const result = await executor.searchKnowledge({ query: 'trading requirements' });
const facts = extractKeyFacts(result);
// Returns array of first sentences from each knowledge item
```

---

## Usage Example

Complete example integrating tools with an agent:

```typescript
import { createAgentWithTools, getAgentToolDefinitions } from '@/agent-sdk/agent-presets';
import { AgentRegistry } from '@/cap/services/agent-registry';

// 1. Create agent configuration
const { client, executor, preset } = createAgentWithTools('trading');

// 2. Get tool definitions for the agent
const toolDefs = getAgentToolDefinitions('trading');

// 3. Register agent with CAP
const registry = new AgentRegistry();
const agent = await registry.register({
  agent_id: preset.config.agentId,
  name: preset.config.agentName,
  version: '1.0.0',
  system: 'clix',
  type: 'llm',
  capabilities: {
    domains: [preset.config.primaryDomain, ...(preset.config.secondaryDomains || [])],
    actions: ['llm_generate', 'query_clicbrain'],
    tools: preset.recommendedTools,
  },
});

// 4. Use tools during execution
async function handleUserQuery(query: string) {
  // Search knowledge
  const knowledge = await executor.searchKnowledge({
    query,
    domain: preset.config.primaryDomain,
    max_results: preset.config.maxResults,
  });

  // Store any insights
  if (knowledge.insights.length > 0) {
    await executor.storeInsight({
      title: `Insight from query: ${query}`,
      content: knowledge.insights.join('\n'),
      tags: ['trading', 'automated'],
    });
  }

  return knowledge;
}
```
