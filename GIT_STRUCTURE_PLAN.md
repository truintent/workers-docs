# Git Structure Migration Plan

**Date:** November 20, 2025
**Purpose:** Align git repos with Worker architecture
**Current State:** 5 repos exist, 1 missing (cdp-gateway), lots of uncommitted changes

---

## Current Git Structure

### ✅ Existing Repos (5 of 6 Workers)

| Worker | Git Repo | Remote | Changed Files | Status |
|--------|----------|--------|---------------|--------|
| **tru-agent** | ✅ Yes | https://github.com/truintent/tru-agent.git | 164 | ⚠️ Many changes |
| **research-gateway** | ✅ Yes | https://github.com/truintent/research-gateway.git | 26 | ⚠️ Uncommitted |
| **data-gateway** | ✅ Yes | https://github.com/truintent/data-gateway.git | 10 | ⚠️ Uncommitted |
| **crm-gateway** | ✅ Yes | https://github.com/truintent/crm-gateway.git | 13 | ⚠️ Uncommitted |
| **content-gateway** | ✅ Yes | https://github.com/truintent/content-gateway.git | 12 | ⚠️ Uncommitted |
| **cdp-gateway** | ❌ No | None | N/A | 🚨 Missing repo |

### ❌ Not Repos (Should They Be?)

| Folder | Purpose | Git Needed? |
|--------|---------|-------------|
| `cf-agent-example` | Example code | ❌ No (example/docs) |
| `tru-agent-v4-sdk` | Experimental SDK | ⚠️ Maybe (separate repo?) |
| `shared/` | Shared utilities | ⚠️ Maybe (npm package?) |
| `services/` | Helper scripts | ❌ No (tooling) |
| `dev-ideas/` | Notes/brainstorming | ❌ No (docs only) |

### 📁 Workers Folder Structure

```
/home/matt/mcp-local-dev/truintent/workers/
├── tru-agent/              # ✅ Git repo (main branch, 164 changes)
├── research-gateway/       # ✅ Git repo (main branch, 26 changes)
├── data-gateway/           # ✅ Git repo (main branch, 10 changes)
├── crm-gateway/            # ✅ Git repo (main branch, 13 changes)
├── content-gateway/        # ✅ Git repo (main branch, 12 changes)
├── cdp-gateway/            # ❌ NO GIT REPO
├── tru-agent-v4-sdk/       # ⚠️ Experimental (has .git but not in org?)
├── cf-agent-example/       # ⚠️ Has .git (example repo)
├── shared/                 # ❌ No git
├── services/               # ❌ No git
├── dev-ideas/              # ❌ No git
├── deploy-all.sh           # ❌ Not tracked
├── *.md docs               # ❌ Not tracked
└── WORKER_ARCHITECTURE.md  # ❌ Not tracked (just created)
```

---

## Problem Analysis

### 1. **Missing cdp-gateway Repo**

**Issue:** cdp-gateway has no git repo but is a production Worker
**Impact:**
- Cannot version control queue consumer code
- Cannot track actor implementations
- Cannot rollback changes
- No CI/CD for cdp-gateway

**Evidence:**
```bash
$ cd cdp-gateway && git status
fatal: not a git repository
```

### 2. **Uncommitted Changes Everywhere**

**tru-agent:** 164 changed files
- Frontend changes (React components)
- Core orchestrator changes
- Deleted migrations
- Deleted agents (.agents/research-specialist.md)

**Other gateways:** 10-26 changes each
- Likely from recent actor deployments
- CLAUDE.md updates
- Tool implementations

### 3. **Root-Level Docs Not Tracked**

**Files not in any repo:**
- `WORKER_ARCHITECTURE.md` (just created)
- `DOCUMENTATION_CLEANUP_PLAN.md` (just created)
- `GIT_STRUCTURE_PLAN.md` (this file)
- `deploy-all.sh` (deployment script)
- `BUSINESS_INTELLIGENCE_PLATFORM_ARCHITECTURE.md`
- `CDP_GATEWAY_IMPLEMENTATION_GUIDE.md`
- `PLATFORM_ALIGNMENT_SUMMARY.md`

