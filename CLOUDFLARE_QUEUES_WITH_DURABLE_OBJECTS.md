# Cloudflare Queues + Durable Objects Integration Guide

**Date:** November 20, 2025
**Purpose:** Comprehensive guide for using Cloudflare Queues with Durable Objects (Actors) for event-driven workflows
**Status:** Production-Ready Patterns

---

## Overview

This guide shows how to integrate **Cloudflare Queues** with **Durable Objects (Actors)** to build scalable, event-driven workflows. Queues decouple producers from consumers, enabling:

- ✅ **Asynchronous processing** - Long-running tasks don't block HTTP responses
- ✅ **Retries & DLQ** - Automatic retries with dead letter queue fallback
- ✅ **Load leveling** - Handle traffic spikes gracefully
- ✅ **Actor coordination** - Trigger actor workflows via queue messages
- ✅ **Dynamic queue creation** - Create queues programmatically at runtime

---

## Architecture Patterns

### Pattern 1: Queue → Single Actor (CDP Gateway Model)

**Use Case:** Workflow orchestration where a single orchestrator actor coordinates child actors

```
HTTP Request → Queue Producer → CDP_COMMAND_QUEUE
                                       ↓
                        Queue Consumer (core/index.js)
                                       ↓
                     WorkflowOrchestratorActor (Durable Object)
                                       ↓
              Child Actors (ProfileScorer, AudienceBuilder, etc.)
```

**Example:**
```javascript
// 1. Producer: Send workflow to queue (from HTTP handler)
await env.CDP_COMMAND_QUEUE.send({
  operation: 'execute_workflow',
  workflowId: 'workflow_123',
  pipeline: [
    { actor: 'ProfileScorerActor', operation: 'calculate_fit_scores' },
    { actor: 'AudienceBuilderActor', operation: 'create_audience_segment' }
  ],
  input_data: { profiles: [...] }
});

// 2. Consumer: Process queue message in core/index.js
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      const { workflowId, pipeline, input_data } = message.body;

      // Get Durable Object stub (NOT new instantiation!)
      const orchestratorId = env.WORKFLOW_ORCHESTRATOR_ACTOR.idFromName(workflowId);
      const orchestrator = env.WORKFLOW_ORCHESTRATOR_ACTOR.get(orchestratorId);

      // Invoke actor via RPC
      const result = await orchestrator.executePipeline(pipeline, input_data);

      // Message auto-acknowledged on success
    }
  }
};
```

### Pattern 2: Queue → Multiple Actor Types (Multi-Gateway Model)

**Use Case:** Different message types trigger different actors across gateways

```
HTTP Request → TASK_QUEUE
                   ↓
         Queue Consumer (route by type)
           /         |         \
    Research      Data        CRM
    Gateway       Gateway     Gateway
       ↓             ↓           ↓
  WebScraper   EmailValidator  GHLContact
  Actor        Actor           Actor
```

**Example:**
```javascript
// Producer: Send typed messages
await env.TASK_QUEUE.send({
  type: 'web_scraping',
  url: 'https://example.com',
  taskId: 'task_123'
});

await env.TASK_QUEUE.send({
  type: 'email_validation',
  emails: ['test@example.com'],
  taskId: 'task_124'
});

// Consumer: Route to appropriate actor
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      const { type, taskId } = message.body;

      switch (type) {
        case 'web_scraping':
          const scraper = env.WEB_SCRAPER_ACTOR.get(
            env.WEB_SCRAPER_ACTOR.idFromName(taskId)
          );
          await scraper.scrape(message.body.url);
          break;

        case 'email_validation':
          const validator = env.EMAIL_VALIDATOR_ACTOR.get(
            env.EMAIL_VALIDATOR_ACTOR.idFromName(taskId)
          );
          await validator.validate(message.body.emails);
          break;

        // ... more cases
      }

      message.ack(); // Explicitly acknowledge
    }
  }
};
```

### Pattern 3: Actor → Queue (Event Publishing)

**Use Case:** Actors publish events to queues for downstream processing

```
Actor completes work → Publishes to EVENT_QUEUE
                              ↓
                    Event Consumer (analytics, webhooks, etc.)
```

