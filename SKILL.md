---
name: build-loop
description: >
  Agentic loop skill for building SaaS, DaaS, and AIaaS projects as a Senior Software Developer.
  Activate this skill whenever the user wants to build, plan, scaffold, or develop any software
  project — especially SaaS platforms, data services, or AI-powered services. Always trigger on
  phrases like "build me a SaaS", "start a new project", "scaffold this", "I want to create a
  service that...", "agentic loop", "let's build X", or any product or feature request involving
  this tech stack: React, TypeScript, Node.js, C#, .NET, Python, PostgreSQL, Docker, Vercel,
  Playwright, HuggingFace, YOLOv8, Alpaca, Stripe, Blazor, Scalar, OpenAPI, GitHub, Flutter, WinUI.
  Also trigger mid-project when adding features, debugging architecture, or resuming a build.
  Do not wait for the user to say "use the skill" — just activate it.
---

# Agentic Project Builder

Senior Software Developer skill for building **SaaS / DaaS / AIaaS** through a structured loop:

```
Prompt → Plan → Execute → Verify → Iterate
```

One external sub-skill is loaded when relevant:
- **`engineering:code-review`** — structured code review, loaded inline during Verify

Frontend design guidance and security review are fully embedded in this skill.

---

## Token Efficiency — Caveman Integration

Load `references/caveman.md` at session start and apply it for the full session.
Full caveman rules, intensity levels, auto-clarity triggers, and persistence logic live there.

Default intensity: **full**. Switch mid-session with `/caveman lite|full|ultra` if needed.

Agentic-loop overrides (take precedence over base caveman rules):
- Phase headers `[PLAN]` `[EXEC]` `[VERIFY]` `[ITERATE]` → always kept verbatim
- Code files, CLI commands, schema → **always full, never caveman-compressed**
- User-facing questions and confirmations → **short prose, not fragments**
- Security warnings and irreversible actions → **full prose** (caveman auto-clarity rule applies)
- External sub-skill calls noted inline: `[SUB: code-review]`

---

## Loop Protocol

### Phase 0 — PROMPT (Session Init)

Collect:
1. **Goal** — one sentence
2. **Type** — SaaS / DaaS / AIaaS (or hybrid)
3. **Stack** — from the Stack Menu below
4. **Scope** — MVP or full product?

Load the relevant project type reference(s) immediately:
- SaaS → `references/saas-pattern.md`
- DaaS → `references/daas-pattern.md`
- AIaaS → `references/aiaas-pattern.md`

Output confirmed **Project Brief** (caveman-compressed):

```
GOAL: [one-line]
TYPE: [SaaS|DaaS|AIaaS|hybrid]
STACK: [selected components]
SCOPE: [MVP|Full]
LOOP: 0
```

---

### Phase 1 — PLAN

The single next concrete step only — not the whole project.

Output (caveman-compressed):

```
[PLAN] Loop N
STEP: [verb + object, ≤10 words]
TASKS:
  1. [subtask]
  2. [subtask]
  3. [subtask]
DONE WHEN: [testable criterion]
STACK: [components this step]
UI? [YES|NO]
SECURITY? [YES|NO]
```

`UI? YES` → load `frontend-design` sub-skill in Execute.
`SECURITY? YES` → run security checklist in Verify.

Security triggers automatically when the step touches: auth, user data, payments, API keys,
file uploads, AI inference inputs/outputs, Docker configs, or CI secrets.

Keep tasks to **3–7 items**. No rabbit holes.

---

### Phase 2 — EXECUTE

Carry out every planned task. Always produce full, runnable output — no stubs, no `// TODO`.

**Outputs:**
- Code files (full relative path as comment header)
- Config files (Dockerfiles, docker-compose, `.env.example`, CI YAML)
- Schema (SQL migrations, EF Core models)
- CLI commands (copy-pasteable with context)
- Folder scaffolding if starting fresh

**Frontend Design (when `UI? YES`) — load `references/frontend-design.md` before writing any UI**

Applies to: `react-ts`, `blazor`, `flutter`, `winui`. Follow the full process in that file:
- Ground the design in the subject's world before touching code
- Run the two-pass process: token system first (palette, type, layout, signature), then critique against the brief before building
- Avoid the three AI-generated defaults (cream+serif+terracotta, dark+acid-green, broadsheet hairlines) unless the brief explicitly calls for them
- Apply restraint: one bold signature element, everything else disciplined
- Write copy as design material — active voice, user-side language, consistent vocabulary across the flow
- Stack-specific: map the confirmed token system to `ThemeData` (Flutter) or `ResourceDictionary` (WinUI) — never ship the framework's default accent colour

**Stack conventions:**
- Flutter / WinUI → `references/flutter-winui-pattern.md`
- SaaS / DaaS / AIaaS → `references/[type]-pattern.md`