**Question:** Where should these live?

### 4. **Shared Code Duplication**

**Problem:** Each gateway likely has duplicate utility code
- No shared npm package
- No monorepo tooling (like Turborepo, Nx)
- Copy-paste risk

---

## Recommended Git Structure

### Option A: Separate Repos (Current + Fix Missing)

**Pros:**
- ✅ Already mostly implemented
- ✅ Independent CI/CD per worker
- ✅ Clear ownership boundaries
- ✅ Deploy one worker without affecting others

**Cons:**
- ❌ Root docs have no home
- ❌ Shared code duplication
- ❌ Hard to coordinate cross-repo changes
- ❌ 6 PRs instead of 1 for multi-worker features

**Structure:**
```
GitHub Organization: truintent/
├── tru-agent               (repo 1)
├── research-gateway        (repo 2)
├── data-gateway            (repo 3)
├── crm-gateway             (repo 4)
├── content-gateway         (repo 5)
├── cdp-gateway             (repo 6) ← CREATE THIS
└── workers-docs            (repo 7) ← NEW: Root docs
```

### Option B: Monorepo with Workspaces

**Pros:**
- ✅ All code in one repo
- ✅ Shared tooling (ESLint, Prettier, TypeScript configs)
- ✅ Easy atomic commits across workers
- ✅ Root docs naturally tracked
- ✅ Shared utilities via npm workspaces

**Cons:**
- ❌ Need to restructure existing repos
- ❌ Single large repo (slower git operations)
- ❌ Need CI/CD for multi-worker changes
- ❌ Merge conflicts more likely

**Structure:**
```
truintent-workers/ (monorepo)
├── workers/
│   ├── tru-agent/
│   ├── research-gateway/
│   ├── data-gateway/
│   ├── crm-gateway/
│   ├── content-gateway/
│   └── cdp-gateway/
├── shared/
│   ├── utils/
│   └── types/
├── docs/
│   ├── WORKER_ARCHITECTURE.md
│   └── ...
├── package.json              # Root workspace
└── pnpm-workspace.yaml       # or npm workspaces
```

### Option C: Hybrid (Separate Repos + Shared Packages)

**Pros:**
- ✅ Independent worker repos (current)
- ✅ Shared code via npm package (`@truintent/shared`)
- ✅ Best of both worlds

**Cons:**
- ❌ More complex setup
- ❌ Need to publish shared package
- ❌ Versioning coordination

**Structure:**
```
GitHub Organization: truintent/
├── tru-agent               (repo)
├── research-gateway        (repo)
├── data-gateway            (repo)
├── crm-gateway             (repo)
├── content-gateway         (repo)
├── cdp-gateway             (repo)
├── workers-shared          (npm package)
└── workers-docs            (repo for root docs)
```

---

## Recommended Approach: Option A+ (Separate Repos + Docs Repo)

**Why:**
- Minimal disruption (already 5 repos exist)
- Easy to add missing cdp-gateway repo
- Create new `workers-docs` repo for root-level docs
- Can evolve to Option C later (shared package)

**Action Items:**

### 1. Create cdp-gateway Repo

```bash
cd /home/matt/mcp-local-dev/truintent/workers/cdp-gateway

# Initialize git
git init
git branch -M main

# Create GitHub repo (via gh CLI or manually)
gh repo create truintent/cdp-gateway --public --source=. --remote=origin

# Add files
git add .
git commit -m "Initial commit: CDP Gateway with queue-based actors

- WorkflowOrchestratorActor
- ProfileScorerActor
- AudienceBuilderActor
- EnrollmentManagerActor
- ProfileEnricherActor

Queue consumer validated and operational."

# Push
git push -u origin main
```

### 2. Create workers-docs Repo

