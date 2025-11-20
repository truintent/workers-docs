# Gateway Architecture Specification

**Last Updated:** November 20, 2025
**Purpose:** Unified architecture spec for all 6 specialist gateway workers
**Status:** ✅ 5 Gateways Deployed, CDP Gateway Phase 1 Complete

---

## 🎯 Overview

TruIntent uses a **6-worker multi-agent architecture** where 1 orchestrator coordinates 5 specialist AI gateways. Each gateway is an AI agent with domain-specific models, system prompts, and focused toolsets.

```
                    TruAgent Orchestrator
                  (No AI Model - Routing Only)
                           │
                           │ HTTP Requests
                           ↓
     ┌──────────────────────────────────────────────────────┐
     │            5 Specialist AI Gateways                   │
     │         (Each with GPT-4o-mini + Domain Prompt)       │
     └──────────────────────────────────────────────────────┘
              │         │         │         │         │
              ↓         ↓         ↓         ↓         ↓
       Research    Data     CRM      CDP     Content
       Gateway   Gateway  Gateway  Gateway  Gateway
```

---

## 📊 Worker Comparison Table

| Worker | Model | System Prompt Focus | Tools | Actors | Purpose |
|--------|-------|---------------------|-------|--------|---------|
| **tru-agent** | None | - | 52 | MCP_AGENT | Orchestrator (routing only) |
| **research-gateway** | GPT-4o-mini | Research + Citations | 12 | 3 | Web research, scraping, knowledge base |
| **data-gateway** | GPT-4o-mini | Data + Validation | 8 | 6 | SQL, B2B audiences, data cleansing |
| **crm-gateway** | GPT-4o-mini | CRM + Campaigns | 10 | 7 | GoHighLevel, HubSpot, Instantly |
| **cdp-gateway** | GPT-4o-mini | Customer Data + Scoring | 10 | 1 | Profile management, scoring, audiences |
| **content-gateway** | GPT-4o-mini | Copywriting + Automation | 15 | 2 | AI copywriting, browser automation |

---

## 🏗️ Standard Gateway Architecture Pattern

All 5 specialist gateways follow this **consistent scaffolding pattern**:

### Directory Structure

```
{gateway-name}/
├── core/
│   └── index.js                 # Main entry point (export default, actor exports)
├── actors/
│   └── {domain}/                # Domain-specific actors
│       ├── actor1.js
│       └── actor2.js
├── tools/
│   ├── {domain}/                # Domain tools (delegate to actors)
│   ├── storage/                 # save_artifact, list_artifacts
│   ├── chat/                    # team_chat (optional)
│   └── communication/           # delegate_to_agent (optional)
├── wrangler.toml                # Worker config + bindings
└── package.json                 # Dependencies (@cloudflare/actors)
```

### core/index.js Pattern

```javascript
// 1. Export actors FIRST (required by wrangler.toml)
export { Actor1 } from '../actors/domain/actor1.js';
export { Actor2 } from '../actors/domain/actor2.js';

// 2. Import tools
import { tool1 } from '../tools/domain/tool1.js';
import { tool2 } from '../tools/domain/tool2.js';
import { saveArtifactTool } from '../tools/storage/save-artifact.js';

// 3. Export default worker
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // Health check endpoint
    if (url.pathname === '/health') {
      return Response.json({
        status: 'healthy',
        gateway: '{gateway-name}',
        version: '1.0.0',
        tools: ['tool1', 'tool2', 'save_artifact'],
        actors: ['Actor1', 'Actor2'],
        timestamp: new Date().toISOString()
      });
    }

    // /tools endpoint (MCP-style tool listing)
    if (url.pathname === '/tools' && request.method === 'GET') {
      return Response.json({
        tools: [
          { name: 'tool1', description: tool1.description, schema: tool1.schema },
          { name: 'tool2', description: tool2.description, schema: tool2.schema }
        ]
      });
    }

    // /query endpoint (tool execution)
    if (url.pathname === '/query' && request.method === 'POST') {
      const { tool, args } = await request.json();
      const toolHandlers = { tool1, tool2, save_artifact: saveArtifactTool };
      const handler = toolHandlers[tool];
      const result = await handler.handler(args, env, ctx);
      return Response.json({ success: true, tool, result });
    }

    // /history and /conversations endpoints (for compatibility)
    // ...

    return Response.json({ error: 'Not Found' }, { status: 404 });
  }
};
```

### wrangler.toml Pattern

