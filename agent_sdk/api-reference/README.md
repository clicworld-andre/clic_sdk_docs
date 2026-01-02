# API Reference

Complete API documentation for the ClicBrain Agent SDK.

## REST API Endpoints

### Agents API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/cap/agents` | GET | List all agents |
| `/api/cap/agents` | POST | Create new agent |
| `/api/cap/agents/:id` | GET | Get agent by ID |
| `/api/cap/agents/:id` | PUT | Update agent |
| `/api/cap/agents/:id` | DELETE | Delete agent |
| `/api/cap/agents/:id/health` | GET | Get agent health |
| `/api/cap/agents/discover` | POST | Discover agents by criteria |

### Threads API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/cap/threads` | POST | Create thread |
| `/api/cap/threads/:id` | GET | Get thread |
| `/api/cap/threads/:id` | PUT | Update thread |
| `/api/cap/threads/:id/messages` | GET | Get messages |
| `/api/cap/threads/:id/messages` | POST | Add message |
| `/api/cap/threads/:id/close` | POST | Close thread |

### Runs API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/cap/runs` | POST | Execute run |
| `/api/cap/runs/:id` | GET | Get run status |
| `/api/cap/runs/:id/cancel` | POST | Cancel run |
| `/api/cap/runs/:id/stream` | GET | Stream run output (SSE) |

### Interrupts API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/cap/interrupts` | GET | List pending interrupts |
| `/api/cap/interrupts/:id` | GET | Get interrupt |
| `/api/cap/interrupts/:id/resolve` | POST | Resolve interrupt |

---

## Agents API

### List Agents

```http
GET /api/cap/agents
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `system` | string | Filter by Clic system |
| `type` | string | Filter by agent type |
| `status` | string | Filter by status |
| `limit` | number | Max results (default: 50) |
| `offset` | number | Pagination offset |

**Response:**

```json
{
  "success": true,
  "data": {
    "agents": [
      {
        "id": "uuid",
        "agent_id": "clicbrain.legal.v1",
        "name": "Legal Agent",
        "version": "1.0.0",
        "system": "clicbrain",
        "type": "legal",
        "status": "active",
        "health_status": {
          "status": "healthy"
        }
      }
    ],
    "total": 15,
    "limit": 50,
    "offset": 0
  }
}
```

### Create Agent

```http
POST /api/cap/agents
Content-Type: application/json
```

**Request Body:**

```json
{
  "agent_id": "my-company.assistant.v1",
  "name": "My Assistant",
  "version": "1.0.0",
  "description": "A helpful assistant agent",
  "system": "clicbrain",
  "type": "llm",
  "capabilities": {
    "domains": ["global"],
    "actions": ["llm_generate", "query_clicbrain"],
    "tools": [],
    "parallel_tool_calls": true
  },
  "cap_extensions": {
    "supports_threads": true,
    "supports_interrupts": true,
    "supports_streaming": true,
    "max_concurrent_runs": 10,
    "default_timeout_ms": 300000
  }
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "agent_id": "my-company.assistant.v1",
    "name": "My Assistant",
    "version": "1.0.0",
    "status": "active",
    "lifecycle_state": "registered",
    "created_at": "2026-01-02T00:00:00Z"
  }
}
```

### Get Agent

```http
GET /api/cap/agents/:id
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "agent_id": "my-company.assistant.v1",
    "name": "My Assistant",
    "version": "1.0.0",
    "description": "A helpful assistant agent",
    "system": "clicbrain",
    "type": "llm",
    "status": "active",
    "lifecycle_state": "ready",
    "capabilities": {
      "domains": ["global"],
      "actions": ["llm_generate", "query_clicbrain"],
      "tools": [],
      "parallel_tool_calls": true
    },
    "cap_extensions": {
      "supports_threads": true,
      "supports_interrupts": true,
      "supports_streaming": true,
      "max_concurrent_runs": 10,
      "default_timeout_ms": 300000
    },
    "health_status": {
      "status": "healthy",
      "metrics": {
        "avg_latency_ms": 150,
        "success_rate": 0.98,
        "active_runs": 2
      }
    },
    "created_at": "2026-01-02T00:00:00Z",
    "updated_at": "2026-01-02T00:00:00Z"
  }
}
```

### Update Agent

```http
PUT /api/cap/agents/:id
Content-Type: application/json
```

**Request Body:**

```json
{
  "name": "My Updated Assistant",
  "description": "Updated description",
  "status": "active",
  "capabilities": {
    "domains": ["global", "trading"],
    "actions": ["llm_generate", "query_clicbrain", "llm_analyze"]
  }
}
```

### Delete Agent

```http
DELETE /api/cap/agents/:id
```

**Response:**

```json
{
  "success": true,
  "data": {
    "deleted": true,
    "agent_id": "my-company.assistant.v1"
  }
}
```

### Get Agent Health

```http
GET /api/cap/agents/:id/health
```

**Response:**

```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "components": {
      "database": { "status": "healthy" },
      "redis": { "status": "healthy" },
      "llm": { "status": "healthy" }
    },
    "metrics": {
      "avg_latency_ms": 150,
      "success_rate": 0.98,
      "active_runs": 2,
      "queued_runs": 0
    },
    "issues": [],
    "updated_at": "2026-01-02T00:00:00Z"
  }
}
```

### Discover Agents

```http
POST /api/cap/agents/discover
Content-Type: application/json
```

**Request Body:**

```json
{
  "system": "clicbrain",
  "type": "llm",
  "status": "active",
  "domain": "trading",
  "has_tool": "search_knowledge",
  "supports_threads": true
}
```

---

## Threads API

### Create Thread

```http
POST /api/cap/threads
Content-Type: application/json
```

**Request Body:**

```json
{
  "agent_id": "my-company.assistant.v1",
  "metadata": {
    "user_id": "user-123",
    "session_id": "session-456"
  },
  "initial_messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "thread-uuid",
    "agent_id": "my-company.assistant.v1",
    "status": "active",
    "metadata": {
      "user_id": "user-123",
      "session_id": "session-456"
    },
    "created_at": "2026-01-02T00:00:00Z"
  }
}
```

### Get Thread

```http
GET /api/cap/threads/:id
```

### Get Messages

```http
GET /api/cap/threads/:id/messages
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | number | Max messages |
| `offset` | number | Pagination offset |
| `role` | string | Filter by role |

