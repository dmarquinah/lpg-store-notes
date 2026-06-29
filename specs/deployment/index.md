# Deployment — lpg-store

---
project: lpg-store
domain: specs
category: deployment
last-updated: 2026-06-18
---

## Context Documents

Read these vault docs before working on deployment specs:

- [[../../ops/deployment]] — current backend deploy pipeline (CI → GHCR → SSH → VPS compose) — the index for the in-repo runbook
- [[../../ops/development]] — local dev stack (host backend + docker compose deps)
- [[../../eng/architecture]] — backend + frontend layout, how the pieces fit
- [[../../index]] — Repositories table (the three repos and their stacks/roles)
- `lpg-backend/docs/DEPLOYMENT.md` — the existing full backend runbook (the thing these specs extend)
- `lpg-backend/docker-compose.yml` — the single dev+prod compose file
- `lpg-backend/legacy/docs/CI-CD-Pipeline.md`, `lpg-backend/legacy/docs/deploy/` — v1 deploy notes (reference only; v1 targeted Supabase/managed, not a self-hosted VPS)

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[first-vps-deploy/index\|first-vps-deploy]] | done | First-time production deploy to a self-hosted VPS: a from-scratch provisioning runbook (OS dev-user, firewall, Docker, DNS, secrets) **and** a Traefik edge reverse proxy with automatic TLS fronting two containerized services — the backend Node app and the frontend (Piloto) served by its own nginx. |

## Notes

This category covers **self-hosted VPS deployment** of the whole product (backend + frontend, eventually bot). The backend already has a working CI→GHCR→VPS pipeline (`lpg-backend/docs/DEPLOYMENT.md`); the open work is (a) a complete first-time provisioning guide for a fresh box including the owner's own OS user, and (b) putting both apps behind one TLS-terminating edge proxy so they're reachable on real domains over HTTPS.
