# `@blackroad/job-scheduler`

> **BlackRoad OS — Enterprise Job Scheduler & Task Queue**

[![npm version](https://img.shields.io/npm/v/@blackroad/job-scheduler?color=black&label=npm)](https://www.npmjs.com/package/@blackroad/job-scheduler)
[![License: Proprietary](https://img.shields.io/badge/License-Proprietary-black)](./LICENSE)
[![BlackRoad OS](https://img.shields.io/badge/BlackRoad-OS-black?logo=data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMTYiIGhlaWdodD0iMTYiIHZpZXdCb3g9IjAgMCAxNiAxNiIgZmlsbD0ibm9uZSIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cmVjdCB3aWR0aD0iMTYiIGhlaWdodD0iMTYiIGZpbGw9ImJsYWNrIi8+PC9zdmc+)](https://blackroad.io)
[![Status: Production](https://img.shields.io/badge/Status-Production-brightgreen)](https://status.blackroad.io)

Production-grade, distributed job scheduling and task queue for the **BlackRoad OS** ecosystem. Schedule cron jobs, manage priority task queues, handle retries with exponential back-off, and process Stripe billing events — all through a single, unified API.

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [Configuration](#configuration)
6. [API Reference](#api-reference)
   - [Job Scheduling](#job-scheduling)
   - [Task Queue](#task-queue)
   - [Stripe Billing Events](#stripe-billing-events)
   - [Execution Logs](#execution-logs)
7. [Cron Syntax Reference](#cron-syntax-reference)
8. [Retry & Back-off Policy](#retry--back-off-policy)
9. [End-to-End Testing](#end-to-end-testing)
10. [Architecture](#architecture)
11. [BlackRoad OS Ecosystem](#blackroad-os-ecosystem)
12. [Support & SLA](#support--sla)
13. [License](#license)

---

## Overview

`@blackroad/job-scheduler` is the **canonical** job scheduling and task queue microservice for BlackRoad OS. It powers every timed operation across the platform — from recurring billing cycles and subscription renewals via Stripe, to data pipeline orchestration, health checks, and automated report delivery.

| | |
|---|---|
| **Production API** | `https://api.blackroad.io/job-scheduler` |
| **Dashboard** | `https://console.blackroad.io/scheduler` |
| **npm Package** | `@blackroad/job-scheduler` |
| **Documentation** | `https://docs.blackroad.io/job-scheduler` |
| **Status Page** | `https://status.blackroad.io` |

---

## Features

- **Cron Scheduling** — Full cron expression support with named presets
- **Priority Task Queue** — Multi-tier priority queues (critical / high / normal / low)
- **Distributed Execution** — Horizontally scalable workers with leader election
- **Stripe Billing Integration** — First-class handlers for subscription lifecycle, invoice events, and payment retries
- **Retry Logic** — Configurable exponential back-off with dead-letter queues (DLQ)
- **Timezone-Aware** — Schedule jobs in any IANA timezone
- **HTTP & Function Triggers** — Invoke REST endpoints or registered TypeScript handlers
- **Execution Logs** — Full audit trail with response payloads, latency, and error context
- **Real-time Observability** — Prometheus metrics, structured JSON logging, OpenTelemetry tracing
- **Zero-Downtime Deploys** — Graceful shutdown with in-flight job protection

---

## Installation

### npm

```bash
npm install @blackroad/job-scheduler
```

### yarn

```bash
yarn add @blackroad/job-scheduler
```

### pnpm

```bash
pnpm add @blackroad/job-scheduler
```

> **Node.js ≥ 18** is required. The package ships full TypeScript types and ESM + CJS dual builds.

---

## Quick Start

### 1. Initialize the scheduler

```typescript
import { JobScheduler } from '@blackroad/job-scheduler';

const scheduler = new JobScheduler({
  apiKey: process.env.BLACKROAD_API_KEY,
  projectId: process.env.BLACKROAD_PROJECT_ID,
});

await scheduler.connect();
```

### 2. Register a cron job

```typescript
await scheduler.jobs.create({
  id: 'daily-report',
  name: 'Daily Analytics Report',
  schedule: '0 9 * * 1-5',        // Mon–Fri at 9 AM
  timezone: 'America/Chicago',
  handler: async (ctx) => {
    await sendDailyReport(ctx.payload);
  },
  retries: 3,
  timeout: 60_000,
});
```

### 3. Enqueue a one-off task

```typescript
await scheduler.queue.enqueue({
  type: 'send-welcome-email',
  payload: { userId: 'usr_abc123' },
  priority: 'high',
  runAt: new Date(Date.now() + 5_000), // run in 5 seconds
});
```

### 4. Handle Stripe billing events

```typescript
scheduler.stripe.on('invoice.payment_failed', async (event) => {
  await notifyCustomer(event.data.object.customer);
  await scheduler.queue.enqueue({
    type: 'retry-payment',
    payload: { invoiceId: event.data.object.id },
    priority: 'critical',
    runAt: new Date(Date.now() + 24 * 60 * 60 * 1_000), // retry in 24 h
  });
});
```

---

## Configuration

| Option | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | **required** | BlackRoad OS API key — obtain from the [Prism Console](https://console.blackroad.io) |
| `projectId` | `string` | **required** | BlackRoad project identifier |
| `concurrency` | `number` | `5` | Number of concurrent job workers per process |
| `pollInterval` | `number` | `1000` | Queue polling interval in milliseconds |
| `timezone` | `string` | `'UTC'` | Default timezone for all jobs |
| `retries` | `number` | `3` | Global default retry count |
| `backoff` | `BackoffConfig` | `{ type: 'exponential', base: 1000 }` | Retry back-off strategy |
| `stripe.webhookSecret` | `string` | — | Stripe webhook signing secret for billing event verification |
| `stripe.apiKey` | `string` | — | Stripe secret key for server-side billing operations |
| `dlqEnabled` | `boolean` | `true` | Route exhausted jobs to the dead-letter queue |
| `telemetry.enabled` | `boolean` | `true` | Enable OpenTelemetry tracing and Prometheus metrics |

### Environment variables

```bash
BLACKROAD_API_KEY=bro_live_...
BLACKROAD_PROJECT_ID=proj_...
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

---

## API Reference

### Job Scheduling

#### `scheduler.jobs.create(definition)`

Create a recurring cron job.

```typescript
const job = await scheduler.jobs.create({
  id: string;                 // Unique job identifier
  name: string;               // Human-readable name
  schedule: string;           // Cron expression or named preset
  timezone?: string;          // IANA timezone (default: config.timezone)
  handler?: JobHandler;       // Inline async function
  endpoint?: string;          // HTTP endpoint to call instead of handler
  method?: 'GET' | 'POST';    // HTTP method (default: 'POST')
  payload?: Record<string, unknown>;
  headers?: Record<string, string>;
  enabled?: boolean;          // (default: true)
  retries?: number;
  timeout?: number;           // Milliseconds
});
```

#### `scheduler.jobs.list()`

Returns all registered jobs with their current status and statistics.

#### `scheduler.jobs.get(id)`

Returns a single job by ID.

#### `scheduler.jobs.update(id, patch)`

Partially update a job (e.g., enable/disable, change schedule).

#### `scheduler.jobs.delete(id)`

Remove a job and all associated execution history.

#### `scheduler.jobs.trigger(id, payload?)`

Manually trigger a job outside its schedule.

---

### Task Queue

#### `scheduler.queue.enqueue(task)`

Add a task to the priority queue.

```typescript
await scheduler.queue.enqueue({
  type: string;                // Task type identifier
  payload?: unknown;
  priority?: 'critical' | 'high' | 'normal' | 'low'; // (default: 'normal')
  runAt?: Date;                // Delay execution to a future time
  deduplicationId?: string;    // Prevent duplicate tasks within a window
  maxRetries?: number;
  ttl?: number;                // Milliseconds before task expires
});
```

#### `scheduler.queue.process(type, handler)`

Register a handler for a task type.

```typescript
scheduler.queue.process('send-welcome-email', async (task) => {
  await emailService.send(task.payload);
});
```

#### `scheduler.queue.stats()`

Returns queue depth, processing rate, and DLQ counts per priority tier.

---

### Stripe Billing Events

`@blackroad/job-scheduler` includes a verified Stripe webhook consumer for subscription and payment lifecycle management.

#### Supported events

| Stripe Event | Default Behavior |
|---|---|
| `customer.subscription.created` | Provision access, log to audit trail |
| `customer.subscription.updated` | Sync plan changes, update metering |
| `customer.subscription.deleted` | Revoke access, enqueue offboarding tasks |
| `invoice.payment_succeeded` | Record payment, reset retry counter |
| `invoice.payment_failed` | Notify customer, schedule payment retry |
| `invoice.finalized` | Generate and store PDF receipt |
| `payment_intent.succeeded` | Confirm one-time purchase fulfillment |
| `payment_intent.payment_failed` | Trigger remediation workflow |

#### Registering handlers

```typescript
// Override or extend default behavior
scheduler.stripe.on('customer.subscription.deleted', async (event) => {
  const subscription = event.data.object;
  await scheduler.queue.enqueue({
    type: 'deprovision-workspace',
    payload: { customerId: subscription.customer },
    priority: 'high',
  });
});
```

#### Express / Next.js webhook endpoint

```typescript
import express from 'express';
import { createStripeWebhookHandler } from '@blackroad/job-scheduler/stripe';

const app = express();

app.post(
  '/webhooks/stripe',
  express.raw({ type: 'application/json' }),
  createStripeWebhookHandler(scheduler),
);
```

---

### Execution Logs

#### `scheduler.logs.list(query?)`

```typescript
const logs = await scheduler.logs.list({
  jobId?: string;
  status?: 'success' | 'failed' | 'running';
  from?: Date;
  to?: Date;
  limit?: number;   // default: 50, max: 500
  cursor?: string;  // pagination
});
```

#### Log entry schema

```typescript
{
  id: string;
  jobId: string;
  jobName: string;
  status: 'success' | 'failed' | 'running';
  startedAt: string;       // ISO 8601
  completedAt: string;     // ISO 8601
  durationMs: number;
  attempt: number;
  response?: {
    status: number;
    body: unknown;
  };
  error?: {
    message: string;
    stack?: string;
  };
}
```

---

## Cron Syntax Reference

```
┌────────── minute       (0–59)
│ ┌──────── hour         (0–23)
│ │ ┌────── day-of-month (1–31)
│ │ │ ┌──── month        (1–12)
│ │ │ │ ┌── day-of-week  (0–6, Sun = 0)
│ │ │ │ │
* * * * *
```

### Named presets

| Preset | Equivalent | Description |
|---|---|---|
| `@yearly` | `0 0 1 1 *` | Once a year, Jan 1 at midnight |
| `@monthly` | `0 0 1 * *` | First day of each month at midnight |
| `@weekly` | `0 0 * * 0` | Every Sunday at midnight |
| `@daily` | `0 0 * * *` | Every day at midnight |
| `@hourly` | `0 * * * *` | Top of every hour |

### Common expressions

| Expression | Description |
|---|---|
| `*/5 * * * *` | Every 5 minutes |
| `0 9 * * 1-5` | Weekdays at 9 AM |
| `0 2 * * *` | Daily at 2 AM |
| `0 4 * * 0` | Sundays at 4 AM |
| `0 0 1 * *` | First of every month at midnight |
| `0 9 * * 1` | Every Monday at 9 AM |

---

## Retry & Back-off Policy

By default, failed jobs are retried using **exponential back-off with jitter**:

| Attempt | Base delay | Jitter (±20%) |
|---|---|---|
| 1 | 1 s | 0.8 – 1.2 s |
| 2 | 2 s | 1.6 – 2.4 s |
| 3 | 4 s | 3.2 – 4.8 s |
| 4 | 8 s | 6.4 – 9.6 s |
| 5 | 16 s | 12.8 – 19.2 s |

After all retries are exhausted the job is moved to the **Dead-Letter Queue (DLQ)** for manual inspection via the Prism Console or the `scheduler.dlq.*` API.

Custom back-off:

```typescript
const scheduler = new JobScheduler({
  // ...
  backoff: {
    type: 'exponential',   // 'exponential' | 'linear' | 'fixed'
    base: 2_000,           // ms
    multiplier: 2,
    maxDelay: 300_000,     // cap at 5 minutes
  },
});
```

---

## End-to-End Testing

### Prerequisites

```bash
npm install --save-dev @blackroad/job-scheduler-testing
```

### Local test environment

```bash
# Start the scheduler test harness (in-memory, no external deps)
npx blackroad-scheduler-dev
```

### Writing E2E tests

```typescript
import { createTestScheduler } from '@blackroad/job-scheduler-testing';

describe('Billing retry flow', () => {
  let scheduler: TestScheduler;

  beforeEach(async () => {
    scheduler = await createTestScheduler();
    await scheduler.start();
  });

  afterEach(() => scheduler.stop());

  it('enqueues a payment retry when invoice.payment_failed fires', async () => {
    await scheduler.stripe.simulateEvent('invoice.payment_failed', {
      customer: 'cus_test123',
      id: 'in_test456',
    });

    const tasks = await scheduler.queue.pending({ type: 'retry-payment' });
    expect(tasks).toHaveLength(1);
    expect(tasks[0].payload.invoiceId).toBe('in_test456');
  });

  it('executes a cron job on schedule', async () => {
    const spy = jest.fn();
    await scheduler.jobs.create({
      id: 'test-job',
      schedule: '* * * * *',
      handler: spy,
    });

    await scheduler.tick(); // advance clock by 1 minute
    expect(spy).toHaveBeenCalledTimes(1);
  });
});
```

### Running the test suite

```bash
npm test              # unit + integration tests
npm run test:e2e      # end-to-end tests against local harness
npm run test:coverage # coverage report
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     BlackRoad OS                         │
│                                                          │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────┐ │
│  │   Cron       │   │   Task       │   │   Stripe    │ │
│  │   Engine     │──▶│   Queue      │◀──│   Webhook   │ │
│  │  (cron-parser│   │  (priority   │   │   Handler   │ │
│  │   + leader   │   │   tiers)     │   │             │ │
│  │   election)  │   │              │   └─────────────┘ │
│  └──────┬───────┘   └──────┬───────┘                   │
│         │                  │                            │
│         ▼                  ▼                            │
│  ┌──────────────────────────────────┐                   │
│  │          Worker Pool             │                   │
│  │  (concurrency-limited executors) │                   │
│  └──────────────────────────────────┘                   │
│         │                                               │
│         ▼                                               │
│  ┌──────────────────────────────────┐                   │
│  │        Execution Logs + DLQ      │                   │
│  │    (Prism Console / API)         │                   │
│  └──────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

---

## BlackRoad OS Ecosystem

`@blackroad/job-scheduler` is one component of the BlackRoad OS platform. Related packages and services:

| Package / Service | Description |
|---|---|
| [`@blackroad/cron`](https://github.com/BlackRoad-OS/blackroad-cron) | Lightweight cron service on Cloudflare Workers |
| [`@blackroad/websocket-manager`](https://github.com/BlackRoad-OS/blackroad-websocket-manager) | Real-time WebSocket mesh |
| [`@blackroad/analytics`](https://github.com/BlackRoad-OS/blackroad-analytics) | API analytics dashboard |
| [Prism Console](https://github.com/BlackRoad-OS/blackroad-prism-console) | Unified management dashboard |
| [BlackRoad Public SDK](https://github.com/BlackRoad-OS/BlackRoad-Public) | Public SDKs and examples |

---

## Support & SLA

| Plan | Response Time | Channels |
|---|---|---|
| Community | Best-effort | [GitHub Issues](https://github.com/BlackRoad-OS/blackroad-job-scheduler/issues) |
| Professional | < 4 business hours | Email + Slack |
| Enterprise | < 1 hour, 24 / 7 | Dedicated Slack + Phone |

- **Documentation**: [docs.blackroad.io/job-scheduler](https://docs.blackroad.io/job-scheduler)
- **API Reference**: [api.blackroad.io](https://api.blackroad.io)
- **Status Page**: [status.blackroad.io](https://status.blackroad.io)
- **Support Portal**: [support.blackroad.io](https://support.blackroad.io)
- **Email**: [support@blackroad.io](mailto:support@blackroad.io)

---

## License

Copyright © 2024–2026 **BlackRoad OS, Inc.** All Rights Reserved.

This software is proprietary and confidential. Unauthorized copying, distribution, or modification is strictly prohibited. See [LICENSE](./LICENSE) for the full terms.

---

<p align="center">
  🖤 Built by <a href="https://blackroad.io"><strong>BlackRoad OS</strong></a>
</p>
