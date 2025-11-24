# TruIntent Multi-Gateway Development Roadmap

**Last Updated:** 2025-11-24
**Scope:** All 5 Specialist Gateways + Orchestrator
**Architecture Score:** 7.5/10 (Production-Ready for 0-100 users)
**Status:** ✅ Production - Optimization Phase Active

---

## 📊 Executive Summary

The TruIntent platform uses a **multi-gateway specialist architecture** with 5 specialist gateways coordinated by a central orchestrator. The architecture is production-ready with strong foundations, validated by 40% performance gains and 60% fewer hallucinations.

**Current State:**
- **Backend:** 7.5/10 (strong foundations, optimization needed)
- **Frontend:** 8.2/10 (production-grade, needs design system)
- **Security:** 7/10 (3 critical vulnerabilities identified)
- **Performance:** 12s workflows (target: 6s with P0+P1)

**Development Priorities:**
1. **P0 Critical (9h)** - Security & CI/CD fixes - Deploy THIS WEEK
2. **P1 High (67.5h)** - Performance & UX optimizations - Deploy THIS SPRINT (3-4 weeks)
3. **P2 Medium (30h)** - Architecture improvements - NEXT SPRINT

---

## 🏗️ Repository Structure

```
/home/matt/mcp-local-dev/truintent/workers/
├── tru-agent/              # 🎯 Orchestrator (main entry point)
│   ├── core/               # REST API + OpenAPI
│   ├── orchestration/      # AI agentic loop
│   ├── features/           # Auth, projects, queues
│   ├── tools/              # 52 MCP tools
│   ├── workflows/          # 6 CrewAI workflows
│   └── frontend/chat-ui/   # React SPA
│
├── research-gateway/       # 🔍 Web Research (12 tools, 3 actors)
├── data-gateway/           # 📊 Data & Validation (8 tools, 6 actors)
├── crm-gateway/            # 📧 CRM Operations (10 tools, 7 actors)
├── cdp-gateway/            # 🗄️ Customer Data Platform (8 tools)
├── content-gateway/        # ✍️ Content & Automation (15 tools, 2 actors)
│
├── shared/                 # Shared utilities (model-config, etc.)
├── deploy-all.sh           # Deploy all workers at once
└── README.md               # Complete architecture overview
```

### Production URLs

| Worker | URL | Tools | Actors | Primary Focus |
|--------|-----|-------|--------|---------------|
| **tru-agent** | https://tru-agent.datasnyperai.workers.dev | 52 | MCP_AGENT | Orchestrator |
| **research-gateway** | https://research-gateway.datasnyperai.workers.dev | 12 | 3 | Web scraping, research |
| **data-gateway** | https://data-gateway.datasnyperai.workers.dev | 8 | 6 | SQL, validation |
| **crm-gateway** | https://crm-gateway.datasnyperai.workers.dev | 10 | 7 | GoHighLevel, HubSpot |
| **cdp-gateway** | https://cdp-gateway.datasnyperai.workers.dev | 8 | 0 | Customer profiles |
| **content-gateway** | https://content-gateway.datasnyperai.workers.dev | 15 | 2 | Copywriting, automation |

---

## 🚨 P0 - CRITICAL (Deploy This Week)

**Total Effort:** 9 hours
**Impact:** Unblocks CI/CD, fixes security vulnerabilities, prevents cost attacks
**Location:** Primarily `tru-agent/` with security fixes affecting all gateways

### 1. Fix Broken Test Suite (1-2 hours)

**File:** `tru-agent/tests/orchestrator.test.js:9`
**Issue:** Test imports non-existent function, blocking all CI/CD validation
**Assignable:** Junior Developer

```javascript
// BEFORE (Line 9)
import { runAgenticLoop } from '../orchestration/orchestrator.js';

// AFTER
import { Orchestrator } from '../orchestration/orchestrator.js';

// Update test cases to use class-based approach
const orchestrator = new Orchestrator(env);
const result = await orchestrator.orchestrate(request);
```

**Verification:**
```bash
cd tru-agent
npm test  # All tests should pass
```

---

### 2. Remove Credential Exposure - ALL GATEWAYS (4 hours)