**Example:**
```javascript
// Inside WorkflowOrchestratorActor
import { Actor, Persist } from '@cloudflare/actors';

export class WorkflowOrchestratorActor extends Actor {
  @Persist workflowState = { status: 'pending', steps: [] };

  async executePipeline(pipeline, input) {
    this.workflowState.status = 'processing';

    for (const step of pipeline) {
      const result = await this.executeStep(step);
      this.workflowState.steps.push({ ...step, result, completedAt: Date.now() });
    }

    this.workflowState.status = 'completed';

    // Publish completion event to queue
    await this.env.CDP_EVENT_QUEUE.send({
      type: 'workflow.completed',
      workflowId: this.id,
      result: this.workflowState,
      timestamp: Date.now()
    });

    return this.workflowState;
  }
}
```

---

## Cloudflare Queues Configuration

### Static Configuration (wrangler.toml)

**Basic Setup:**
```toml
# Producer binding - Send messages to queue
[[queues.producers]]
binding = "MY_QUEUE"
queue = "my-queue-name"

# Consumer binding - Process messages from queue
[[queues.consumers]]
queue = "my-queue-name"
max_batch_size = 10       # Process up to 10 messages per batch
max_batch_timeout = 30    # Wait max 30 seconds before processing partial batch
max_retries = 3           # Retry failed messages 3 times
dead_letter_queue = "dlq" # Send failed messages to DLQ after max retries
```

**CDP Gateway Example:**
```toml
# Command queue (workflow requests)
[[queues.producers]]
binding = "CDP_COMMAND_QUEUE"
queue = "cdp-command-queue"

[[queues.consumers]]
queue = "cdp-command-queue"
max_batch_size = 10
max_batch_timeout = 30
max_retries = 3
dead_letter_queue = "cdp-dlq"

# Event queue (workflow completions)
[[queues.producers]]
binding = "CDP_EVENT_QUEUE"
queue = "cdp-event-queue"

# Dead letter queue (failed messages)
[[queues.producers]]
binding = "CDP_DLQ"
queue = "cdp-dlq"
```

### Dynamic Queue Creation (Runtime API)

**Use Case:** Create queues programmatically for multi-tenant systems or dynamic workflows

**API Endpoint:**
```
POST https://api.cloudflare.com/client/v4/accounts/{account_id}/queues
Authorization: Bearer {CLOUDFLARE_API_TOKEN}
Content-Type: application/json

{
  "queue_name": "tenant-workflow-queue-123"
}
```

**Example: Create Queue from Worker**
```javascript
async function createQueue(queueName, env) {
  const accountId = env.CLOUDFLARE_ACCOUNT_ID;
  const apiToken = env.CLOUDFLARE_API_TOKEN;

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/queues`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ queue_name: queueName })
    }
  );

  const result = await response.json();

  if (!result.success) {
    throw new Error(`Failed to create queue: ${result.errors[0]?.message}`);
  }

  return result.result; // Queue object with queue_id, created_on, etc.
}

// Usage
const queue = await createQueue('tenant-workflow-queue-abc123', env);
console.log('Created queue:', queue.queue_id);
```

**Multi-Tenant Pattern:**
```javascript
// Create per-tenant queues dynamically
export default {
  async fetch(request, env) {
    const { tenantId } = await request.json();
    const queueName = `tenant-${tenantId}-queue`;

    // Check if queue exists (store queue_id in KV or D1)
    let queueId = await env.KV.get(`queue:${tenantId}`);

    if (!queueId) {
      // Create queue via API
      const queue = await createQueue(queueName, env);
      queueId = queue.queue_id;

      // Store queue_id for future lookups
      await env.KV.put(`queue:${tenantId}`, queueId);
    }

    // Send message to tenant-specific queue
    // Note: Dynamic queues require direct API calls (no binding)
    await sendToQueue(queueId, { task: 'process_data' }, env);

    return new Response('Task queued', { status: 202 });
  }
};

