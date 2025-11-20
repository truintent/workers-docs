# TruIntent Workers Architecture

**Date:** November 20, 2025
**Architecture:** Grandparent-Parent-Child (6 Independent Workers)
**Deployment:** Each Worker folder deploys separately

---

## 🎯 Core Principle

**Each Worker folder is an independent deployment unit** with:
- Own `wrangler.toml` (bindings, routes, DO migrations)
- Own `package.json` (dependencies)
- Own `actors/` folder (Durable Objects housed in this Worker)
- Own deployment cycle (`cd workers/{name} && npx wrangler deploy`)

---

## 📂 Worker Folder Structure

```
workers/
├── deploy-all.sh                    # Deploys all 6 workers
├── tru-agent/                       # Grandparent (Orchestrator)
│   ├── wrangler.toml
│   ├── package.json
│   ├── core/index.openapi.js
│   └── (no actors - just delegates)
│
├── research-gateway/                # Parent 1
│   ├── wrangler.toml
│   ├── package.json
│   ├── actors/
│   │   ├── web-scraper-actor.js
│   │   ├── data-parser-actor.js
│   │   └── summarizer-actor.js
│   └── core/index.ts
│
├── data-gateway/                    # Parent 2
│   ├── wrangler.toml
│   ├── package.json
│   ├── actors/data/
│   │   ├── email-validator.js
│   │   ├── phone-validator.js
│   │   ├── data-cleaner.js
│   │   ├── data-deduplicator.js
│   │   ├── sql-query.js
│   │   └── taxonomic-lenses.js
│   └── core/index.ts
│
├── crm-gateway/                     # Parent 3
│   ├── wrangler.toml
│   ├── package.json
│   ├── actors/crm/
│   │   ├── ghl-contact.js          # ✅ Deployed
│   │   ├── ghl-campaign.js         # ⏳ In Progress
│   │   ├── ghl-workflow.js         # ⏳ Planned
│   │   ├── ghl-tag.js
│   │   ├── ghl-note.js
│   │   ├── ghl-opportunity.js
│   │   └── ghl-customfield.js
│   └── core/index.js
│
├── cdp-gateway/                     # Parent 4
│   ├── wrangler.toml
│   ├── package.json
│   ├── actors/
│   │   ├── workflow-orchestrator-actor.js
│   │   ├── profile-scorer-actor.js
│   │   ├── audience-builder-actor.js
│   │   ├── enrollment-manager-actor.js
│   │   └── profile-enricher-actor.js
│   └── core/index.js
│
└── content-gateway/                 # Parent 5
    ├── wrangler.toml
    ├── package.json
    ├── actors/
    │   ├── node-sandbox-actor.js
    │   └── firecrawl-scraper-actor.js
    └── core/index.ts
```

---

## 🏗️ Grandparent-Parent-Child Hierarchy

### Level 1: Grandparent (tru-agent)

**Purpose:** Orchestrator that routes requests to specialist gateways

**Responsibilities:**
- Intent classification → gateway selection
- Multi-gateway CrewAI workflows
- OAuth 2.0 authentication
- React SPA frontend

**No Actors:** tru-agent doesn't house any Durable Objects (just delegates)

**URL:** https://tru-agent.datasnyperai.workers.dev

### Level 2: Parents (5 Specialist Gateways)

Each gateway is a **domain-specific AI specialist** that houses its own actors:

| Gateway | Domain | Tools | Actors | Status |
|---------|--------|-------|--------|--------|
| **research-gateway** | Web research, scraping, knowledge base | 12 | 3 | ✅ Production |
| **data-gateway** | SQL, B2B audiences, validation | 8 | 6 | ✅ Production |
| **crm-gateway** | GoHighLevel, HubSpot, Instantly | 10 | 7 (1 deployed) | ⏳ Phase 3 Active |
| **cdp-gateway** | Customer data, workflows, campaigns | 8 | 5 | ✅ Production |
| **content-gateway** | Copywriting, automation, files | 15 | 2 | ✅ Production |

**Key Feature:** Each gateway has **3-15 tools** (not 49+) for optimal LLM performance

### Level 3: Grandchildren (Cloudflare Actors)

**Actors are Durable Objects** bound to their parent gateway via `wrangler.toml`:

```toml
# Example: crm-gateway/wrangler.toml
[[durable_objects.bindings]]
name = "GHLContactActor"
class_name = "GHLContactActor"
script_name = "crm-gateway"  # ← HOUSED IN CRM GATEWAY

[[migrations]]
tag = "v1"
new_sqlite_classes = ["GHLContactActor"]
```

