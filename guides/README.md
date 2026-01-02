# Guides

This section provides how-to guides for configuring and deploying ClicBrain agents.

## Table of Contents

1. [Configuration](#configuration)
2. [Environment Setup](#environment-setup)
3. [Database Configuration](#database-configuration)
4. [Distributed Mode Setup](#distributed-mode-setup)
5. [Observability Configuration](#observability-configuration)
6. [Deployment Strategies](#deployment-strategies)
7. [Security Best Practices](#security-best-practices)

---

## Configuration

### Environment Variables

Essential environment variables for the Agent SDK:

```bash
# Server Configuration
PORT=3008
NODE_ENV=production

# Database (PostgreSQL)
PGHOST=localhost
PGPORT=5435
PGDATABASE=clicbrain
PGUSER=clicbrain
PGPASSWORD=your-secure-password

# Redis (for distributed mode)
REDIS_URL=redis://localhost:6379

# Neo4j (for GraphRAG)
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your-neo4j-password

# DSPy Service
DSPY_SERVICE_URL=http://localhost:8000
DSPY_API_KEY=your-dspy-key

# LLM Providers
ANTHROPIC_API_KEY=your-anthropic-key
OPENAI_API_KEY=your-openai-key

# Observability
OTEL_SERVICE_NAME=clicbrain-cap
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# CAP Configuration
CAP_MAX_CONCURRENT_RUNS=100
CAP_DEFAULT_TIMEOUT_MS=300000
CAP_CHECKPOINT_INTERVAL_MS=30000
```

### CAP Hub Configuration

```typescript
import { CAPHub } from '@/cap/services/cap-hub';

const hub = new CAPHub({
  // Persistence settings
  persistence: {
    enabled: true,
    writeThrough: true,
    syncOnStartup: true,
  },

  // Distributed mode
  distributed: {
    enabled: process.env.NODE_ENV === 'production',
    redis: process.env.REDIS_URL,
    queue: 'cap:runs',
    concurrency: parseInt(process.env.CAP_MAX_CONCURRENT_RUNS || '100'),
  },

  // Observability
  observability: {
    tracing: true,
    metrics: true,
    logging: {
      level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
    },
  },

  // Health checks
  healthCheck: {
    enabled: true,
    interval: 30000,
    timeout: 5000,
  },
});
```

### Run Executor Configuration

```typescript
import { RunExecutor } from '@/cap/services/run-executor';

const executor = new RunExecutor({
  // Execution mode
  mode: 'local', // or 'distributed'

  // Redis for distributed mode
  redis: process.env.REDIS_URL,
  queue: 'cap:runs',
  concurrency: 10,

  // Timeouts
  defaultTimeout: 300000, // 5 minutes
  maxTimeout: 1800000, // 30 minutes

  // Checkpointing
  checkpointInterval: 30000,
  checkpointStorage: 'redis', // or 'postgres'

  // Retries
  maxRetries: 3,
  retryDelay: 1000,
  retryBackoff: 'exponential',
});
```

---

## Environment Setup

### Development Environment

```bash
# 1. Install dependencies
npm install

# 2. Start PostgreSQL
docker run -d \
  --name clicbrain-postgres \
  -e POSTGRES_DB=clicbrain \
  -e POSTGRES_USER=clicbrain \
  -e POSTGRES_PASSWORD=dev-password \
  -p 5435:5432 \
  postgres:15

# 3. Start Redis (optional, for distributed mode)
docker run -d \
  --name clicbrain-redis \
  -p 6379:6379 \
  redis:7

# 4. Start Neo4j (optional, for GraphRAG)
docker run -d \
  --name neo4j \
  -e NEO4J_AUTH=neo4j/dev-password \
  -p 7474:7474 \
  -p 7687:7687 \
  neo4j:5

# 5. Run migrations
npm run migrate

# 6. Start development server
npm run dev:server
```

### Production Environment

```bash
# Use docker-compose for production
version: '3.8'

services:
  clicbrain:
    image: clicbrain:latest
    ports:
      - "3008:3008"
    environment:
      - NODE_ENV=production
      - PGHOST=postgres
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=clicbrain
      - POSTGRES_USER=clicbrain
      - POSTGRES_PASSWORD_FILE=/run/secrets/pg_password
    secrets:
      - pg_password

  redis:
    image: redis:7
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:

secrets:
  pg_password:
    external: true
```

---

## Database Configuration

### PostgreSQL Setup

```sql
-- Create database and user
CREATE DATABASE clicbrain;
CREATE USER clicbrain WITH ENCRYPTED PASSWORD 'your-password';
GRANT ALL PRIVILEGES ON DATABASE clicbrain TO clicbrain;

-- Enable required extensions
\c clicbrain
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "vector"; -- For embeddings
```

### Migrations

Migrations are automatically run on server startup. To add a new migration:

1. Create SQL file in `src/storage/postgres/migrations/`:
```sql
-- v9.0.0_my_feature.sql
CREATE TABLE IF NOT EXISTS my_table (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

2. Register in `src/storage/postgres/migrations.ts`:
```typescript
{
  version: '9.0.0',
  name: 'my_feature',
  description: 'Add my feature table',
  up: fs.readFileSync(path.join(__dirname, 'migrations/v9.0.0_my_feature.sql'), 'utf-8'),
  down: 'DROP TABLE IF EXISTS my_table;'
}
```

### Connection Pooling

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.PGHOST,
  port: parseInt(process.env.PGPORT || '5432'),
  database: process.env.PGDATABASE,
  user: process.env.PGUSER,
  password: process.env.PGPASSWORD,

  // Pool configuration
  max: 20, // Maximum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,

  // SSL for production
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: true } : false,
});
```

---

## Distributed Mode Setup

### Redis Configuration

```typescript
import { Redis } from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,

  // Retry strategy
  retryStrategy: (times) => Math.min(times * 50, 2000),

  // Connection options
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  lazyConnect: true,
});
```

### BullMQ Queue Setup

```typescript
import { Queue, Worker } from 'bullmq';

// Create queue
const runQueue = new Queue('cap:runs', {
  connection: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
    removeOnComplete: true,
    removeOnFail: false,
  },
});

// Create worker
const worker = new Worker('cap:runs', async (job) => {
  const executor = new RunExecutor({ mode: 'local' });
  return executor.executeJob(job.data);
}, {
  connection: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
  concurrency: parseInt(process.env.CAP_CONCURRENCY || '10'),
});
```

### Scaling Workers

```bash
# Run multiple worker instances
docker-compose up --scale worker=5

# Or using PM2
pm2 start worker.js -i 5 --name cap-worker
```

---

## Observability Configuration

### OpenTelemetry Setup

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'clicbrain-cap',
    [SemanticResourceAttributes.SERVICE_VERSION]: '8.0.0',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + '/v1/traces',
  }),
});

sdk.start();
```

### Logging Configuration

```typescript
import { createLogger } from '@/core/logger';

const logger = createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: 'json',
  context: {
    service: 'clicbrain-cap',
    environment: process.env.NODE_ENV,
  },
});

// Log levels: error, warn, info, debug, trace
logger.info('CAP Hub started', { port: 3008 });
```

### Metrics Collection

```typescript
import { MeterProvider } from '@opentelemetry/sdk-metrics';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';

const meterProvider = new MeterProvider({
  readers: [
    {
      exporter: new OTLPMetricExporter({
        url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + '/v1/metrics',
      }),
      exportIntervalMillis: 10000,
    },
  ],
});

const meter = meterProvider.getMeter('clicbrain-cap');

// Create metrics
const runCounter = meter.createCounter('cap.runs.total');
const runDuration = meter.createHistogram('cap.runs.duration_ms');
const activeRuns = meter.createUpDownCounter('cap.runs.active');
```

---

## Deployment Strategies

### Blue-Green Deployment

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clicbrain-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: clicbrain
      version: blue
  template:
    spec:
      containers:
        - name: clicbrain
          image: clicbrain:v8.0.0
          ports:
            - containerPort: 3008
          readinessProbe:
            httpGet:
              path: /health
              port: 3008
            initialDelaySeconds: 10
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: clicbrain
spec:
  selector:
    app: clicbrain
    version: blue  # Switch to green for deployment
  ports:
    - port: 3008
```

### Agent Version Management

```typescript
// Deploy new agent version
const v2Agent = await registry.register({
  agent_id: 'my-agent.v2',
  name: 'My Agent',
  version: '2.0.0',
  // ... config
});

// Canary deployment - route 10% traffic
await registry.setRoutingWeight('my-agent.v1', 90);
await registry.setRoutingWeight('my-agent.v2', 10);

// Gradual rollout
await registry.setRoutingWeight('my-agent.v1', 50);
await registry.setRoutingWeight('my-agent.v2', 50);

// Complete migration
await registry.setRoutingWeight('my-agent.v1', 0);
await registry.setRoutingWeight('my-agent.v2', 100);

// Deprecate old version
await registry.deprecate('my-agent.v1');
```

---

## Security Best Practices

### API Authentication

```typescript
import { verifyToken } from '@/auth/jwt';

// Middleware for API authentication
async function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  try {
    const payload = await verifyToken(token);
    req.user = payload;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Agent Authorization

```typescript
// Check agent permissions before execution
async function authorizeAgentExecution(
  userId: string,
  agentId: string
): Promise<boolean> {
  const agent = await registry.get(agentId);

  // Check agent is active
  if (agent.status !== 'active') {
    return false;
  }

  // Check user has permission for agent's domain
  const userPermissions = await getUserPermissions(userId);
  const requiredDomains = agent.capabilities?.domains || [];

  return requiredDomains.every((domain) =>
    userPermissions.includes(`agent:${domain}:execute`)
  );
}
```

### Secrets Management

```typescript
// Use environment variables or secret manager
const secrets = {
  anthropicKey: process.env.ANTHROPIC_API_KEY,
  dbPassword: process.env.PGPASSWORD,
  jwtSecret: process.env.JWT_SECRET,
};

// Validate required secrets on startup
function validateSecrets() {
  const required = ['ANTHROPIC_API_KEY', 'PGPASSWORD', 'JWT_SECRET'];
  const missing = required.filter((key) => !process.env[key]);

  if (missing.length > 0) {
    throw new Error(`Missing required secrets: ${missing.join(', ')}`);
  }
}
```

### Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';

// API rate limiting
const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many requests',
      retryAfter: 60,
    });
  },
});

// Agent execution rate limiting
const executionLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 20, // 20 executions per minute
  keyGenerator: (req) => req.user?.id || req.ip,
});

app.use('/api/cap', apiLimiter);
app.use('/api/cap/runs', executionLimiter);
```

### Input Validation

```typescript
import { z } from 'zod';

// Validate run input
const RunInputSchema = z.object({
  agent_id: z.string().min(1).max(100),
  thread_id: z.string().uuid().optional(),
  input: z.object({
    messages: z.array(z.object({
      role: z.enum(['user', 'assistant', 'system']),
      content: z.string().max(100000),
    })),
    context: z.record(z.unknown()).optional(),
  }),
  options: z.object({
    timeout_ms: z.number().min(1000).max(1800000).optional(),
    stream: z.boolean().optional(),
  }).optional(),
});

// Validate in route handler
app.post('/api/cap/runs', async (req, res) => {
  const result = RunInputSchema.safeParse(req.body);

  if (!result.success) {
    return res.status(400).json({
      error: 'Validation failed',
      details: result.error.issues,
    });
  }

  const run = await executor.execute(result.data);
  res.json({ success: true, data: run });
});
```

---

## Health Checks

### Liveness Probe

```typescript
app.get('/health/live', (req, res) => {
  res.json({ status: 'alive' });
});
```

### Readiness Probe

```typescript
app.get('/health/ready', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    dspy: await checkDSPy(),
  };

  const allHealthy = Object.values(checks).every((c) => c.healthy);

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ready' : 'not_ready',
    checks,
  });
});

async function checkDatabase(): Promise<{ healthy: boolean; latency: number }> {
  const start = Date.now();
  try {
    await pool.query('SELECT 1');
    return { healthy: true, latency: Date.now() - start };
  } catch {
    return { healthy: false, latency: -1 };
  }
}
```

### Startup Probe

```typescript
app.get('/health/startup', async (req, res) => {
  const initialized = await hub.isInitialized();
  res.status(initialized ? 200 : 503).json({
    status: initialized ? 'started' : 'starting',
  });
});
```