async function sendToQueue(queueId, message, env) {
  // Cloudflare Queues API for sending messages
  // (Use wrangler queue producer bindings for static queues)
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${env.CLOUDFLARE_ACCOUNT_ID}/queues/${queueId}/messages`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${env.CLOUDFLARE_API_TOKEN}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ messages: [{ body: message }] })
    }
  );

  return response.json();
}
```

---

## Queue JavaScript API

### Producer API

**Send Single Message:**
```javascript
await env.MY_QUEUE.send({
  taskId: 'task_123',
  operation: 'process',
  data: { ... }
});
```

**Send with Delay:**
```javascript
await env.MY_QUEUE.send(
  { taskId: 'task_123' },
  { delaySeconds: 60 } // Delay delivery by 60 seconds
);
```

**Send Batch:**
```javascript
await env.MY_QUEUE.sendBatch([
  { body: { taskId: 'task_1', operation: 'process' } },
  { body: { taskId: 'task_2', operation: 'validate' } },
  { body: { taskId: 'task_3', operation: 'enrich' }, delaySeconds: 30 }
]);
```

**Content Type (for Dashboard Preview):**
```javascript
await env.MY_QUEUE.send(
  { taskId: 'task_123' },
  { contentType: 'json' } // 'json', 'text', 'bytes', or 'v8'
);
```

### Consumer API

**Basic Consumer:**
```javascript
export default {
  async queue(batch, env) {
    // batch.messages is an array of Message objects
    for (const message of batch.messages) {
      try {
        await processMessage(message.body, env);
        // Message automatically acknowledged on success
      } catch (error) {
        console.error('Error processing message:', error);
        // Message will be retried automatically
        message.retry(); // Optional: explicit retry with custom delay
      }
    }
  }
};
```

**Explicit Acknowledgment:**
```javascript
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      const success = await processMessage(message.body, env);

      if (success) {
        message.ack(); // Explicitly acknowledge
      } else {
        message.retry({ delaySeconds: 60 }); // Retry with 1-minute delay
      }
    }
  }
};
```

**Batch Operations:**
```javascript
export default {
  async queue(batch, env) {
    try {
      // Process all messages together
      await processBatch(batch.messages.map(m => m.body), env);

      // Acknowledge entire batch
      batch.ackAll();
    } catch (error) {
      console.error('Batch processing failed:', error);

      // Retry entire batch
      batch.retryAll({ delaySeconds: 30 });
    }
  }
};
```

---

## Integration with Durable Objects (Actors)

### Correct Actor Invocation Pattern

**❌ WRONG - In-Memory Instantiation:**
```javascript
// This creates a plain JavaScript object, NOT a Durable Object!
const actor = new WorkflowOrchestratorActor(taskId, workflowId);
const result = await actor.executePipeline(pipeline, input);
```

**✅ RIGHT - Durable Object Stub:**
```javascript
// Get DO stub via binding
const actorId = env.WORKFLOW_ORCHESTRATOR_ACTOR.idFromName(workflowId);
const actor = env.WORKFLOW_ORCHESTRATOR_ACTOR.get(actorId);

// Invoke via RPC
const result = await actor.executePipeline(pipeline, input);
```

### Queue Consumer with Multiple Actors

**CDP Gateway Pattern:**
```javascript
// core/index.js - Queue consumer
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      const { operation, workflowId, pipeline, input_data } = message.body;

      try {
        // Route to appropriate handler
        switch (operation) {
          case 'execute_workflow':
            await executeWorkflow(workflowId, pipeline, input_data, env);
            break;

          case 'retry_failed_step':
            await retryStep(workflowId, message.body.stepIndex, env);
            break;

          case 'cancel_workflow':
            await cancelWorkflow(workflowId, env);
            break;

          default:
            console.error('Unknown operation:', operation);
            message.retry({ delaySeconds: 5 });
        }

        // Auto-acknowledged on successful completion
      } catch (error) {
        console.error('Error processing message:', error);

        if (message.attempts < 3) {
          message.retry({ delaySeconds: 30 }); // Exponential backoff
        } else {
          // Send to DLQ (configured in wrangler.toml)
          console.error('Max retries exceeded, sending to DLQ');
        }
      }
    }
  }
};

async function executeWorkflow(workflowId, pipeline, input, env) {
  // Get orchestrator DO stub
  const orchestratorId = env.WORKFLOW_ORCHESTRATOR_ACTOR.idFromName(workflowId);
  const orchestrator = env.WORKFLOW_ORCHESTRATOR_ACTOR.get(orchestratorId);

  // Execute pipeline via RPC
  const result = await orchestrator.executePipeline(pipeline, input);

  // Update D1 with results
  await env.DB.prepare(
    'UPDATE cdp_workflows SET status = ?, result_data = ?, completed_at = ? WHERE id = ?'
  )
    .bind('completed', JSON.stringify(result), Date.now(), workflowId)
    .run();

  // Publish completion event
  await env.CDP_EVENT_QUEUE.send({
    type: 'workflow.completed',
    workflowId,
    result,
    timestamp: Date.now()
  });
}
```

### Actor Chain with Shared State (KV-Based)

**Pattern: Multiple actors share state via KV with same taskId**

```javascript
// WorkflowOrchestratorActor coordinates child actors
import { Actor, Persist } from '@cloudflare/actors';

export class WorkflowOrchestratorActor extends Actor {
  @Persist stats = { pipelines_executed: 0, steps_completed: 0 };

  async executePipeline(pipeline, input) {
    const taskId = `task_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;

    // Store initial input in KV (shared state)
    await this.env.CACHE.put(
      `${taskId}:input`,
      JSON.stringify(input),
      { expirationTtl: 3600 } // 1 hour
    );

    // Execute each step via separate actors
    for (const step of pipeline) {
      const result = await this.executeActorStep(step, taskId);

      // Store step result in KV
      await this.env.CACHE.put(
        `${taskId}:${step.actor}:output`,
        JSON.stringify(result),
        { expirationTtl: 3600 }
      );
    }

    this.stats.pipelines_executed++;
    this.stats.steps_completed += pipeline.length;

    return { success: true, taskId, stats: this.stats };
  }

  async executeActorStep(step, taskId) {
    const { actor, operation } = step;

    // Get child actor stub (all actors share same taskId for state)
    const actorStub = this.getActorStub(actor, taskId);

    // Invoke operation via RPC
    const result = await actorStub[operation]();

    return result;
  }

  getActorStub(actorName, taskId) {
    switch (actorName) {
      case 'ProfileScorerActor':
        return this.env.PROFILE_SCORER_ACTOR.get(
          this.env.PROFILE_SCORER_ACTOR.idFromName(taskId)
        );

      case 'AudienceBuilderActor':
        return this.env.AUDIENCE_BUILDER_ACTOR.get(
          this.env.AUDIENCE_BUILDER_ACTOR.idFromName(taskId)
        );

      // ... more actors
    }
  }
}