**What This Means:**
- `GHLContactActor` **runs inside** the crm-gateway Worker
- Has its own **SQLite storage** (via `@Persist` decorator)
- **Not shared** with other Workers (isolated)
- Communicates via **RPC** (Durable Object stubs)

---

## 🔄 How Actors Work

### 1. Actor Instantiation (RPC, Not In-Memory)

**WRONG (in-memory class):**
```javascript
const actor = new GHLContactActor(taskId, workflowId);  // ❌ Not how it works
```

**RIGHT (Durable Object RPC):**
```javascript
import { GHLContactActor } from '../actors/crm/ghl-contact.js';

// Cloudflare Actors framework provides .get() method
const actor = GHLContactActor.get(taskId);  // ✅ RPC stub

// Method call triggers RPC to Durable Object instance
const result = await actor.create(contact, location);  // ✅ RPC call
```

### 2. Actor Communication Patterns

**Within Same Gateway (Local RPC):**
```javascript
// crm-gateway: GHLContactActor → GHLTagActor
const taskId = `workflow_${Date.now()}`;

const creator = GHLContactActor.get(taskId);
const contact = await creator.create(data, location);

const tagger = GHLTagActor.get(taskId);
await tagger.addTags(contact.id, ['Lead', 'Cold']);
```

**Across Gateways (HTTP):**
```javascript
// tru-agent → crm-gateway (HTTP fetch)
const response = await fetch('https://crm-gateway.datasnyperai.workers.dev/query', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    tool: 'ghl_contact_operations',
    args: { operation: 'create', ... }
  })
});
```

**Shared State (D1 Database):**
```javascript
// All gateways share same D1 database
await env.DB.prepare('SELECT * FROM conversations WHERE id = ?')
  .bind(conversationId)
  .first();
```

### 3. Actor State Persistence

**@Persist Decorator (SQLite Storage):**
```javascript
import { Actor, Persist } from '@cloudflare/actors';

export class GHLContactActor extends Actor {
  @Persist contactStats = { created: 0, updated: 0, errors: 0 };

  async create(contact, location) {
    this.contactStats.created++;  // ← Persists to SQLite!
    // ... API call
    return { success: true, stats: this.contactStats };
  }
}
```

**Why SQLite (not KV)?**
- ✅ Automatic persistence (no manual `setState()`)
- ✅ ACID transactions
- ✅ Query support (SQL)
- ✅ Lower latency (local to DO instance)

---

## 🚀 Deployment Model

### Deploy Single Worker

```bash
cd /home/matt/mcp-local-dev/truintent/workers/crm-gateway
npx wrangler deploy
```

**What Happens:**
1. Builds Worker code (core/index.js + tools + actors)
2. Uploads to Cloudflare
3. Creates/updates Durable Object classes listed in `wrangler.toml`
4. Applies migrations (new_sqlite_classes)
5. Updates routes and bindings

### Deploy All Workers

```bash
cd /home/matt/mcp-local-dev/truintent/workers
./deploy-all.sh
```

**What Happens:**
```bash
# Deploys each Worker sequentially:
cd tru-agent && npx wrangler deploy
cd research-gateway && npx wrangler deploy
cd data-gateway && npx wrangler deploy
cd crm-gateway && npx wrangler deploy
cd cdp-gateway && npx wrangler deploy
cd content-gateway && npx wrangler deploy
```

### Deploy Specific Worker

```bash
cd /home/matt/mcp-local-dev/truintent/workers
./deploy-all.sh crm-gateway  # Only crm-gateway
```

---

## 🔗 Bindings Configuration

### Durable Objects (Per Worker)

Each Worker's `wrangler.toml` defines **which actors it houses**:

**crm-gateway/wrangler.toml:**
```toml
[[durable_objects.bindings]]
name = "GHLContactActor"
class_name = "GHLContactActor"
script_name = "crm-gateway"  # ← THIS WORKER

[[durable_objects.bindings]]
name = "GHLCampaignActor"
class_name = "GHLCampaignActor"
script_name = "crm-gateway"  # ← THIS WORKER

[[migrations]]
tag = "v1"
new_sqlite_classes = ["GHLContactActor", "GHLCampaignActor"]
```

**data-gateway/wrangler.toml:**
```toml
[[durable_objects.bindings]]
name = "EmailValidatorActor"
class_name = "EmailValidatorActor"
script_name = "data-gateway"  # ← THIS WORKER

[[migrations]]
tag = "v1"
new_sqlite_classes = ["EmailValidatorActor", "PhoneValidatorActor", ...]
```

### Shared Bindings (All Workers)