**Files:**
- `tru-agent/orchestration/tool-executor.js:179`
- All gateway credential handling code

**Issue:** **CRITICAL SECURITY** - Credentials exposed as plaintext HTTP headers
**Assignable:** Senior Developer

**Current Code:**
```javascript
// ❌ CRITICAL: Credentials exposed in ALL gateway calls
for (const [key, value] of Object.entries(credentials)) {
  headers[`X-Credential-${key}`] = value; // Visible in logs!
}
```

**Solution:**
```javascript
// ✅ SECURE: Use encrypted service tokens (60s expiration)
import { SignJWT } from 'jose';

async function generateShortLivedServiceToken(userId, credentialHashes, env) {
  const secret = new TextEncoder().encode(env.JWT_SECRET);

  return await new SignJWT({
    sub: userId,
    credentials: credentialHashes, // Hashed, not plaintext
    type: 'service_token'
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('60s') // Short-lived
    .sign(secret);
}

// Replace credential headers with service token
headers['X-Service-Token'] = serviceToken; // No plaintext credentials
```

**Deploy to ALL gateways:**
```bash
cd /home/matt/mcp-local-dev/truintent/workers
./deploy-all.sh  # Deploys security fix to all 6 workers
```

---

### 3. Implement Rate Limiting - Orchestrator (4 hours)

**File:** `tru-agent/core/index.openapi.js:34`
**Issue:** No rate limiting = unlimited AI queries = cost attack vector
**Assignable:** Mid-Level Developer

```javascript
// Add KV-based rate limiter
async function rateLimiter(c, next) {
  const userId = c.get('user_id') || 'anonymous';
  const cacheKey = `ratelimit:${userId}:${Math.floor(Date.now() / 60000)}`;

  const count = await c.env.CACHE.get(cacheKey);
  const currentCount = parseInt(count || '0');

  if (currentCount >= 100) {
    return c.json({
      error: 'Rate limit exceeded',
      limit: 100,
      window: '1 minute',
      retry_after: 60 - (Date.now() % 60000) / 1000
    }, 429);
  }

  await c.env.CACHE.put(cacheKey, (currentCount + 1).toString(), {
    expirationTtl: 60
  });

  await next();
}

// Apply to all API routes
app.use('/api/*', rateLimiter);
```

**Success Criteria:**
- ✅ Rate limiter enforces 100 req/min per user
- ✅ 429 response includes retry_after header
- ✅ No impact on performance (<5ms overhead)

---

## 🎯 P1 - HIGH PRIORITY (Deploy This Sprint)

**Total Effort:** 67.5 hours (3-4 weeks)
**Impact:** 50% faster workflows, 60% faster initial load, improved UX consistency

### Backend Optimizations (15.5 hours) - Orchestrator

#### 1. Parallelize Tool Execution (1 hour)

**File:** `tru-agent/orchestration/orchestrator.js:606`
**ROI:** ⭐⭐⭐⭐⭐ (1 hour = 40% performance gain)

```javascript
// BEFORE: Sequential execution (3 tools × 2s = 6s)
for (const toolCall of toolCalls) {
  const result = await executeTool(toolCall);
  results.push(result);
}

// AFTER: Parallel execution (max(3 tools) = 2s)
const results = await Promise.all(
  toolCalls.map(tc => executeTool(tc))
);
```

**Impact:** Saves 3.9 seconds per multi-tool workflow

---

#### 2. Batch D1 Operations (2 hours)

**File:** `tru-agent/orchestration/conversation.js:254`
**Issue:** 4 separate D1 queries per message (440ms overhead)

```javascript
// BEFORE: 4 separate queries (440ms total)
await env.AI_CONVERSATIONS.prepare('INSERT INTO conversations ...').run();
await env.AI_CONVERSATIONS.prepare('INSERT INTO messages ...').run();
await env.AI_CONVERSATIONS.prepare('UPDATE conversations ...').run();
await env.AI_CONVERSATIONS.prepare('SELECT * FROM messages ...').all();

// AFTER: Single batch operation (atomic + faster)
const batch = [
  env.AI_CONVERSATIONS.prepare('INSERT INTO conversations ...'),
  env.AI_CONVERSATIONS.prepare('INSERT INTO messages ...'),
  env.AI_CONVERSATIONS.prepare('UPDATE conversations ...'),
  env.AI_CONVERSATIONS.prepare('SELECT * FROM messages ...')
];

const results = await env.AI_CONVERSATIONS.batch(batch);
```

