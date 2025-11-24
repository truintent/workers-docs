# TruIntent Workers Documentation

**Architecture:** 6 Independent Cloudflare Workers (Grandparent-Parent-Child)
**Version:** 6.0.0
**Last Updated:** November 20, 2025

---

## 🎯 Overview

This repository contains **root-level documentation** for the TruIntent multi-agent platform. Individual worker codebases are maintained in separate repositories.

```
                    TruAgent Orchestrator
                  (Grandparent - Routing Only)
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                   │
        ↓                  ↓                   ↓
   Research            Data                CRM
   Gateway           Gateway            Gateway
   (Parent)          (Parent)           (Parent)
        │                  │                   │
    ┌───┴────┐        ┌────┴─────┐       ┌────┴─────┐
    │ 3 DOs  │        │  6 DOs   │       │  7 DOs   │
    │(Child) │        │ (Child)  │       │ (Child)  │
    └────────┘        └──────────┘       └──────────┘

        ↓                  ↓
      CDP              Content
    Gateway           Gateway
   (Parent)          (Parent)
        │                  │
    ┌───┴────┐        ┌────┴─────┐
    │ 1 DO   │        │  2 DOs   │
    │(Child) │        │ (Child)  │
    └────────┘        └──────────┘
```

---

## 📦 Worker Repositories (Separate Repos)

