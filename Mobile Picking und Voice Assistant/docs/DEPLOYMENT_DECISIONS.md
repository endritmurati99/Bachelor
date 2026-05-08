# Deployment Decisions and Iteration Notes

Status: working decision log after multiple review passes.  
Scope: Hostinger VPS, Docker stack, n8n rollout, Quality Vision, Atlas/NotebookLM coordination.

## 1. Current verification state

Last local checks from the OpenClaw runtime:

- GitHub is the canonical transfer path.
- `docs/VPS_RUNBOOK.md` is now the canonical VPS runbook.
- `docs/QUALITY_VISION_WORKFLOW_PLAN.md` stays separate because it is product/workflow design, not VPS operations.
- `python3 infrastructure/scripts/verify-workflows.py` passes.
- Existing warnings are known contract/noise warnings, not new blockers.
- This OpenClaw runtime has no Docker binary; Docker checks must run on the VPS or another host with Docker access.

## 2. What I would change now

### 2.1 Make Whisper optional before first VPS deployment

Reason: Whisper `small` is one of the heaviest services in the stack. On a 4 GB VPS it can push the system into swap pressure or out-of-memory failures.

Decision:

- If VPS has less than 8 GB RAM: start without local Whisper.
- Keep voice/ASR integration structurally ready, but do not make it block first deployment.
- Add Whisper back after base stack stability is proven.

Implementation direction:

- Add a Compose profile or override for `whisper`.
- Backend must fail gracefully if `WHISPER_URL` is unavailable.

### 2.2 Start n8n Quality Vision in shadow mode only

Reason: current Quality Alert AI path is text-first. The new visual assessment must not make operational decisions until the output is measured.

Decision:

- Build `quality-alert-vision-assessment` as shadow workflow first.
- Write assessment output to separate fields/logs.
- Do not let it mark goods as sellable operationally.

Acceptance gate:

- No known defective/unclear test image may be marked confidently sellable.
- Bad/irrelevant images must go to manual review.

### 2.3 Keep n8n import controlled and reversible

Reason: workflow imports can overwrite or activate wrong flows.

Decision:

- Always export backup first.
- Import inactive.
- Activate selected workflows only.
- Keep rollback command ready.

### 2.4 Reduce production exposure

Reason: Odoo/n8n/Postgres/Mailpit should not be public.

Decision:

- Public: only 80/443, and SSH if needed.
- Private/internal: Odoo, n8n, Postgres, Mailpit.
- n8n Public API stays disabled unless explicitly needed for a controlled operation.

### 2.5 Use GitHub, not ZIP transfer

Reason: ZIP transfer is too easy to pollute with secrets, caches, and local runtime data.

Decision:

- Use `git clone` or `git pull` on VPS.
- Move only `.env` and persistent data separately.

## 3. What I would not change now

### 3.1 Do not train a custom defect model yet

Reason: not enough real defect images, too much labeling effort, and high thesis risk.

Keep:

- Vision API/LLM as an assisted interpretation layer.
- Strict JSON schema.
- Guardrails and human review.
- LEGO/synthetic fixtures for evaluation, not as proof of industrial-grade model quality.

### 3.2 Do not merge Atlas into the mobile-picking deployment

Reason: Atlas/NotebookLM is a separate learning pipeline. Mixing it into the VPS deployment would increase failure modes and distract from the thesis demo.

Keep separate:

- Photon: build/debug/deployment/n8n/mobile-picking.
- Phanes/Atlas: source discovery, NotebookLM, learning workflow.

Bridge later only through explicit APIs/files once both sides are stable.

### 3.3 Do not activate old P1 Telegram/Gmail workflows by default

Reason: they are not central to the mobile-picking thesis path and add credentials/integration noise.

Keep them in repo if useful, but do not activate on production VPS unless explicitly needed.

### 3.4 Do not expose n8n MCP/API secrets in repo

Reason: n8n API/MCP credentials are personal operational secrets.

Keep:

- local environment variables
- local Claude MCP config if needed
- no shared `.mcp.json` for n8n tokens

## 4. Open decision gates

Before deploying the full stack, we need:

1. VPS RAM.
2. VPS vCPU count.
3. Free disk space.
4. Docker/Compose availability.
5. Current exposed ports.
6. Domain strategy:
   - direct domain to Caddy, or
   - Cloudflare Tunnel.
7. Whether local Whisper is mandatory for the first demo.
8. Whether NotebookLM is authenticated at the computer.

## 5. Recommended next order

### Phase 1: VPS baseline

1. Run read-only VPS check.
2. Decide full or reduced stack.
3. Install Docker only if missing and approved.
4. Add swap if RAM is small.
5. Clone repo from GitHub.
6. Prepare `.env`.
7. Render Compose and inspect exposure.

### Phase 2: Base stack

1. Start DB, backend, Odoo, Caddy, n8n.
2. Delay Whisper if RAM is tight.
3. Verify health endpoints and logs.
4. Confirm n8n is reachable internally.

### Phase 3: n8n workflow control

1. Export n8n backup.
2. Import workflows inactive.
3. Activate only required production workflows.
4. Test callback path.

### Phase 4: Quality Vision

1. Add local validation module and tests.
2. Add shadow workflow JSON.
3. Add LEGO fixture labels.
4. Run offline evaluator.
5. Only then consider operational writeback.

### Phase 5: Atlas / NotebookLM

1. Inspect existing Atlas project state.
2. Stabilize local discovery + approval queue.
3. Confirm NotebookLM login at computer.
4. Build connector behind replaceable interface.
5. Verify Phanes boundaries.

## 6. Current recommendation

Do not start by moving files manually. Start by measuring the VPS.

If the VPS has 8 GB RAM or more: deploy the base stack normally, but still keep Quality Vision shadow-only.

If the VPS has 4 GB RAM: deploy reduced stack first, without local Whisper, and avoid heavy builds/tests on the VPS.

If the VPS has 2 GB RAM: do not run the full stack there. Use it for a reduced n8n/backend demo only or upgrade the VPS.
