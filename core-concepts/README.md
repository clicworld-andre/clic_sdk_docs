# Core Concepts

This section covers the fundamental concepts of the ClicBrain Agent SDK.

## Contents

1. [CAP Architecture](./cap-architecture.md) - The Clic Agent Protocol
2. [Agents](./agents.md) - Agent types, lifecycle, and capabilities
3. [Threads](./threads.md) - Stateful conversation management
4. [Runs](./runs.md) - Execution lifecycle and step handling
5. [Interrupts](./interrupts.md) - Human-in-the-loop interactions
6. [Skills](./skills.md) - Reusable agent capabilities

## Overview

The ClicBrain Agent SDK is built on three core pillars:

### 1. CAP (Clic Agent Protocol)

CAP is the foundational protocol that enables:
- Agent registration and discovery
- Thread-based stateful execution
- Interrupt handling for human oversight
- Distributed execution with BullMQ

### 2. Knowledge Integration

Agents seamlessly integrate with ClicBrain's knowledge base:
- RAG (Retrieval-Augmented Generation) for context
- QKO (Quantum Knowledge Objects) for structured data
- Domain-partitioned access control

### 3. Observability

Built-in observability for production systems:
- OpenTelemetry distributed tracing
- Structured JSON logging
- Prometheus-compatible metrics
