# Campaign Gateway Setup Notes

**Date:** 2025-11-20 21:45 UTC  
**Owner:** GPT-5 Codex (handoff for future DevOps work)

## Summary
- Scaffolded new `campaign-gateway/` by cloning `data-gateway`, removing `node_modules`/`package-lock.json`, and aligning `package.json`/`wrangler.toml` with the delegation checklist (four queues, three DO bindings, shared D1/KV/R2 vars).
- Added placeholder actors (`CampaignImportActor`, `CampaignStatusActor`, `CampaignOrchestratorActor`) plus queue-aware `core/index.js` with `/health`, `/import`, `/status/:campaignId`, and stub queue consumers.
- Pruned template-specific directories (`actors/data`, `tools/data`) and created `actors/campaigns/`, `tools/campaigns/` for future implementation.
- `npm install` failed due to sandboxed network (`EAI_AGAIN` hitting registry.npmjs.org); rerun when outbound access is available. No other blockers.

## Outstanding To-Dos
1. Re-run `npm install` inside `campaign-gateway` (requires registry access).  
   - Command: `cd workers/campaign-gateway && npm install`
2. Configure secrets + deploy once dependencies install:  
   - `npx wrangler secret put OPENAI_API_KEY --name campaign-gateway`  
   - `npx wrangler deploy`  
   - `curl https://campaign-gateway.datasnyperai.workers.dev/health`
3. Future day-by-day tasks (per delegation doc): implement actor methods, queue consumers, and tool wrappers.

## References
- Task instructions: `workers-docs/DELEGATION_TASK_CAMPAIGN_GATEWAY_SETUP.md`  
- Fresh scaffold path: `workers/campaign-gateway/`