```bash
cd /home/matt/mcp-local-dev/truintent/workers

# Create new repo for root docs
mkdir -p workers-docs
cd workers-docs
git init
git branch -M main

# Move root-level docs
mv ../WORKER_ARCHITECTURE.md .
mv ../DOCUMENTATION_CLEANUP_PLAN.md .
mv ../GIT_STRUCTURE_PLAN.md .
mv ../BUSINESS_INTELLIGENCE_PLATFORM_ARCHITECTURE.md .
mv ../CDP_GATEWAY_IMPLEMENTATION_GUIDE.md .
mv ../PLATFORM_ALIGNMENT_SUMMARY.md .
mv ../CLOUDFLARE_STORAGE_INVENTORY.md .
mv ../README.md .
mv ../QUICK_REFERENCE.md .

# Create GitHub repo
gh repo create truintent/workers-docs --public --source=. --remote=origin

# Commit
git add .
git commit -m "Initial commit: TruIntent Workers documentation

Root-level documentation for 6-worker architecture:
- WORKER_ARCHITECTURE.md - Grandparent-Parent-Child hierarchy
- BUSINESS_INTELLIGENCE_PLATFORM_ARCHITECTURE.md - BI platform roadmap
- CDP_GATEWAY_IMPLEMENTATION_GUIDE.md - CDP guide
- PLATFORM_ALIGNMENT_SUMMARY.md - Strategy
- And more..."

# Push
git push -u origin main
```

### 3. Commit Pending Changes (All 5 Existing Repos)

**tru-agent (164 changes):**
```bash
cd /home/matt/mcp-local-dev/truintent/workers/tru-agent

# Review changes
git status
git diff

# Stage all changes
git add -A

# Commit
git commit -m "Major updates: Actor migration Phase 3 + Frontend improvements

Backend:
- Update CLAUDE.md with actor migration status
- Agent orchestrator improvements
- Intent classifier enhancements
- Tool executor updates

Frontend:
- Campaign wizard enhancements
- Chat UI improvements
- Project management updates
- TypeScript type improvements

Cleanup:
- Remove old migrations
- Remove deprecated agents
- Clean up static assets

Phase 3 Status: GHLContactActor deployed (1 of 7 CRM actors)"

# Push
git push origin main
```

**Other gateways (10-26 changes each):**
```bash
# research-gateway
cd /home/matt/mcp-local-dev/truintent/workers/research-gateway
git add -A
git commit -m "Phase 1 complete: 3 actors deployed + actor conversion docs"
git push origin main

# data-gateway
cd /home/matt/mcp-local-dev/truintent/workers/data-gateway
git add -A
git commit -m "Phase 2 complete: 6 actors deployed (validation + SQL)"
git push origin main

# crm-gateway
cd /home/matt/mcp-local-dev/truintent/workers/crm-gateway
git add -A
git commit -m "Phase 3 started: GHLContactActor deployed (1 of 7)"
git push origin main

# content-gateway
cd /home/matt/mcp-local-dev/truintent/workers/content-gateway
git add -A
git commit -m "Actors deployed: NodeSandbox + FirecrawlScraper"
git push origin main
```

### 4. Update Root README (workers-docs)

Create `/workers-docs/README.md`:

```markdown
# TruIntent Workers Documentation

**Architecture:** 6 independent Cloudflare Workers (Grandparent-Parent-Child)

## 6 Workers (Separate Repos)

1. **[tru-agent](https://github.com/truintent/tru-agent)** - Orchestrator (Grandparent)
2. **[research-gateway](https://github.com/truintent/research-gateway)** - Research specialist
3. **[data-gateway](https://github.com/truintent/data-gateway)** - Data specialist
4. **[crm-gateway](https://github.com/truintent/crm-gateway)** - CRM specialist
5. **[cdp-gateway](https://github.com/truintent/cdp-gateway)** - CDP specialist
6. **[content-gateway](https://github.com/truintent/content-gateway)** - Content specialist

## Documentation

- **[WORKER_ARCHITECTURE.md](WORKER_ARCHITECTURE.md)** - Complete architecture guide
- **[DOCUMENTATION_INDEX.md](DOCUMENTATION_INDEX.md)** - All docs index
- **[GIT_STRUCTURE_PLAN.md](GIT_STRUCTURE_PLAN.md)** - Git repo structure

## Quick Deploy

```bash
# Deploy all workers
cd /home/matt/mcp-local-dev/truintent/workers
./deploy-all.sh