**Impact:** Saves 440ms per message save operation

---

#### 3. Re-enable Tool Retry Logic (30 minutes)

**File:** `tru-agent/orchestration/orchestrator.js:615`

```javascript
// Currently commented out - re-enable with exponential backoff
const maxRetries = 3;
for (let i = 0; i < maxRetries; i++) {
  try {
    return await executeTool(toolCall);
  } catch (e) {
    if (i === maxRetries - 1) throw e;

    const backoffMs = Math.pow(2, i) * 1000; // 1s, 2s, 4s
    await new Promise(resolve => setTimeout(resolve, backoffMs));
  }
}
```

---

#### 4. Replace Custom JWT Implementation (12 hours)

**File:** `tru-agent/features/auth/oauth.js:14`
**Security Impact:** High - Vulnerable to algorithm confusion attacks

```bash
# Install jose library
cd tru-agent
npm install jose
```

```javascript
// Replace custom JWT with jose library
import { SignJWT, jwtVerify } from 'jose';

export async function createJWT(payload, env) {
  const secret = new TextEncoder().encode(env.JWT_SECRET);
  return await new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d')
    .sign(secret);
}
```

---

### Frontend Optimizations (52 hours) - tru-agent/frontend/chat-ui/

#### 5. Implement Code Splitting (4 hours)

**ROI:** ⭐⭐⭐⭐⭐ (4 hours = 60% faster load)
**Impact:** 5s → 2s initial load time

```typescript
// tru-agent/frontend/chat-ui/src/App.tsx
import { lazy, Suspense } from 'react';

const ChatPage = lazy(() => import('./pages/ChatPage'));
const WizardPage = lazy(() => import('./pages/WizardPage'));
const ProjectsPage = lazy(() => import('./pages/ProjectsPage'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<ChatPage />} />
        <Route path="/wizard" element={<WizardPage />} />
        <Route path="/projects" element={<ProjectsPage />} />
      </Routes>
    </Suspense>
  );
}
```

**Expected:** 3MB → 800KB initial bundle (73% smaller)

---

#### 6. Memoize useChat Callbacks (4 hours)

**File:** `tru-agent/frontend/chat-ui/src/hooks/useChat.ts`
**Impact:** Eliminates laggy typing in chat

```typescript
import { useCallback, useMemo } from 'react';

const handleSendMessage = useCallback(async (message: string) => {
  // Implementation
}, []); // Empty deps = stable reference
```

---

#### 7-8. Implement shadcn/ui Design System (40 hours)

**Location:** `tru-agent/frontend/chat-ui/`
**ROI:** ⭐⭐⭐⭐ (60% faster component development long-term)

```bash
cd tru-agent/frontend/chat-ui
npx shadcn@latest init
npx shadcn@latest add button input card dialog select textarea
```

**Refactor 12+ components** to use consistent shadcn/ui primitives.

---

#### 9. Fix WCAG 2.1 AA Accessibility (16 hours)

**Priority:** Legal/Compliance requirement for enterprise customers

**Violations to Fix:**
- 12 missing ARIA labels
- 8 color contrast failures
- 9 keyboard navigation issues
- 5 form errors not associated
- 3 focus indicators missing

---

## 📋 P2 - MEDIUM PRIORITY (Next Sprint)

**Total Effort:** 30 hours
**Impact:** Architecture improvements, maintainability

### 1. Implement Service Bindings (8 hours)

**Current:** HTTP calls between gateways (50-100ms overhead)
**Target:** Service bindings (<5ms overhead)

**Update all gateway wrangler.toml files:**
```toml
# tru-agent/wrangler.toml
[[services]]
binding = "RESEARCH_GATEWAY"
service = "research-gateway"

[[services]]
binding = "DATA_GATEWAY"
service = "data-gateway"

# ... etc for all 4 gateways
```

**Impact:** Saves 1.5s per multi-gateway workflow

---

