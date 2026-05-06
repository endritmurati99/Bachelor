# Mobile Picking And Voice Assistant

Mobile Picking And Voice Assistant is a warehouse picking proof of concept that combines an Odoo 18 backend, a FastAPI application layer, a mobile PWA, local voice recognition, and n8n-based exception workflows.

## Current Status

The repository represents a bachelor-thesis PoC for assisted mobile picking. The current implementation includes the Wave A quality-alert AI evaluation path, controlled n8n workflow rollout scripts, a PWA for scanner/voice/touch flows, FastAPI routes, Odoo customizations, and Playwright/backend tests.

The project keeps Odoo as the system of record. FastAPI is the only app-facing API for the PWA, n8n handles asynchronous or exception workflows, and touch input remains the operational fallback when voice is unavailable.

## Key Capabilities

- Mobile picking list and picking-line confirmation through the PWA.
- Soft claiming, heartbeats, idempotent mutation requests, and barcode/touch/voice confirmation paths.
- Local voice hot path through `POST /api/voice/recognize`.
- Synchronous exception assistance through `POST /api/voice/assist`.
- Quality alert creation with controlled n8n handoff and structured Odoo writeback.
- Integration logging and telemetry export for staged rollout windows.

## Architecture

```text
PWA -> Caddy -> FastAPI -> Odoo
                |-> local ASR service
                `-> n8n workflows

n8n -> internal FastAPI callbacks -> Odoo
```

Core rules:

- Odoo remains the system of record.
- FastAPI owns validation, idempotency, Odoo adaptation, and PWA-facing contracts.
- n8n does not sit in the normal voice hot path.
- n8n writes back only through internal FastAPI callbacks.
- Voice is an enhancement; touch remains the fallback.

Useful docs:

- [Architecture](Mobile%20Picking%20und%20Voice%20Assistant/docs/ARCHITECTURE.md)
- [Setup](Mobile%20Picking%20und%20Voice%20Assistant/docs/SETUP.md)
- [Voice Commands](Mobile%20Picking%20und%20Voice%20Assistant/docs/VOICE_COMMANDS.md)
- [n8n Contract Freeze](Mobile%20Picking%20und%20Voice%20Assistant/docs/N8N_CONTRACT_FREEZE_V1.md)
- [Quality Alert AI Fields](Mobile%20Picking%20und%20Voice%20Assistant/docs/QUALITY_ALERT_AI_FIELDS.md)

## Quick Start

```bash
cd "Mobile Picking und Voice Assistant"
docker compose build
docker compose up -d
```

Typical local endpoints after setup:

- PWA: `https://<LAN-IP>/`
- API docs: `https://<LAN-IP>/api/docs`
- Odoo admin: `http://<HOST>:8069/`
- n8n: `https://<LAN-IP>/n8n/`

The full setup path, certificate handling, Odoo initialization, and workflow rollout steps are documented in [`docs/SETUP.md`](Mobile%20Picking%20und%20Voice%20Assistant/docs/SETUP.md).

## Verification

Recommended checks:

```bash
cd "Mobile Picking und Voice Assistant"
python infrastructure/scripts/verify-workflows.py
pytest backend/tests -q
node --test n8n/tests/*.test.mjs
npm run test:voice
npm run test:ui
git diff --check
```

Run live n8n workflow activation only after creating a backup with `infrastructure/scripts/import-workflows.sh`.

## Privacy And Safety

- Store credentials only in local environment files; never commit real Odoo, n8n, or webhook secrets.
- Use a dedicated Odoo service user for backend access.
- Keep operator-visible Odoo records truthful even when n8n handoff fails.
- Quality-alert AI output is written through controlled fields and chatter text, not uncontrolled HTML fragments.
- Do not treat local voice recognition as the only operational path; scanner and touch flows must continue to work.

## Roadmap

- Complete staged live rollout for Quality, Voice, and Replenishment paths.
- Validate success and failure paths against the live Odoo and n8n runtime.
- Export telemetry for rollout windows and compare it against expected behavior.
- Continue hardening mobile UI, accessibility, and operator fallback flows.
