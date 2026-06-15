# Users — lpg-store

---
project: lpg-store
domain: specs
category: users
last-updated: 2026-06-04
---

## Context Documents

Read before working on users specs:

- [[../../eng/architecture]] — backend v2 architecture (modules, repository pattern, error handling)
- [[../../eng/patterns/module-template]] — vertical module folder convention
- [[../auth/auth-foundation/index]] — auth owns the `users` table, `user_role` enum, user creation (register+invite), and self password change; the users module manages existing users only

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[users-crud/index\|users-crud]] | done · backend ✓ frontend ✓ | Barebones management of existing users: list, view one, partial update of name/phone/role/active. Role assignment is the primary driver. No creation (auth owns it), no password handling. |
