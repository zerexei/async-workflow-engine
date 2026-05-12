# Async Workflow Engine

A distributed asynchronous job processing system built with FastAPI and Redis Streams, designed to explore reliability engineering patterns in real-world backend systems: retries, idempotency, worker leasing, and failure recovery under distributed execution.

This system focuses on production failure modes—duplicate delivery, worker crashes, partial execution, and retry storms—and implements coordination primitives to handle them safely.

## Architecture Overview

The system is composed of multiple FastAPI services coordinated through Redis Streams, with PostgreSQL used for service-level persistence.
```
Clients / Gateway
        │
        ▼
FastAPI Services (Workers + API Layer)
        │
        ▼
Redis Streams (Job Transport Layer)
(XADD / XREADGROUP consumer groups)
        │
        ▼
Worker Execution Layer
        │
        ├── Idempotency Store (Redis)
        ├── State Store (PostgreSQL)
        └── DLQ Streams (failure routing)
```
## Core Components
- Queueing System: Redis Streams (consumer groups for distributed workers)
- API Layer: FastAPI gateway and service endpoints
- Workers: Async job processors consuming stream entries
- Persistence Layer: PostgreSQL per service for durable state tracking
- Coordination Layer: Redis-based idempotency + leasing + retry control

## Async Execution Model

Jobs are executed through a distributed consumer group model using Redis Streams.

### Flow
1. API produces a job using XADD
2. Job is consumed via XREADGROUP
3. Worker validates execution constraints:
    - idempotency check
    - retry state
    - job type routing
4. Job executes asynchronously
5. Result is persisted and acknowledged
6. Failure paths trigger retry or DLQ routing

### Key Characteristics
- At-least-once delivery model with deduplication layer
- Concurrent worker execution across services
- Explicit separation between job ingestion and execution
- Stateless workers with externalized coordination state

## Reliability Design

This system is built around explicit failure assumptions in distributed systems.

## 1. Retry Strategy

Retry behavior is designed to prevent cascading failure amplification.

- Exponential backoff: 1s → 5s → 25s → 125s
- Maximum retries: 5 attempts per job
- Retry classification:
    - retryable: transient errors (timeouts, rate limits)
    - non-retryable: validation or logic errors

Retry state is persisted per job using attempt_count.

## 2. Idempotency Model

To prevent duplicate side effects in at-least-once delivery systems:

- Every job requires an idempotency_key
- Before execution, worker checks Redis (SET NX)
- If key exists → job is skipped safely
- Keys are time-bounded (TTL-based expiration, e.g. 24h)

This ensures safe execution under:

- retry storms
- duplicate stream delivery
- worker reprocessing after failure
- 
## 3. Dead Letter Queue (DLQ)

Jobs that cannot be safely processed are routed to a DLQ stream.

A job is sent to DLQ when:

- max retry attempts exceeded
- non-retryable error occurs
- missing or invalid handler

## DLQ Operations
- Inspect:
    - GET /dlq/{service_name}
- Replay:
    - POST /dlq/{service_name}/{job_id}/replay

DLQ enables controlled recovery without data loss or silent failure.

## 4. Worker Leasing & Execution Safety

To prevent duplicate execution across distributed workers:

- Jobs are leased to a worker during processing
- Lease expiration handles worker crashes
- Unacknowledged jobs become eligible for reprocessing

This prevents:

- split-brain execution
- duplicate side effects from worker failures
- stuck jobs due to crashed consumers

## 5. Partial Failure Recovery

Multi-step workflows persist intermediate state.

- Each step writes progress via StateManager
- On retry, execution resumes from last successful checkpoint
- Prevents re-running completed steps

This enables:

- resumable workflows
- crash-safe multi-step pipelines
- deterministic recovery behavior


## 6. Failure Simulation Layer

The system includes controlled failure injection for testing resilience:

- Latency injection (artificial delays)
- Transient errors (retry-triggering failures)
- Permanent errors (DLQ routing)
- Retry storm simulation
- Partial pipeline failure injection

This is used to validate:

- retry correctness
- DLQ routing behavior
- system stability under stress

## 7. Rate Limiting Simulation

External API constraints are simulated using a Redis Token Bucket.

- Excess requests trigger retryable failures
- Backoff behavior is exercised under pressure
- Models real-world API throttling patterns

## Key Engineering Concepts

This system is designed to demonstrate production-grade distributed systems thinking:

- At-least-once delivery with idempotency safety
- Worker leasing and crash recovery
- Visibility timeout-based reprocessing
- Retry policy design with backoff control
- DLQ-based failure isolation
- Partial execution recovery (checkpointing)
- Stream-based distributed coordination (Redis Streams)

## Tech Stack
### Backend
- FastAPI
- Python 3.12+
### Queueing & Coordination
- Redis Streams (XADD, XREADGROUP)
- Redis (idempotency + leasing + rate limiting)
### Persistence
- PostgreSQL (service-level state tracking)
### Infrastructure
- Docker Compose

## Running Locally
```
cd infrastructure
docker-compose up --build
```

## API Gateway

The gateway routes requests into the async job system:

- GET /health — service health check
- POST /users — async user creation job
- GET /jobs/{job_id} — job status tracking
- GET /dlq/{service_name} — inspect failed jobs
- POST /dlq/{service_name}/{job_id}/replay — replay DLQ job

## System Design Intent

This system is intentionally built around real distributed systems failure modes:

- Workers crash mid-execution
- Messages are delivered multiple times
- Retries overlap under load
- Execution order is not guaranteed
- Partial system failure is normal, not exceptional

Instead of hiding these conditions, the system makes them explicit and recoverable through:

idempotency boundaries
- leasing-based execution control
- retry discipline
- durable state checkpoints
- DLQ-based recovery paths
  
## Design Tradeoffs
- Redis Streams chosen over full workflow engines to reduce operational complexity while retaining coordination guarantees
- Idempotency handled at application layer instead of strict queue semantics
- PostgreSQL used per-service to isolate state and reduce cross-service coupling
- Leasing + visibility timeout model used instead of central orchestrator

## What This Demonstrates

This project highlights backend engineering maturity in:

- distributed job processing systems
- failure-aware system design
- concurrency-safe execution models
- production retry and idempotency strategies
- operational debugging of async systems
- stream-based coordination patterns