// Child actors read shared state from KV
export class ProfileScorerActor extends Actor {
  @Persist stats = { scored: 0, errors: 0 };

  async calculate_fit_scores() {
    // Read input from shared KV state
    const input = JSON.parse(
      await this.env.CACHE.get(`${this.id.toString()}:input`)
    );

    // Process profiles
    const scored = await this.scoreProfiles(input.profiles);

    this.stats.scored += scored.length;

    return { profiles: scored, stats: this.stats };
  }
}
```

---

## CDP Gateway Refactoring Example

### Phase 1: Add @cloudflare/actors Dependency

```bash
cd /home/matt/mcp-local-dev/truintent/workers/cdp-gateway
npm install @cloudflare/actors
```

### Phase 2: Refactor Actor Class

**Before (In-Memory Class):**
```javascript
// actors/workflow-orchestrator-actor.js
export class WorkflowOrchestratorActor {
  constructor(taskId, workflowId) {
    this.taskId = taskId;
    this.workflowId = workflowId;
  }

  async execute(operation, args, env) {
    // Manual state management
    const state = await this.getState(env);
    // ... operation logic
    await this.setState(state, env);
  }

  async setState(state, env) {
    await env.CACHE.put(`${this.taskId}:state`, JSON.stringify(state));
  }

  async getState(env) {
    const data = await env.CACHE.get(`${this.taskId}:state`, 'json');
    return data || {};
  }
}
```

**After (Durable Object with @cloudflare/actors):**
```javascript
// actors/workflow-orchestrator-actor.js
import { Actor, Persist } from '@cloudflare/actors';

export class WorkflowOrchestratorActor extends Actor {
  @Persist workflowState = { status: 'pending', steps: [], errors: [] };
  @Persist stats = { pipelines_executed: 0, steps_completed: 0, errors: 0 };

  // Public RPC methods (called via stub.methodName())
  async executePipeline(pipeline, input) {
    this.workflowState.status = 'processing';

    for (const step of pipeline) {
      try {
        const result = await this.executeActorStep(step);
        this.workflowState.steps.push({ ...step, result, completedAt: Date.now() });
        this.stats.steps_completed++;
      } catch (error) {
        this.workflowState.errors.push({ step, error: error.message });
        this.stats.errors++;
      }
    }

    this.workflowState.status = 'completed';
    this.stats.pipelines_executed++;

    return { success: true, state: this.workflowState, stats: this.stats };
  }

  async getWorkflowState() {
    return { state: this.workflowState, stats: this.stats };
  }

  async executeActorStep(step) {
    // Implementation (calls other actors via their stubs)
    const actorStub = this.getActorStub(step.actor, this.id.toString());
    return await actorStub[step.operation]();
  }
}
```

### Phase 3: Update wrangler.toml

```toml
# Durable Object Bindings
[[durable_objects.bindings]]
name = "WORKFLOW_ORCHESTRATOR_ACTOR"
class_name = "WorkflowOrchestratorActor"
script_name = "cdp-gateway"