### 2. Split Monolithic Orchestrator (16 hours)

**File:** `tru-agent/orchestration/orchestrator.js` (1,285 lines)
**Issue:** Violates Single Responsibility Principle

**Refactor into:**
```
tru-agent/orchestration/
├── orchestrator.js (300 lines) - Main class
├── modes/
│   ├── chat-mode.js (200 lines)
│   ├── plan-mode.js (250 lines)
│   ├── tool-mode.js (150 lines)
│   └── workflow-mode.js (300 lines)
└── utils/
    ├── response-builder.js (100 lines)
    ├── error-handler.js (100 lines)
    └── context-manager.js (185 lines)
```

---

### 3. Add D1 Batch API for Atomicity (2 hours)

**Note:** D1 doesn't support transactions yet. Use batch API as workaround.

```javascript
// Atomic multi-query operations
const batch = [
  env.AI_CONVERSATIONS.prepare('INSERT INTO conversations ...'),
  env.AI_CONVERSATIONS.prepare('INSERT INTO messages ...')
];

const results = await env.AI_CONVERSATIONS.batch(batch);
// All succeed or all fail - no partial writes
```

---

### 4. Add Optimistic Locking (4 hours)

**Files:** Campaign tables across all databases
**Issue:** Race conditions when multiple users update same campaign

```sql
-- Add version column
ALTER TABLE cdp_campaigns ADD COLUMN version INTEGER DEFAULT 1;

-- Update with version check
UPDATE cdp_campaigns
SET name = ?1, version = version + 1
WHERE id = ?2 AND version = ?3
```

---

## 📊 Expected Improvements (After P0 + P1)

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Workflow Latency** | 12s | 6s | 50% faster |
| **Initial Page Load** | 5s | 2s | 60% faster |
| **Bundle Size** | 3MB | 800KB | 73% smaller |
| **Throughput** | 250 req/s | 750 req/s | 3x increase |
| **Security Score** | 7/10 | 8.5/10 | Major improvement |
| **Code Quality** | 6.5/10 | 8/10 | Much better |

---

## 📅 Development Timeline

### Week 1 (P0 - Critical)
- Day 1: Fix test suite (2h)
- Day 2-3: Remove credential exposure ALL GATEWAYS (4h)
- Day 4-5: Implement rate limiting (4h)
- **Deploy to production Friday with `./deploy-all.sh`**

### Weeks 2-3 (P1 Backend)
- Week 2: Parallelize tools (1h) + Batch D1 (2h) + Retry (0.5h)
- Week 3: Replace JWT (12h)
- **Deploy orchestrator**

### Weeks 4-7 (P1 Frontend)
- Week 4: Code splitting (4h) + Memoization (4h)
- Week 5-6: shadcn/ui setup (8h) + Refactoring (32h)
- Week 7: Accessibility (16h)
- **Deploy frontend**

### Weeks 8-10 (P2)
- Week 8: Service bindings (8h) + Batch API (2h)
- Week 9-10: Split orchestrator (16h) + Locking (4h)
- **Deploy all improvements**

---

## 🚀 Quick Start for Developers

### Setup All Workers (30 minutes)

```bash
cd /home/matt/mcp-local-dev/truintent/workers

# Install dependencies for all workers
cd tru-agent && npm install && cd ..
cd research-gateway && npm install && cd ..
cd data-gateway && npm install && cd ..
cd crm-gateway && npm install && cd ..
cd cdp-gateway && npm install && cd ..
cd content-gateway && npm install && cd ..

# Set environment variables
export CLOUDFLARE_API_TOKEN="your-token-here"
```

### Deploy All Workers

```bash
# Deploy all 6 workers at once
./deploy-all.sh

# Or deploy individually
cd tru-agent && npx wrangler deploy && cd ..
cd research-gateway && npx wrangler deploy && cd ..
# ... etc
```

### Health Check All Workers

```bash
for gateway in tru-agent research-gateway data-gateway crm-gateway cdp-gateway content-gateway; do
  echo "=== $gateway ==="
  curl -s https://$gateway.datasnyperai.workers.dev/health | jq -r '.status'
done
```

### View Logs (Any Worker)