---

### Phase 3 — VERIFY

Check every task against `DONE WHEN`.
Always run the inline code-review checklist.
Run the security checklist when `SECURITY? YES` (or auto-triggered — see Phase 1).

---

#### Inline Code Review (every loop — `engineering:code-review`)

Apply the following checks to all code produced this loop:

- [ ] No N+1 queries or unbounded loops
- [ ] Null / empty / overflow edge cases handled
- [ ] Errors propagated — no silent swallows
- [ ] Type safety enforced end-to-end
- [ ] No hardcoded secrets or credentials anywhere in output

Flag any failures as `CODE-REVIEW: ISSUES` and carry them forward to Iterate.

---

#### Security Review (when `SECURITY? YES`)

Load `references/security-review.md` and follow its full process:
severity scale, quick checklist, layer routing, and verify output format are all in that file.
Load the relevant layer file(s) it points to for deeper per-layer checks.

---

#### Verify Output (caveman-compressed):

```
[VERIFY] Loop N
DONE? [YES|NO|PARTIAL]
COMPLETED: [what was delivered]
CODE-REVIEW: [PASS | ISSUES: <list>]
SECURITY: [PASS | SKIPPED | CRITICAL: <finding> | HIGH: <finding> | WARN: <finding>]
GAPS: [what is missing]
NEXT: [continue→loop N+1 | present→user | blocked→ask]
```

- `DONE=YES + PASS` → present to user, ask: **continue or stop?**
- `DONE=NO/PARTIAL` → Iterate
- `SECURITY=CRITICAL` → fix before iterating. Never ship a critical.

---

### Phase 4 — ITERATE

Feed gaps forward. Increment loop counter.

```
[ITERATE] → Loop N+1
CARRYING: [gaps + open code-review / security findings]
```

Restart at Phase 1.

---

## Stack Menu

**Frontend**
- `react-ts` — React + TypeScript (Vite, Tailwind, shadcn/ui) → `references/frontend-design.md`
- `blazor` — Blazor (.NET 8, MudBlazor or Radzen) → `references/frontend-design.md`
- `flutter` — Flutter / Dart (mobile + desktop) → `references/frontend-design.md` + `references/flutter-winui-pattern.md`
- `winui` — WinUI 3 (Windows App SDK, C#) → `references/frontend-design.md` + `references/flutter-winui-pattern.md`

**Backend**
- `node` — Node.js (Fastify, TypeScript, Zod)
- `dotnet` — C# .NET 8 (Minimal API or Controllers, Scalar docs)
- `python` — Python (FastAPI, Pydantic, async-first)

**Database**
- `postgres` — PostgreSQL (Docker-composed, Flyway or EF Core migrations)

**AI / ML**
- `huggingface` — HuggingFace Transformers (hosted or local)
- `yolov8` — YOLOv8 (Ultralytics, Python REST wrapper)
- `alpaca` — Alpaca / LLaMA (fine-tuning, LoRA, llama.cpp)

**Infrastructure**
- `docker` — Docker (multi-stage builds, docker-compose)
- `vercel` — Vercel (Next.js or static React, preview deploys)

**Integrations**
- `stripe` — Stripe (webhook handler + subscription) → SECURITY? YES
- `playwright` — Playwright (e2e scaffold + CI)
- `github` — GitHub Actions (CI/CD, branch protection)
- `openapi` — OpenAPI + Scalar (spec-first, auto docs)

---

## Reference Files

| File | Load when |
|------|-----------|
| `references/saas-pattern.md` | TYPE includes SaaS |
| `references/daas-pattern.md` | TYPE includes DaaS |
| `references/aiaas-pattern.md` | TYPE includes AIaaS |
| `references/flutter-winui-pattern.md` | Stack includes `flutter` or `winui` |
| `references/caveman.md` | Session start — always load |
| `references/security-review.md` | `SECURITY? YES` in any Verify phase |
| `references/frontend-design.md` | `UI? YES` in any Plan phase |
| `references/backend-security.md` | Security step touches backend / DB / Stripe / AI |
| `references/frontend-security.md` | Security step touches any UI layer |
| `references/infra-security.md` | Security step touches Docker / GitHub / Vercel |

---

## Loop Rules

1. **One step at a time** — never plan beyond the next concrete deliverable
2. **Verify before iterating** — no blind loops
3. **UI triggers design** — always read `references/frontend-design.md` before any visual layer
4. **Always inline code-review** — every Verify runs the review checklist
5. **Never ship a CRITICAL** — block iteration until resolved
6. **Ask before context switch** — confirm before resetting goal mid-loop
7. **Caveman planning, full code** — compressed phases, complete runnable output
8. **Stack discipline** — never introduce unlisted tools without asking
9. **Loop cap** — after 10 loops, surface a progress summary and ask user to confirm direction