[[durable_objects.bindings]]
name = "PROFILE_SCORER_ACTOR"
class_name = "ProfileScorerActor"
script_name = "cdp-gateway"

# ... more actor bindings

# Migrations (SQLite storage for @Persist)
[[migrations]]
tag = "v1"
new_sqlite_classes = [
  "WorkflowOrchestratorActor",
  "ProfileScorerActor",
  "AudienceBuilderActor",
  "EnrollmentManagerActor",
  "ProfileEnricherActor"
]
```

### Phase 4: Update Queue Consumer (core/index.js)

**Before (In-Memory Instantiation):**
```javascript
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      const { workflowId, pipeline, input_data } = message.body;

      // ❌ WRONG: Creates in-memory object, not Durable Object
      const orchestrator = new WorkflowOrchestratorActor(taskId, workflowId);
      const result = await orchestrator.execute('execute_pipeline', { pipeline, input_data }, env);
    }
  }
};
```

**After (Durable Object Stub):**
```javascript
// Export actor classes (REQUIRED for Cloudflare Workers)
export { WorkflowOrchestratorActor } from '../actors/workflow-orchestrator-actor.js';
export { ProfileScorerActor } from '../actors/profile-scorer-actor.js';
// ... more exports

export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      const { workflowId, pipeline, input_data } = message.body;

      // ✅ RIGHT: Get Durable Object stub
      const orchestratorId = env.WORKFLOW_ORCHESTRATOR_ACTOR.idFromName(workflowId);
      const orchestrator = env.WORKFLOW_ORCHESTRATOR_ACTOR.get(orchestratorId);

      // ✅ RIGHT: Invoke via RPC (no 'execute' wrapper method)
      const result = await orchestrator.executePipeline(pipeline, input_data);

      // Auto-acknowledged on success
    }
  }
};
```

---

## Best Practices

### 1. Actor ID Strategy

**Use deterministic IDs for consistent routing:**
```javascript
// ✅ GOOD: Same workflowId always gets same DO instance
const id = env.WORKFLOW_ORCHESTRATOR_ACTOR.idFromName(workflowId);

// ❌ BAD: Random IDs create new instances every time
const id = env.WORKFLOW_ORCHESTRATOR_ACTOR.newUniqueId();
```

### 2. Error Handling & Retries

**Implement exponential backoff:**
```javascript
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      try {
        await processMessage(message.body, env);
      } catch (error) {
        const delay = Math.min(60, Math.pow(2, message.attempts) * 5);
        message.retry({ delaySeconds: delay });
      }
    }
  }
};
```

### 3. State Persistence

**Use @Persist for automatic SQLite storage:**
```javascript
import { Actor, Persist } from '@cloudflare/actors';

export class MyActor extends Actor {
  // ✅ GOOD: Auto-persisted to SQLite
  @Persist stats = { processed: 0, errors: 0 };

  // ❌ BAD: Lost on instance restart
  // stats = { processed: 0, errors: 0 };
}
```

### 4. Queue Message Size

**Keep messages small (<128 KB):**
```javascript
// ✅ GOOD: Store large data in R2/D1, send reference
await env.R2.put(`data/${taskId}`, largeData);
await env.MY_QUEUE.send({ taskId, dataKey: `data/${taskId}` });

// ❌ BAD: Send large data directly
await env.MY_QUEUE.send({ taskId, largeData }); // May fail if >128 KB
```

### 5. Dead Letter Queue Handling

**Monitor and process DLQ messages:**
```javascript
// Separate worker to process DLQ
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      // Log for investigation
      await env.DB.prepare(
        'INSERT INTO failed_messages (message_id, body, error, timestamp) VALUES (?, ?, ?, ?)'
      )
        .bind(message.id, JSON.stringify(message.body), message.error, Date.now())
        .run();

      // Alert team
      await env.ALERT_QUEUE.send({
        type: 'dlq_message',
        message: message.body,
        attempts: message.attempts
      });

      message.ack(); // Don't retry DLQ messages
    }
  }
};
```

---

## Performance Considerations

### Batch Size Tuning

**For fast operations (<100ms each):**
```toml
[[queues.consumers]]
queue = "fast-queue"
max_batch_size = 100     # Process many messages at once
max_batch_timeout = 1    # Short timeout
```

**For slow operations (>1s each):**
```toml
[[queues.consumers]]
queue = "slow-queue"
max_batch_size = 5       # Process fewer at once
max_batch_timeout = 30   # Longer timeout
```

### Actor Instance Reuse

**Use consistent IDs to reuse instances:**
```javascript
// ✅ GOOD: Same taskId = same DO instance (reused)
const id = env.MY_ACTOR.idFromName(taskId);