**All 5 specialist gateways share:**

```toml
# D1 Database (conversations, projects, files)
[[d1_databases]]
binding = "DB"
database_name = "ai-gateway-conversations"
database_id = "393031ae-3868-4d92-9a0a-825035b58c19"

# KV Cache (per-gateway keys)
[[kv_namespaces]]
binding = "CACHE"
id = "5e5c27b2d1424088999e86a3a84fb677"
```

**Why Shared?**
- Agents can access conversation context from other specialists
- Seamless handoffs (Research → Data → CRM)
- Single source of truth for workflow state

---

## 📊 Actor Distribution by Gateway

### research-gateway (3 actors)

| Actor | Purpose | Status |
|-------|---------|--------|
| WebScraperActor | Firecrawl scraping with state | ✅ Deployed |
| DataParserActor | Extract structured data | ✅ Deployed |
| SummarizerActor | Summarize research findings | ✅ Deployed |

### data-gateway (6 actors)

| Actor | Purpose | Status |
|-------|---------|--------|
| EmailValidatorActor | Email validation with stats | ✅ Deployed |
| PhoneValidatorActor | Phone validation with stats | ✅ Deployed |
| DataCleanerActor | Normalize and clean data | ✅ Deployed |
| DataDeduplicatorActor | Find and merge duplicates | ✅ Deployed |
| SQLQueryActor | Execute SQL with caching | ✅ Deployed |
| TaxonomicLensesActor | Industry/vertical search | ✅ Deployed |

### crm-gateway (7 actors - 1 deployed, 6 in progress)

| Actor | Purpose | Status |
|-------|---------|--------|
| GHLContactActor | GoHighLevel contact CRUD | ✅ Deployed Nov 19 |
| GHLCampaignActor | Campaign management | ⏳ In Progress |
| GHLWorkflowActor | Workflow automation | ⏳ Planned |
| GHLTagActor | Tag operations | ⏳ Planned |
| GHLNoteActor | Note management | ⏳ Planned |
| GHLOpportunityActor | Opportunity tracking | ⏳ Planned |
| GHLCustomFieldActor | Custom field operations | ⏳ Planned |

### cdp-gateway (5 actors)

| Actor | Purpose | Status |
|-------|---------|--------|
| WorkflowOrchestratorActor | Pipeline coordination | ✅ Deployed |
| ProfileScorerActor | ICP fit scoring | ✅ Deployed |
| AudienceBuilderActor | Segment creation | ✅ Deployed |
| EnrollmentManagerActor | Campaign enrollment | ✅ Deployed |
| ProfileEnricherActor | Data enrichment | ✅ Deployed |

### content-gateway (2 actors)

| Actor | Purpose | Status |
|-------|---------|--------|
| NodeSandboxActor | Node.js code execution | ✅ Deployed |
| FirecrawlScraperActor | Web scraping | ✅ Deployed |

### ⚠️ cdp-gateway (5 actors - REFACTORING IN PROGRESS)

| Actor | Purpose | Current State | Target State |
|-------|---------|---------------|--------------|
| WorkflowOrchestratorActor | Pipeline coordination | ❌ In-memory class | ⏳ Phase 2 - Refactoring |
| ProfileScorerActor | Calculate fit/intent scores | ❌ In-memory class | ⏳ Phase 1 - Next |
| AudienceBuilderActor | Segment creation | ❌ In-memory class | ⏳ Phase 3 - Planned |
| EnrollmentManagerActor | Campaign enrollment | ❌ In-memory class | ⏳ Phase 3 - Planned |
| ProfileEnricherActor | Data enrichment | ❌ In-memory class | ⏳ Phase 3 - Planned |

**⚠️ IMPORTANT - CDP Gateway Exception:**

The CDP Gateway actors are **NOT yet true Durable Objects**. They are currently implemented as plain JavaScript classes that extend `BaseActor` and are instantiated with `new` in the queue consumer.

**Current Implementation (WRONG):**
```javascript
// core/index.js - Queue consumer
const orchestrator = new WorkflowOrchestratorActor(taskId, workflowId); // ❌ In-memory
const result = await orchestrator.execute('execute_pipeline', args, env);
```

**Target Implementation (CORRECT):**
```javascript
// core/index.js - Queue consumer
const orchestratorId = env.WORKFLOW_ORCHESTRATOR_ACTOR.idFromName(workflowId);
const orchestrator = env.WORKFLOW_ORCHESTRATOR_ACTOR.get(orchestratorId); // ✅ DO stub
const result = await orchestrator.executePipeline(args.pipeline, args.input_data);
```

