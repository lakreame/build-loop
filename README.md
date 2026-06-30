# build-loop

> Agentic loop skill for building SaaS, DaaS, and AIaaS projects — as a Senior Software Developer.

A [Claude Skill](https://docs.claude.ai/skills) that drives a structured **Prompt → Plan → Execute → Verify → Iterate** loop across your full stack. Token-efficient by default via built-in Caveman mode. Security review, frontend design, and code review are embedded — no external dependencies.

---

## What It Does

```
Prompt → Plan → Execute → Verify → Iterate
```

| Phase | What happens |
|-------|-------------|
| **Prompt** | You state the goal. Skill collects type, stack, and scope. |
| **Plan** | One concrete next step only — tasks, done-criteria, stack flags. |
| **Execute** | Full runnable code, configs, schema, CLI commands. No stubs. |
| **Verify** | Inline code review + security checklist. Blocks on CRITICAL findings. |
| **Iterate** | Gaps feed back into the next loop. Repeats until done. |

---

## Tech Stack Supported

**Frontend** — React + TypeScript, Blazor, Flutter, WinUI 3  
**Backend** — Node.js (Fastify), C# .NET 8, Python (FastAPI)  
**Database** — PostgreSQL  
**AI / ML** — HuggingFace, YOLOv8, Alpaca / LLaMA  
**Infrastructure** — Docker, Vercel, GitHub Actions  
**Integrations** — Stripe, Playwright, OpenAPI + Scalar  

---

## Built-in Skills

All embedded — no external skill dependencies:

| Skill | When it activates |
|-------|-------------------|
| **Caveman mode** | Session start — compresses planning phases ~75% token reduction |
| **Frontend design** | Any UI step — full design-lead process, token system, anti-template critique |
| **Security review** | Auth, payments, data, AI layers — severity scale, checklist, layer routing |
| **Code review** | Every Verify phase — N+1, nulls, type safety, error propagation |

---

## File Structure

```
build-loop/
├── SKILL.md                        ← Main skill — loop protocol + stack menu
└── references/
    ├── caveman.md                  ← Token compression rules
    ├── frontend-design.md          ← Design-lead process
    ├── security-review.md          ← Severity scale, checklist, layer routing
    ├── saas-pattern.md             ← Auth, multi-tenancy, RBAC, Stripe billing
    ├── daas-pattern.md             ← Ingestion, transforms, OpenAPI, rate limiting
    ├── aiaas-pattern.md            ← Model serving, async inference, observability
    ├── flutter-winui-pattern.md    ← Flutter BLoC/GoRouter + WinUI MVVM, testing, CI
    ├── backend-security.md         ← Injection, JWT, bcrypt, Stripe webhooks, AI/ML
    ├── frontend-security.md        ← XSS, CSP, Flutter secure storage, WinUI DPAPI
    └── infra-security.md           ← Docker hardening, GitHub Actions, Vercel headers
```

---

## Installation

### Claude.ai
1. Download `build-loop.skill`
2. Go to **Claude.ai → Settings → Skills**
3. Click **Install skill** and upload the file

### Claude Code
```bash
claude skill install build-loop.skill
```

---

## Usage

Just describe what you want to build:

```
Build me a SaaS for team task management with Stripe billing
```

```
Start a new AIaaS project for YOLOv8 object detection with a React dashboard
```

```
Scaffold a DaaS API for ingesting CSV uploads and serving them via OpenAPI
```

The skill activates automatically on project/build requests. No trigger phrase needed.

---

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Good first contributions:**
- New stack patterns (more AI/ML frameworks, new cloud providers)
- Additional security reference files (mobile, GraphQL, WebSockets)
- More project type references (marketplace, multi-sided platform, analytics)
- Improved test coverage and eval sets for the skill

---

## License

MIT — see [LICENSE](LICENSE)