// Instance stays warm, stats persist
const actor = env.MY_ACTOR.get(id);
await actor.process(data); // Uses existing instance
```

### Parallel Processing

**Process independent messages in parallel:**
```javascript
export default {
  async queue(batch, env) {
    await Promise.all(
      batch.messages.map(async (message) => {
        try {
          await processMessage(message.body, env);
          message.ack();
        } catch (error) {
          message.retry();
        }
      })
    );
  }
};
```

---

## Monitoring & Debugging

### Queue Analytics

**Check queue metrics in dashboard:**
```bash
npx wrangler queues list
npx wrangler queues consumer list <queue-name>
```

**Log key metrics:**
```javascript
export default {
  async queue(batch, env) {
    const startTime = Date.now();

    for (const message of batch.messages) {
      await processMessage(message.body, env);
    }

    const duration = Date.now() - startTime;

    // Log to Analytics Engine
    env.ANALYTICS.writeDataPoint({
      blobs: ['queue-consumer'],
      doubles: [duration, batch.messages.length],
      indexes: [batch.queue]
    });
  }
};
```

### Actor State Inspection

**Add getStats() methods for debugging:**
```javascript
export class MyActor extends Actor {
  @Persist stats = { processed: 0, errors: 0 };

  async getStats() {
    return {
      stats: this.stats,
      actorId: this.id.toString(),
      timestamp: Date.now()
    };
  }
}

// Inspect from HTTP endpoint
app.get('/debug/actor/:id/stats', async (c) => {
  const actorId = c.req.param('id');
  const actor = c.env.MY_ACTOR.get(c.env.MY_ACTOR.idFromName(actorId));
  const stats = await actor.getStats();
  return c.json(stats);
});
```

---

## Migration Checklist

### CDP Gateway Refactoring

- [ ] **Phase 1: Setup**
  - [ ] `npm install @cloudflare/actors`
  - [ ] Refactor `ProfileScorerActor` (extend Actor, add @Persist)
  - [ ] Update `wrangler.toml` (DO binding + migration)
  - [ ] Export actor in `core/index.js`
  - [ ] Deploy and test

- [ ] **Phase 2: Orchestrator**
  - [ ] Refactor `WorkflowOrchestratorActor`
  - [ ] Update queue consumer to use DO stubs (NOT `new`)
  - [ ] Update `wrangler.toml` (add orchestrator binding)
  - [ ] Deploy and test end-to-end workflow

- [ ] **Phase 3: Remaining Actors**
  - [ ] Refactor `AudienceBuilderActor`
  - [ ] Refactor `EnrollmentManagerActor`
  - [ ] Refactor `ProfileEnricherActor`
  - [ ] Update `wrangler.toml` (all bindings + migration)
  - [ ] Deploy and test

- [ ] **Phase 4: Enhancements**
  - [ ] Add `@Persist` decorators to all stateful properties
  - [ ] Create `ACTOR_DEVELOPMENT_GUIDE.md`
  - [ ] Add actor stats endpoints for monitoring
  - [ ] Document queue message formats

---

## Resources

### Cloudflare Documentation

- **Queues Overview:** https://developers.cloudflare.com/queues/
- **Queues API:** https://developers.cloudflare.com/api/resources/queues/
- **Queues JavaScript API:** https://developers.cloudflare.com/queues/configuration/javascript-apis/
- **Durable Objects:** https://developers.cloudflare.com/durable-objects/
- **Cloudflare Actors:** https://developers.cloudflare.com/actors/

### Internal Documentation

- **CDP Gateway Refactoring Plan:** `cdp-gateway/CDP_GATEWAY_ACTOR_REFACTORING_PLAN.md`
- **Actor Framework Adoption:** `/truintent/docs/ACTOR_FRAMEWORK_ADOPTION.md`
- **Queue Consumer Validation:** `cdp-gateway/QUEUE_CONSUMER_VALIDATED.md`

---

**Document Version:** 1.0
**Date:** November 20, 2025
**Author:** Claude Code (AI Assistant)
**Status:** Ready for Implementation
