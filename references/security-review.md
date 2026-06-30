# Security Review

Run this during the Verify phase whenever `SECURITY? YES` is flagged in the Plan,
or automatically when a loop touches: auth, user data, payments, API keys, file uploads,
AI inference inputs/outputs, Docker configs, or CI secrets.

Pairs with `engineering:code-review` — code-review catches correctness; this catches exploits.

---

## How to Run

1. Check which layers were touched this loop (backend / frontend / infra / AI/ML)
2. Run the Quick Checklist below — applies to every security-active loop
3. Load the relevant layer reference file(s) for deeper checks
4. Output the result in the Verify block
5. Block iteration on any CRITICAL finding — fix first, then continue

---

## Severity Scale

| Level | Meaning | Action |
|-------|---------|--------|
| 🔴 CRITICAL | Exploitable now — auth bypass, injection, RCE, exposed secret | **Block. Fix before iterating.** |
| 🟠 HIGH | Likely exploited in production with moderate effort | Fix this loop |
| 🟡 MEDIUM | Exploitable under specific conditions | Fix before release |
| 🔵 LOW | Defence-in-depth, hardening | Fix when convenient |
| ✅ PASS | No issues found in this area | — |

---

## Quick Checklist (always run every security-active loop)

### Secrets & Config
- [ ] No secrets / API keys in source code or committed `.env`
- [ ] `.env.example` exists with dummy values; `.env` in `.gitignore`
- [ ] Docker Compose does not hardcode production credentials
- [ ] All secrets injected via environment variables

### Authentication
- [ ] Passwords hashed with bcrypt / Argon2 — never MD5, SHA1, or plain
- [ ] JWT secrets ≥256 bit, environment-injected, never hardcoded
- [ ] Refresh tokens rotated on use and invalidated on logout
- [ ] Auth endpoints rate-limited (10 req/min per IP minimum)

### Authorization
- [ ] Every API route checks caller role/permission server-side before executing
- [ ] Tenant isolation enforced server-side — never trust client-supplied tenant ID
- [ ] Horizontal privilege escalation tested — user A cannot reach user B's data

### Input Validation
- [ ] All user input validated and sanitised server-side (Zod / Pydantic / FluentValidation)
- [ ] File uploads validated by MIME type + magic bytes, not just extension
- [ ] Query parameters bounds-checked (page size limits, max string lengths)

### Error Handling
- [ ] Stack traces never returned to clients in production
- [ ] Error messages do not reveal internal paths, DB schema, or library versions
- [ ] 404 and 403 are indistinguishable for sensitive resources (no user enumeration)

---

## Layer Routing

Load only the files for layers touched this loop:

| Layer | Reference file |
|-------|---------------|
| Node.js / .NET / Python, PostgreSQL, Stripe, AI/ML | `references/backend-security.md` |
| React / Blazor / Flutter / WinUI | `references/frontend-security.md` |
| Docker / GitHub Actions / Vercel / CI-CD | `references/infra-security.md` |

---

## Verify Output (caveman-compressed)

```
[SECURITY] Loop N  layers: [backend|frontend|infra|ai]
CRITICAL: [finding] | none
HIGH:     [finding] | none
MEDIUM:   [finding] | none
VERDICT:  [PASS | BLOCK: <reason>]
```

CRITICAL = BLOCK. Never ship a critical. Fix inline and re-verify before iterating.