**Why This Matters:**
- ❌ **No state persistence** - Stats lost on instance restart
- ❌ **No distribution** - All actors run in single Worker instance
- ❌ **30s CPU limit** - Long workflows fail
- ❌ **No instance reuse** - Performance penalty on every request

**Refactoring Plan:** See `cdp-gateway/CDP_GATEWAY_ACTOR_REFACTORING_PLAN.md`

**Status:** Phase 1 starting - ProfileScorerActor refactoring next

---

## 🔍 How to Verify Actor Distribution

### Check Actor Bindings

```bash
# View actor bindings in wrangler.toml
grep -A 3 "durable_objects.bindings" crm-gateway/wrangler.toml
```

**Output:**
```toml
[[durable_objects.bindings]]
name = "GHLContactActor"
class_name = "GHLContactActor"
script_name = "crm-gateway"  # ← Confirms actor lives in crm-gateway
```

### Test Actor RPC

```bash
# Test GHLContactActor (should be in crm-gateway)
curl -X POST https://crm-gateway.datasnyperai.workers.dev/query \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "ghl_contact_operations",
    "args": {
      "operation": "search",
      "searchCriteria": {"query": "test", "limit": 5},
      "location": {"id": "test", "apiKey": "test"}
    }
  }' | jq '.result.stats'
```

**Expected Response:**
```json
{
  "searched": 1,
  "created": 0,
  "updated": 0,
  "errors": 0
}
```

### Check Health Endpoints

```bash
# All gateways should list their actors
for gateway in research-gateway data-gateway crm-gateway cdp-gateway content-gateway; do
  echo "=== $gateway ==="
  curl -s https://$gateway.datasnyperai.workers.dev/health | jq -r '.actors'
  echo ""
done
```

---

## ⚡ Performance Benefits

### Independent Scaling

Each Worker scales independently:
- **crm-gateway** gets traffic spike → Cloudflare scales only crm-gateway
- **research-gateway** remains unaffected

### Isolated Failures

If one Worker crashes:
- Other Workers continue operating
- Actors in crashed Worker restart automatically (Durable Objects)

### Optimized Bundle Sizes

Each Worker only includes its dependencies:
- **crm-gateway:** GoHighLevel SDK, HubSpot SDK, Instantly SDK
- **data-gateway:** Zod validators, SQL builders
- **research-gateway:** Firecrawl SDK, Perplexity SDK

---

## 🛠️ Development Workflow

### Adding a New Actor

**1. Choose Parent Gateway:**
```bash
# Adding HubSpot actor → goes in crm-gateway
cd /home/matt/mcp-local-dev/truintent/workers/crm-gateway
```

**2. Create Actor Class:**
```bash
vim actors/crm/hubspot-contact.js
```

```javascript
import { Actor, Persist } from '@cloudflare/actors';

export class HubSpotContactActor extends Actor {
  @Persist contactStats = { created: 0, errors: 0 };

  async create(contact) {
    this.contactStats.created++;
    // API implementation
    return { success: true, stats: this.contactStats };
  }
}
```

**3. Update wrangler.toml:**
```toml
[[durable_objects.bindings]]
name = "HubSpotContactActor"
class_name = "HubSpotContactActor"
script_name = "crm-gateway"  # ← THIS WORKER

[[migrations]]
tag = "v2"  # ← Increment from v1
new_sqlite_classes = ["HubSpotContactActor"]
```

**4. Export from core/index.js:**
```javascript
export { HubSpotContactActor } from '../actors/crm/hubspot-contact.js';
```

**5. Deploy THIS WORKER:**
```bash
npx wrangler deploy
```

### Modifying Existing Actor

```bash
# Edit actor code
vim actors/crm/ghl-contact.js

# No wrangler.toml changes needed

# Deploy THIS WORKER
npx wrangler deploy

# Existing actor instances will use new code
```

---

## 🎯 Key Takeaways

1. **6 Independent Workers** - Each folder is a separate deployment unit
2. **Actors are Durable Objects** - Not in-memory classes
3. **script_name = Parent Gateway** - Binds actor to its parent Worker
4. **Per-Worker Deployment** - `cd workers/{name} && npx wrangler deploy`
5. **RPC Communication** - Actors use `Actor.get(id).method()` pattern
6. **SQLite State** - `@Persist` decorator stores data automatically
7. **Shared D1 Database** - Cross-gateway context sharing
8. **Independent Scaling** - Each Worker scales separately

---

**Document Version:** 1.0
**Date:** November 20, 2025
**Author:** Claude Code (AI Assistant)
**Status:** ✅ Architecture Validated and Documented
