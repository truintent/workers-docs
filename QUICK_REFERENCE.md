# TruIntent Workers - Quick Reference Card

**Location:** `/home/matt/mcp-local-dev/truintent/workers/`
**Updated:** 2025-11-13

---

## 🚀 Deploy All Workers
```bash
cd /home/matt/mcp-local-dev/truintent/workers
export CLOUDFLARE_API_TOKEN="T05noK_9901_jceSr_V8GGCizySVSyoZmfa8krT3"
./deploy-all.sh
```

---

## 🎯 Deploy Single Worker
```bash
./deploy-all.sh tru-agent
./deploy-all.sh research-gateway
./deploy-all.sh data-gateway
./deploy-all.sh crm-gateway
./deploy-all.sh content-gateway
```

---

## 🔍 View Logs
```bash
CLOUDFLARE_API_TOKEN="T05noK_9901_jceSr_V8GGCizySVSyoZmfa8krT3" \
  npx wrangler tail tru-agent --format pretty
```

---

## ✅ Health Checks
```bash
curl https://mcp.truintent.io/health
curl https://research-gateway.datasnyperai.workers.dev/health
curl https://data-gateway.datasnyperai.workers.dev/health
curl https://crm-gateway.datasnyperai.workers.dev/health
curl https://content-gateway.datasnyperai.workers.dev/health
```

---

## 📋 Worker URLs

| Worker | URL | Tools |
|--------|-----|-------|
| tru-agent | https://mcp.truintent.io | 52 |
| research-gateway | https://research-gateway.datasnyperai.workers.dev | 12 |
| data-gateway | https://data-gateway.datasnyperai.workers.dev | 8 |
| crm-gateway | https://crm-gateway.datasnyperai.workers.dev | 10 |
| content-gateway | https://content-gateway.datasnyperai.workers.dev | 15 |

---

## 📂 File Structure
```
workers/
├── tru-agent/              # Orchestrator
├── research-gateway/       # Research specialist
├── data-gateway/           # Data specialist
├── crm-gateway/            # CRM specialist
├── content-gateway/        # Content specialist
├── shared/                 # Shared utilities (future)
├── deploy-all.sh           # Deployment script
├── README.md               # Full documentation
├── REORGANIZATION_SUMMARY.md
└── QUICK_REFERENCE.md      # This file
```

---

## 🔧 Common Tasks

### Navigate to Workers
```bash
cd /home/matt/mcp-local-dev/truintent/workers
```

### List Workers
```bash
ls -la
```

### Check Versions
```bash
for worker in tru-agent research-gateway data-gateway crm-gateway content-gateway; do
  echo "=== $worker ==="
  cd $worker && grep version package.json && cd ..
done
```

### View Full Docs
```bash
cat README.md
```

---

## 🐛 Troubleshooting

### Worker Not Responding
```bash
# Check logs
CLOUDFLARE_API_TOKEN="token" npx wrangler tail worker-name --format pretty

# Check health
curl https://worker-url/health
```

### Deployment Fails
```bash
# Clear cache
rm -rf node_modules .wrangler
npm install

# Deploy again
CLOUDFLARE_API_TOKEN="token" npx wrangler deploy
```

---

## 📚 Documentation Files

- **README.md** - Complete documentation
- **REORGANIZATION_SUMMARY.md** - File structure changes
- **QUICK_REFERENCE.md** - This card
- **shared/README.md** - Shared utilities docs

---

## ⚡ Quick Copy-Paste Commands

```bash
# Export token (do this once per terminal session)
export CLOUDFLARE_API_TOKEN="T05noK_9901_jceSr_V8GGCizySVSyoZmfa8krT3"

# Navigate to workers
cd /home/matt/mcp-local-dev/truintent/workers

# Deploy all
./deploy-all.sh

# View logs (tru-agent)
npx wrangler tail tru-agent --format pretty

# Health check all
for url in https://mcp.truintent.io/health \
           https://research-gateway.datasnyperai.workers.dev/health \
           https://data-gateway.datasnyperai.workers.dev/health \
           https://crm-gateway.datasnyperai.workers.dev/health \
           https://content-gateway.datasnyperai.workers.dev/health; do
  echo "=== $url ==="
  curl -s $url | jq
done
```

---

**Cloudflare Account:** `07c8e6d07bb6a43fc8829184d552baf1`
**Support:** Check logs first, then review README.md
