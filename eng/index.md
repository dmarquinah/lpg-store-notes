# Engineering — lpg-store

---
project: lpg-store
domain: eng
last-updated: 2026-05-08
---

## Documents

| File | Updated | Summary |
|------|---------|---------|
| [[architecture]] | 2026-05-08 | Backend v2 architecture + frontend v2 architecture (current). v1 backend reference under `lpg-backend/legacy/`; v1 frontend under `lpg-frontend-vue/legacy/`. |
| [[decisions]] | 2026-05-07 | ADR log — append-only |
| [[legacy-bloat-analysis]] | 2026-05-07 | Diagnosis of why v1 backend grew to ~35k LOC for ~15 endpoints; the ten patterns v2 must avoid |
| [[frontend-bloat-analysis]] | 2026-05-08 | Diagnosis of why v1 frontend grew to ~33k LOC for ~25 screens; the ten drivers (role-as-folder, i18n, FCM, etc.) |
| [[patterns/module-template]] | 2026-05-07 | Vertical module folder convention for `lpg-backend/src/modules/<feature>/` |
| [[patterns/frontend-module-template]] | 2026-05-08 | Vertical module folder convention for `lpg-frontend-vue/src/modules/<feature>/` |

## Cross-References

- [[../../_shared/conventions]]
- [[../../_shared/tooling]]