| Worker | Repository | Status |
|--------|------------|--------|
| **tru-agent** | [github.com/truintent/tru-agent](https://github.com/truintent/tru-agent) | ✅ Live |
| **research-gateway** | [github.com/truintent/research-gateway](https://github.com/truintent/research-gateway) | ✅ Live |
| **data-gateway** | [github.com/truintent/data-gateway](https://github.com/truintent/data-gateway) | ✅ Live |
| **crm-gateway** | [github.com/truintent/crm-gateway](https://github.com/truintent/crm-gateway) | ✅ Live |
| **cdp-gateway** | [github.com/truintent/cdp-gateway](https://github.com/truintent/cdp-gateway) | ✅ Live |
| **content-gateway** | [github.com/truintent/content-gateway](https://github.com/truintent/content-gateway) | ✅ Live |

---

## 📚 Documentation Index

### Essential Guides

1. **[DEVELOPMENT_ROADMAP.md](DEVELOPMENT_ROADMAP.md)** ✨ NEWEST - START HERE!
   - **P0 Critical (9h)** - Deploy this week: Fix tests, security, rate limiting
   - **P1 High (67.5h)** - Deploy this sprint: Backend + Frontend optimizations
   - **P2 Medium (30h)** - Next sprint: Architecture improvements
   - Complete timeline, success metrics, developer onboarding
   - Expected improvements: 50% faster workflows, 60% faster initial load

2. **[GATEWAY_ARCHITECTURE_SPEC.md](GATEWAY_ARCHITECTURE_SPEC.md)**
   - AI model specifications for all 5 gateways
   - System prompt templates per gateway
   - Consistent scaffolding patterns
   - Deployment commands

3. **[CURRENT_STATE_SUMMARY.md](CURRENT_STATE_SUMMARY.md)**
   - What's deployed vs. what needs work
   - Actor deployment summary by gateway
   - Next steps priority list
   - Verification commands

4. **[WORKER_ARCHITECTURE.md](WORKER_ARCHITECTURE.md)**
   - Complete 6-worker architecture
   - Grandparent-Parent-Child hierarchy
   - Tool distribution strategy (why 5 gateways?)
   - Queue + Actor integration patterns

5. **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)**
   - Copy-paste deployment commands
   - Health check scripts
   - Common troubleshooting
   - Per-gateway curl examples

### Infrastructure Guides

5. **[CLOUDFLARE_QUEUES_WITH_DURABLE_OBJECTS.md](CLOUDFLARE_QUEUES_WITH_DURABLE_OBJECTS.md)**
   - Queue + Durable Object integration patterns
   - Dynamic queue creation via API
   - 3 architectural patterns (Queue→Actor, Multi-Actor routing, Actor→Queue events)
   - CDP Gateway refactoring integration

6. **[GIT_STRUCTURE_PLAN.md](GIT_STRUCTURE_PLAN.md)**
   - Git repository structure (6 worker repos + workers-docs)
   - Migration checklist from monorepo to separate repos
   - Commit pending changes across all repos

---

## 🚀 Quick Start

### Clone All Worker Repos

```bash
mkdir truintent-workers
cd truintent-workers

gh repo clone truintent/tru-agent
gh repo clone truintent/research-gateway
gh repo clone truintent/data-gateway
gh repo clone truintent/crm-gateway
gh repo clone truintent/cdp-gateway
gh repo clone truintent/content-gateway
```

### Deploy All Workers

```bash
export CLOUDFLARE_API_TOKEN="your-token-here"

for gateway in tru-agent research-gateway data-gateway crm-gateway cdp-gateway content-gateway; do
  cd $gateway
  npx wrangler deploy
  cd ..
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

## 🏗️ Architecture Summary

### Grandparent: tru-agent
- **Purpose:** Orchestrator (routing only, no AI model)
- **Features:** OAuth 2.0, React SPA, 6 CrewAI workflows, WebSocket/SSE
- **URL:** https://tru-agent.datasnyperai.workers.dev
- **Frontend:** https://mcp.truintent.io

### Parents: 5 Specialist Gateways

| Gateway | Model | Tools | Actors | Purpose |
|---------|-------|-------|--------|---------|
| **research-gateway** | gemini-2.5-flash | 12 | 3 | Web research, scraping, KB queries |
| **data-gateway** | gpt-5-mini | 8 | 6 | SQL, B2B audiences, data validation |
| **crm-gateway** | gpt-5-mini | 10 | 7 | GoHighLevel, HubSpot, Instantly |
| **cdp-gateway** | gpt-5-mini | 10 | 5 | Customer data, scoring, audiences |
| **content-gateway** | gpt-5-mini | 15 | 2 | Copywriting, automation, browser tools |

### Children: Durable Objects (Cloudflare Actors)

**Total:** 23 actors planned, 13 deployed (57%)

- **Research (3):** WebScraperActor, DataParserActor, SummarizerActor
- **Data (6):** EmailValidatorActor, PhoneValidatorActor, DataCleanerActor, DataDeduplicatorActor, SQLQueryActor, TaxonomicLensesActor
- **CRM (7):** GHLContactActor, GHLCampaignActor, GHLTagActor, GHLNoteActor, GHLOpportunityActor, GHLCustomFieldActor, WorkflowActor
- **CDP (5):** ProfileScorerActor, WorkflowOrchestratorActor, AudienceBuilderActor, EnrollmentManagerActor, ProfileEnricherActor
- **Content (2):** NodeSandboxActor, FirecrawlScraperActor

---

## 📊 Current Migration Status

**Latest Deployment:** ProfileScorerActor (CDP Gateway) - Nov 20, 2025

| Gateway | Deployed Actors | In Progress | Total |
|---------|-----------------|-------------|-------|
| research-gateway | 3 | 0 | 3 ✅ |
| data-gateway | 6 | 0 | 6 ✅ |
| crm-gateway | 1 | 6 | 7 ⏳ |
| cdp-gateway | 1 | 4 | 5 ⏳ |
| content-gateway | 2 | 0 | 2 ✅ |

**Next Priority:** Complete CRM Gateway Phase 3 (6 more GHL actors)

---

## 🔑 Shared Infrastructure

All 5 gateways share:

### D1 Database
```toml
[[d1_databases]]
binding = "DB"
database_name = "ai-gateway-conversations"
database_id = "393031ae-3868-4d92-9a0a-825035b58c19"
```

### KV Cache (Per-Gateway Keys)
```toml
[[kv_namespaces]]
binding = "CACHE"
id = "5e5c27b2d1424088999e86a3a84fb677"
```

Cache keys: `mcp_tools_v3_{gateway_name}`

### R2 Buckets
- **PROJECT_DOCS** - Project file storage (max 100MB per file)

---

## 💡 Why Separate Repos?

**Benefits:**
- ✅ Independent CI/CD per worker
- ✅ Clear ownership boundaries
- ✅ Deploy one worker without affecting others
- ✅ Easier code review (smaller PRs)

**Trade-offs:**
- ❌ Root docs need separate repo (this one!)
- ❌ 6 repos to manage instead of 1
- ❌ Cross-repo changes require coordination

**Decision:** Separate repos provide better isolation and deployment independence, worth the coordination overhead.

---

## 🛠️ Development Workflow

### Making Changes to a Worker

```bash
# 1. Clone the worker repo
gh repo clone truintent/crm-gateway
cd crm-gateway

# 2. Make changes
vim actors/crm/ghl-contact.js

# 3. Test locally
npx wrangler dev

# 4. Deploy to staging (if available)
npx wrangler deploy --env staging

# 5. Deploy to production
npx wrangler deploy

# 6. Commit and push
git add -A
git commit -m "Add GHLCampaignActor with stats tracking"
git push origin main
```

### Cross-Worker Changes

If a change affects multiple workers (e.g., shared tool interface):

1. Create feature branch in each affected repo
2. Make coordinated changes
3. Test all workers together
4. Deploy in sequence (data dependencies first)
5. Merge all PRs together

---

## 📞 Support

- **Architecture Questions:** See [WORKER_ARCHITECTURE.md](WORKER_ARCHITECTURE.md)
- **Deployment Issues:** See [QUICK_REFERENCE.md](QUICK_REFERENCE.md)
- **Git Structure:** See [GIT_STRUCTURE_PLAN.md](GIT_STRUCTURE_PLAN.md)
- **AI Model Specs:** See [GATEWAY_ARCHITECTURE_SPEC.md](GATEWAY_ARCHITECTURE_SPEC.md)

---

**Repository:** https://github.com/truintent/workers-docs
**Platform Version:** 6.0.0
**Last Updated:** November 20, 2025