### Add Message

```http
POST /api/cap/threads/:id/messages
Content-Type: application/json
```

**Request Body:**

```json
{
  "role": "user",
  "content": "Hello, agent!"
}
```

### Close Thread

```http
POST /api/cap/threads/:id/close
Content-Type: application/json
```

**Request Body:**

```json
{
  "summary": "User inquiry completed",
  "resolution": "completed"
}
```

---

## Runs API

### Execute Run

```http
POST /api/cap/runs
Content-Type: application/json
```

**Request Body:**

```json
{
  "agent_id": "my-company.assistant.v1",
  "thread_id": "thread-uuid",
  "input": {
    "messages": [
      { "role": "user", "content": "Analyze my portfolio" }
    ],
    "context": {
      "portfolio_id": "portfolio-123"
    }
  },
  "options": {
    "timeout_ms": 60000,
    "stream": false
  }
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "run-uuid",
    "agent_id": "my-company.assistant.v1",
    "thread_id": "thread-uuid",
    "status": "completed",
    "output": {
      "response": "Your portfolio analysis...",
      "usage": {
        "input_tokens": 150,
        "output_tokens": 500,
        "total_tokens": 650
      }
    },
    "created_at": "2026-01-02T00:00:00Z",
    "completed_at": "2026-01-02T00:00:05Z"
  }
}
```

### Stream Run (SSE)

```http
GET /api/cap/runs/:id/stream
Accept: text/event-stream
```

**Events:**

```
event: step:started
data: {"step":{"id":"step-1","type":"llm_call","name":"generate"}}

event: token
data: {"token":"Your"}

event: token
data: {"token":" portfolio"}

event: step:completed
data: {"step":{"id":"step-1","status":"completed"}}

event: completed
data: {"run":{"id":"run-uuid","status":"completed"}}
```

### Cancel Run

```http
POST /api/cap/runs/:id/cancel
Content-Type: application/json
```

**Request Body:**

```json
{
  "reason": "User requested cancellation"
}
```

---

## Interrupts API

### List Pending Interrupts

```http
GET /api/cap/interrupts
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `run_id` | string | Filter by run |
| `type` | string | Filter by interrupt type |
| `priority` | string | Filter by priority |
| `status` | string | Filter by status |

### Get Interrupt

```http
GET /api/cap/interrupts/:id
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "interrupt-uuid",
    "run_id": "run-uuid",
    "type": "approval_required",
    "priority": "high",
    "status": "pending",
    "message": "Approve this action?",
    "options": ["Approve", "Reject"],
    "timeout_ms": 300000,
    "created_at": "2026-01-02T00:00:00Z",
    "expires_at": "2026-01-02T00:05:00Z"
  }
}
```

### Resolve Interrupt

```http
POST /api/cap/interrupts/:id/resolve
Content-Type: application/json
```

**Request Body:**

```json
{
  "response": "Approve",
  "resolved_by": "user-123",
  "metadata": {
    "comment": "Looks good"
  }
}
```

---

## Error Responses

All endpoints return errors in a consistent format:

```json
{
  "success": false,
  "error": {
    "code": "AGENT_NOT_FOUND",
    "message": "Agent not found: my-company.assistant.v1",
    "details": {}
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `AGENT_NOT_FOUND` | 404 | Agent not found |
| `THREAD_NOT_FOUND` | 404 | Thread not found |
| `RUN_NOT_FOUND` | 404 | Run not found |
| `VALIDATION_ERROR` | 400 | Invalid request |
| `AGENT_UNHEALTHY` | 503 | Agent not healthy |
| `RUN_TIMEOUT` | 408 | Run timed out |
| `INTERRUPT_EXPIRED` | 410 | Interrupt expired |
| `INTERNAL_ERROR` | 500 | Internal error |