```bash
# Orchestrator logs
cd tru-agent
npx wrangler tail tru-agent --format pretty

# Gateway logs
cd crm-gateway
npx wrangler tail crm-gateway --format pretty
```

---

## 📝 Developer Onboarding

### For Backend Developers

**Pick Your First Task:**
- **Easy (30m):** Re-enable retry logic (P1.3) in `tru-agent/`
- **Medium (2h):** Batch D1 operations (P1.2) in `tru-agent/`
- **Hard (12h):** Replace JWT library (P1.4) in `tru-agent/`

**Read First:**
1. [README.md](README.md) - Complete architecture overview
2. [tru-agent/CLAUDE.md](tru-agent/CLAUDE.md) - Orchestrator details
3. [crm-gateway/CLAUDE.md](crm-gateway/CLAUDE.md) - Gateway pattern (if working on gateways)

---

### For Frontend Developers

**Pick Your First Task:**
- **Easy (4h):** Memoize useChat callbacks (P1.6)
- **Medium (4h):** Code splitting (P1.5)
- **Hard (40h):** Implement shadcn/ui (P1.7-8)

**Read First:**
1. [tru-agent/frontend/chat-ui/README.md](tru-agent/frontend/chat-ui/README.md)
2. [FRONTEND_OPTIMIZATION_PLAN.md](tru-agent/FRONTEND_OPTIMIZATION_PLAN.md) (if exists)

---

## 🎯 Success Metrics

Track these KPIs weekly:

**Performance:**
- Average workflow latency (target: <6s)
- P95 workflow latency (target: <10s)
- Initial page load time (target: <2s)

**Quality:**
- Test coverage (target: >85%)
- Production errors (target: <5/week)
- Code review turnaround (target: <24h)

**Security:**
- Security vulnerabilities (target: 0 critical, <3 high)
- Rate limit violations (target: <10/day)
- Failed auth attempts (monitor for attacks)

---

## 🚨 Deployment Checklist

Before deploying ANY worker:

**Pre-Deployment:**
- [ ] All tests pass for that worker
- [ ] Code reviewed by senior developer
- [ ] Security review (if touching auth/credentials)
- [ ] Performance benchmark (if optimization)

**Deployment:**
- [ ] Deploy to preview environment first (if available)
- [ ] Test critical workflows
- [ ] Deploy to production
- [ ] Monitor logs for 5 minutes

**Post-Deployment:**
- [ ] Verify `/health` endpoint
- [ ] Test critical user flows
- [ ] Check Cloudflare dashboard for errors
- [ ] Update project board

---

## 📞 Support & Escalation

**Documentation:**
- [README.md](README.md) - Architecture overview
- Worker-specific CLAUDE.md files in each directory
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) (if exists)

**Escalate if:**
- Security vulnerability discovered (P0)
- Production down (P0)
- Architectural decision needed (schedule review)

---

## 🎓 Key Architecture Decisions

### ✅ Decisions to Keep

1. **Specialist Gateway Pattern** - Solves LLM tool selection problem (40% gain)
2. **Cloudflare Actors** - 47% performance improvement (9.5s → 5s)
3. **Shared D1 Database** - Enables cross-gateway workflows
4. **Hono Framework** - Sub-50ms cold starts

### 🔄 Decisions to Revisit (P2)

1. **HTTP URLs for gateways** → Hybrid (service bindings + HTTP)
2. **Monolithic orchestrator** → Split into mode-specific classes
3. **Custom JWT** → Standard library (jose)
4. **Sequential tools** → Parallel execution

---

**Last Updated:** 2025-11-24
**Architecture:** Multi-Gateway Specialist Pattern (5 gateways + 1 orchestrator)
**Next Review:** End of P0 implementation (1 week)
**Questions?** Create an issue or ask in team chat

---

## 📚 Related Documentation

- **[README.md](README.md)** - Complete architecture overview
- **[tru-agent/CLAUDE.md](tru-agent/CLAUDE.md)** - Orchestrator implementation details
- **[crm-gateway/CLAUDE.md](crm-gateway/CLAUDE.md)** - Gateway pattern & actors
- **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** - Copy-paste commands
- **[shared/model-config.js](shared/model-config.js)** - Centralized model configuration