```toml
name = "{gateway-name}"
main = "core/index.js"
compatibility_date = "2024-11-01"

# Durable Objects (Actors)
[[durable_objects.bindings]]
name = "ACTOR1"
class_name = "Actor1"
script_name = "{gateway-name}"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["Actor1", "Actor2"]

# D1 Database (Shared)
[[d1_databases]]
binding = "DB"
database_name = "ai-gateway-conversations"
database_id = "393031ae-3868-4d92-9a0a-825035b58c19"

# KV Cache (Per-gateway)
[[kv_namespaces]]
binding = "CACHE"
id = "5e5c27b2d1424088999e86a3a84fb677"

# Environment Variables
[vars]
OPENAI_API_URL = "https://api.openai.com/v1/responses"
OPENAI_MODEL = "gpt-4o-mini"
MAX_TURNS = "5"
MCP_CLOUD_URL = "https://mcp.truintent.io/mcp"
TOOL_FILTER = "{gateway-name}"  # "research" | "data" | "crm" | "cdp" | "content"
```

---

## 🤖 Worker Details

### 1. TruAgent (Orchestrator)

**URL:** https://tru-agent.datasnyperai.workers.dev
**Model:** None (routing only)
**System Prompt:** N/A

**Purpose:**
- OAuth 2.0 authentication (Google)
- React SPA frontend (https://mcp.truintent.io)
- 6 CrewAI multi-agent workflows
- WebSocket + SSE streaming
- Durable Objects (MCP_AGENT for persistent sessions)

**Architecture:**
- No AI orchestration (delegates to gateways)
- Uses service bindings to call specialist gateways
- Manages workflow state via D1 database

**Bindings:**
- MCP_AGENT (Durable Object)
- AI_CONVERSATIONS (D1)
- DNA_DB (D1)
- AUTH_DB (D1)
- TOOL_CACHE (KV)
- TOOL_QUEUE (Queue)
- PROJECT_DOCS (R2)
- VECTORIZE (Vector Index)
- AI (Cloudflare AI)

**Key Files:**
- `core/index.openapi.js` - Hono + OpenAPI REST API
- `features/` - Auth, projects, queues
- `orchestration/` - Workflow execution
- `frontend/chat-ui/` - React SPA

---

### 2. Research Gateway

**URL:** https://research-gateway.datasnyperai.workers.dev
**Model:** GPT-4o-mini
**System Prompt Focus:** Research accuracy + Citation tracking

**System Prompt:**
```
You are a research specialist AI focused on accurate web research and knowledge retrieval.

Your capabilities:
- Deep web research with Perplexity (citations required)
- Website scraping and mapping (Firecrawl)
- Knowledge base queries (Context7)
- YouTube transcript extraction
- Document analysis and summarization

Guidelines:
1. Always cite sources in your responses
2. Verify information from multiple sources when possible
3. Summarize long documents concisely
4. Extract key insights and actionable findings
5. Save important research as artifacts for future reference

Use the research tools to provide accurate, well-cited answers.
```

**Tools (12):**
- `perplexity_research` - Web research with citations
- `firecrawl_deep_research` - Deep web scraping
- `firecrawl_search` - Web search
- `firecrawl_map` - Site mapping
- `query_system_knowledge` - Internal KB queries
- `context7_docs` - Document Q&A
- `context7_resolve` - Context resolution
- `youtube_transcript` - Video transcripts
- `fetch_advanced` - Advanced HTTP requests
- `save_artifact` - Save research findings
- `list_artifacts` - List saved items
- `team_chat` - Team communication

**Actors (3):**
- `WebScraperActor` - Firecrawl scraping with state persistence
- `DataParserActor` - Extract structured data from raw content
- `SummarizerActor` - Summarize research findings

---

### 3. Data Gateway

**URL:** https://data-gateway.datasnyperai.workers.dev
**Model:** GPT-4o-mini
**System Prompt Focus:** Data validation + Quality assurance

**System Prompt:**
```
You are a data specialist AI focused on data validation, cleansing, and B2B audience intelligence.

Your capabilities:
- SQL query execution (read-only)
- B2B audience search (125K+ DNA Lenses audiences)
- Email and phone validation
- Data deduplication and cleansing
- Industry/vertical taxonomy search

Guidelines:
1. Validate all contact data before use
2. Clean and normalize data for consistency
3. Use taxonomic lenses for precise audience targeting
4. Execute SQL queries safely (read-only)
5. Track validation statistics for quality metrics

Focus on data quality and accurate audience targeting.
```

**Tools (8):**
- `sql_list_tables` - List database tables
- `sql_schema` - Get table schemas
- `sql_query` - Execute SQL queries (read-only)
- `dna_lenses` - B2B audience search (125K+ audiences)
- `taxonomic_lenses` - Industry/vertical search
- `agent_orchestrator` - Sub-agent delegation
- `query_system_knowledge` - Internal KB
- `save_artifact` - Save outputs

**Actors (6):**
- `EmailValidatorActor` - Email validation with stats tracking
- `PhoneValidatorActor` - Phone validation with stats tracking
- `DataCleanerActor` - Normalize and clean contact data
- `DataDeduplicatorActor` - Find and merge duplicate records
- `SQLQueryActor` - Execute SQL queries with caching
- `TaxonomicLensesActor` - Industry/vertical audience search

---

### 4. CRM Gateway

**URL:** https://crm-gateway.datasnyperai.workers.dev
**Model:** GPT-4o-mini
**System Prompt Focus:** CRM operations + Campaign management

**System Prompt:**
```
You are a CRM specialist AI focused on contact management and campaign execution.

Your capabilities:
- GoHighLevel CRM operations (contacts, campaigns, workflows, tags, notes, opportunities, custom fields)
- HubSpot contact and deal management
- Instantly.ai email campaign tracking
- Campaign status monitoring

Guidelines:
1. Always validate location ID and API key before CRM operations
2. Track operation statistics for monitoring
3. Use batch operations for bulk updates (>100 contacts)
4. Provide clear error messages with troubleshooting steps
5. Log all CRM modifications for audit trail

Focus on reliable CRM data management and campaign tracking.
```

**Tools (10):**
- `ghl_operations` - Legacy GoHighLevel operations
- `ghl_contact_operations` - GHL contact CRUD (actor-based)
- `ghl_campaign_operations` - GHL campaign management (actor-based)
- `ghl_tag_operations` - GHL tag operations (actor-based)
- `ghl_note_operations` - GHL note management (actor-based)
- `ghl_opportunity_operations` - GHL opportunity tracking (actor-based)
- `ghl_customfield_operations` - GHL custom field operations (actor-based)
- `hubspot_contacts` - HubSpot contact management
- `hubspot_deals` - HubSpot deal management
- `instantly_campaign_status` - Instantly.ai campaign tracking

**Actors (7):**
- `GHLContactActor` - GoHighLevel contact CRUD (✅ deployed)
- `GHLCampaignActor` - Campaign management (⏳ in progress)
- `GHLWorkflowActor` - Workflow automation (planned)
- `GHLTagActor` - Tag operations (planned)
- `GHLNoteActor` - Note management (planned)
- `GHLOpportunityActor` - Opportunity tracking (planned)
- `GHLCustomFieldActor` - Custom field operations (planned)

---

### 5. CDP Gateway

**URL:** https://cdp-gateway.datasnyperai.workers.dev
**Model:** GPT-4o-mini
**System Prompt Focus:** Customer data + Scoring + Audience building

**System Prompt:**
```
You are a customer data platform specialist AI focused on profile management, scoring, and audience building.

Your capabilities:
- Profile import and management
- ICP fit scoring (0-100)
- Intent scoring based on behavioral signals
- Engagement scoring from historical activity
- Composite scoring (weighted averages)
- Audience segmentation and building
- Campaign enrollment management
- Workflow orchestration for multi-step CDP operations

Guidelines:
1. Score profiles against ICP criteria (industry, company size, revenue, location, job title)
2. Track intent signals (website visits, content downloads, email opens, demo requests)
3. Measure engagement (email responses, meeting attendance, deal progression)
4. Build audiences based on score thresholds (high: ≥70, medium: 40-69, low: <40)
5. Orchestrate multi-actor workflows for complex CDP operations

Focus on accurate profile scoring and intelligent audience building.
```

**Tools (10):**
- `cdp_import_profiles` - Import profiles to CDP
- `cdp_import_profiles_async` - Queue-based large imports
- `cdp_query_profiles` - Query CDP profiles
- `cdp_query_profiles_async` - Async profile queries
- `cdp_create_audience` - Create audience segments
- `cdp_list_campaigns` - List active campaigns
- `cdp_list_workflows` - List available workflows
- `cdp_get_workflow_details` - Get workflow details
- `cdp_recommend_workflows` - Recommend workflows based on data
- `cdp_execute_workflow_async` - Execute modular workflow pipelines

**Actors (5 total, 1 deployed):**
- `ProfileScorerActor` - ICP fit, intent, engagement, composite scoring (✅ deployed)
- `WorkflowOrchestratorActor` - Multi-step workflow orchestration (planned)
- `AudienceBuilderActor` - Audience segmentation (planned)
- `EnrollmentManagerActor` - Campaign enrollment (planned)
- `ProfileEnricherActor` - Profile data enrichment (planned)

**Queue Architecture:**
- `cdp-command-queue` - Async workflow commands
- `cdp-event-queue` - Completion events
- `cdp-dlq` - Failed message queue

---

### 6. Content Gateway

**URL:** https://content-gateway.datasnyperai.workers.dev
**Model:** GPT-4o-mini
**System Prompt Focus:** Copywriting + Content creation + Automation

**System Prompt:**
```
You are a content specialist AI focused on AI copywriting, content creation, and web automation.

Your capabilities:
- AI copywriting (emails, landing pages, ads, social posts)
- Web scraping and data extraction (Firecrawl)
- Browser automation (Playwright)
- File operations (read, search)
- Python code execution (sandboxed)
- YouTube transcript extraction

Guidelines:
1. Write persuasive, on-brand copy tailored to target audience
2. Optimize content for specific platforms (email, landing page, LinkedIn, etc.)
3. Extract structured data from web pages accurately
4. Use browser automation for complex interactions
5. Save generated content as artifacts for reuse

Focus on high-quality content creation and intelligent automation.
```

**Tools (15):**
- `copy_agent` - AI copywriting (email, landing pages, ads)
- `firecrawl_scrape` - Web scraping
- `firecrawl_extract` - Data extraction
- `firecrawl_map` - Site mapping
- `browser_automation` - Playwright automation
- `perplexity_research` - Web research
- `youtube_transcript` - Video transcripts
- `fetch_advanced` - Advanced HTTP
- `context7_docs` - Document Q&A
- `read_file` - File reading
- `search_code` - Code search
- `python_execute` - Python code execution
- `team_chat` - Team communication
- `save_artifact` - Save outputs
- `list_artifacts` - List saved items

**Actors (2):**
- `NodeSandboxActor` - Sandboxed Node.js code execution
- `FirecrawlScraperActor` - Advanced web scraping

---

## 🔑 Shared Infrastructure

### D1 Database (Shared)

All 5 gateways share the **same D1 database** for cross-gateway context:

```toml
[[d1_databases]]
binding = "DB"
database_name = "ai-gateway-conversations"
database_id = "393031ae-3868-4d92-9a0a-825035b58c19"
```

**Tables:**
- `conversations` - Chat history across all gateways
- `messages` - Message content with tool calls
- `projects` - Project grouping
- `project_files` - File metadata (R2 storage)
- `cdp_profiles` - Customer Data Platform profiles
- `cdp_campaigns` - Campaign tracking
- `cdp_enrollments` - Profile-campaign relationships
- `cdp_events` - Event logging
- `cdp_workflows` - Workflow execution state

### KV Cache (Per-Gateway)

Each gateway has **unique cache keys** in shared KV namespace:

```toml
[[kv_namespaces]]
binding = "CACHE"
id = "5e5c27b2d1424088999e86a3a84fb677"
```

**Cache Key Pattern:**
- `mcp_tools_v3_research` - Research gateway tool cache
- `mcp_tools_v3_data` - Data gateway tool cache
- `mcp_tools_v3_crm` - CRM gateway tool cache
- `mcp_tools_v3_cdp` - CDP gateway tool cache
- `mcp_tools_v3_content` - Content gateway tool cache

**TTL:** 5 minutes per tool response

### R2 Buckets (Shared)

- **PROJECT_DOCS** - Project file storage (max 100MB per file)

### Secrets (Per-Gateway)

```bash
# Set for each gateway
OPENAI_API_KEY  # OpenAI API authentication
MCP_TOKEN       # MCP server authentication
```

---

## 📝 AI Model Configuration

### Current State (All Gateways)

```javascript
// In each gateway's orchestrator/AI client
const OPENAI_CONFIG = {
  model: 'gpt-4o-mini',
  temperature: 0.7,
  max_tokens: 4096,
  stream: true
};
```

### Future Model Diversification (Planned)

```toml
# research-gateway/wrangler.toml
[vars]
OPENAI_MODEL = "gpt-4o-mini"  # Fast, accurate research

# data-gateway/wrangler.toml
[vars]
OPENAI_MODEL = "gpt-4o-mini"  # Structured data analysis

# crm-gateway/wrangler.toml
[vars]
OPENAI_MODEL = "gpt-4o-mini"  # Reliable CRM operations

# cdp-gateway/wrangler.toml
[vars]
OPENAI_MODEL = "gpt-4o"       # Complex scoring logic (upgrade candidate)

# content-gateway/wrangler.toml
[vars]
OPENAI_MODEL = "gpt-4o"       # Creative copywriting (upgrade candidate)
```

**Rationale:**
- **Research, Data, CRM**: GPT-4o-mini is sufficient (speed + cost)
- **CDP, Content**: GPT-4o recommended for complex reasoning and creativity

---

## 🚀 Deployment Commands

### Deploy All Gateways

```bash
cd /home/matt/mcp-local-dev/truintent/workers
export CLOUDFLARE_API_TOKEN="your-token-here"
./deploy-all.sh
```

### Deploy Single Gateway

```bash
cd /home/matt/mcp-local-dev/truintent/workers/crm-gateway
npx wrangler deploy
```

### Set Secrets for All Gateways

```bash
export OPENAI_KEY=$(echo $OPENAI_API_KEY)
export MCP_TOKEN=$(cat ~/.mcp-token)

for gateway in research-gateway data-gateway crm-gateway cdp-gateway content-gateway; do
  echo "$OPENAI_KEY" | npx wrangler secret put OPENAI_API_KEY --name $gateway
  echo "$MCP_TOKEN" | npx wrangler secret put MCP_TOKEN --name $gateway
done
```

### Health Check All Gateways

```bash
for gateway in tru-agent research-gateway data-gateway crm-gateway cdp-gateway content-gateway; do
  echo "=== $gateway ==="
  curl -s https://$gateway.datasnyperai.workers.dev/health | jq -r '.status, .version'
  echo ""
done
```

---

## 📊 Performance Metrics

### Current Production Performance

| Gateway | Health Check | Actor Execution | Full Workflow |
|---------|--------------|-----------------|---------------|
| research-gateway | <5ms | 650ms | 3-8s |
| data-gateway | <5ms | 350ms | 2-5s |
| crm-gateway | <5ms | 650ms | 4-10s |
| cdp-gateway | <5ms | 450ms | 5-15s |
| content-gateway | <5ms | 800ms | 6-12s |

**Actor Benefits:**
- **47% faster** than stateless tools (9.5s → 5s)
- **Instance reuse** eliminates cold starts
- **State persistence** via @Persist decorator

### Resource Usage

- **D1 Database:** Shared across all 5 gateways
- **KV Cache:** 60-80% hit rate per gateway
- **Workers Cost:** ~$5/month per gateway ($30/month total)
- **Cold Start:** 45-50ms per gateway

---

## 🔒 Security & Authentication

### Gateway-to-Gateway Communication

Currently: **Public HTTPS URLs** (no service bindings)

```javascript
// In tru-agent workflow executor
const response = await fetch(`${gatewayConfig.gateway_url}/query`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(request)
});
```

**Benefits:**
- Simple architecture (no service bindings)
- Externally callable (n8n, Zapier, etc.)
- Standard HTTP/REST interface
- Easy monitoring and debugging

**Future Enhancement (Optional):**
- Add JWT token validation for gateway-to-gateway calls
- Rate limiting per gateway
- API key authentication for external access

---

## 📚 Next Steps

### Phase 1 (Complete)
- ✅ ProfileScorerActor deployed in CDP Gateway

### Phase 2 (Active)
- ⏳ Complete CRM Gateway Phase 3 (6 more GHL actors)
- ⏳ CDP Gateway Phase 2 (WorkflowOrchestratorActor)

### Phase 3 (Planned)
- [ ] CDP Gateway remaining actors (AudienceBuilder, EnrollmentManager, ProfileEnricher)
- [ ] Model diversification (upgrade CDP + Content to GPT-4o)
- [ ] System prompt optimization per gateway
- [ ] Cross-gateway workflow monitoring

---

**Document Version:** 1.0
**Date:** November 20, 2025
**Author:** Claude Code (AI Assistant)
**Purpose:** Spec AI models and system prompts for all specialist gateways