# Deploy single worker
cd /home/matt/mcp-local-dev/truintent/workers/crm-gateway
npx wrangler deploy
```

## Local Development

Each worker has its own:
- `wrangler.toml` - Cloudflare config
- `package.json` - Dependencies
- `actors/` - Durable Objects (children)
- `tools/` - MCP tools
- `core/` - Entry point

See individual repo READMEs for per-worker setup.
```

---

## Migration Checklist

### Phase 1: Create Missing Repos
- [ ] Create cdp-gateway repo on GitHub
- [ ] Initialize git in `/workers/cdp-gateway`
- [ ] Push initial commit
- [ ] Create workers-docs repo on GitHub
- [ ] Move root docs to workers-docs
- [ ] Push initial commit

### Phase 2: Commit Pending Changes
- [ ] Commit tru-agent (164 files)
- [ ] Commit research-gateway (26 files)
- [ ] Commit data-gateway (10 files)
- [ ] Commit crm-gateway (13 files)
- [ ] Commit content-gateway (12 files)

### Phase 3: Update Documentation
- [ ] Update each worker's README to link to workers-docs
- [ ] Update CLAUDE.md in each worker
- [ ] Add git clone instructions to WORKER_ARCHITECTURE.md

### Phase 4: CI/CD (Future)
- [ ] Add GitHub Actions to each repo
- [ ] Auto-deploy on push to main
- [ ] Wrangler secrets in GitHub Secrets

---

## Alternative: Clean Slate Approach

If current repos are too messy, could:

1. **Archive old repos:**
   - Rename: `tru-agent` → `tru-agent-archive`
   - Keep for reference

2. **Create fresh repos:**
   - Start with clean commit history
   - Only include current code (no legacy)

3. **Benefits:**
   - Clean git history
   - No accumulated cruft
   - Fresh start with proper structure

4. **Cons:**
   - Lose git history
   - Need to migrate GitHub issues/PRs
   - More disruptive

**Recommendation:** Don't do clean slate unless absolutely necessary. Current repos are salvageable.

---

## Questions to Answer

1. **Should shared/ become an npm package?**
   - If yes: Create `@truintent/shared` package
   - If no: Keep duplicating utilities (simpler)

2. **Should tru-agent-v4-sdk stay separate?**
   - If experimental: Keep in separate repo
   - If active: Move to main workers-docs or archive

3. **What about cf-agent-example?**
   - Likely just example code → archive or keep separate

4. **Deploy script location?**
   - Keep in workers-docs repo
   - Or symlink from each worker?

---

## Summary

**Current State:**
- ✅ 5 repos exist (good foundation)
- ❌ 1 repo missing (cdp-gateway)
- ⚠️ 225+ uncommitted changes across all repos
- ❌ Root docs have no git home

**Recommended Actions:**
1. Create cdp-gateway repo ← **PRIORITY 1**
2. Create workers-docs repo for root docs ← **PRIORITY 2**
3. Commit all pending changes ← **PRIORITY 3**
4. Document git structure in workers-docs ← **PRIORITY 4**

**Timeline:**
- Phase 1 (Create repos): 30 minutes
- Phase 2 (Commit changes): 1-2 hours
- Phase 3 (Update docs): 30 minutes
- **Total:** 2-3 hours

**Risk:** Low (not deleting anything, just organizing)

---

**Document Version:** 1.0
**Date:** November 20, 2025
**Author:** Claude Code (AI Assistant)
**Status:** Ready for review
