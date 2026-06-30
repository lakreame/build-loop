# Contributing to build-loop

Thank you for contributing! This skill improves through real-world use, so your additions matter.

---

## What We're Looking For

### High-value contributions
- **New stack references** — patterns for frameworks not yet covered (Remix, SvelteKit, FastHTML, .NET MAUI, Tauri, etc.)
- **Security reference files** — GraphQL security, WebSocket security, mobile-specific (React Native), gRPC
- **New project type patterns** — marketplace, analytics platform, multi-sided platform, IoT backend
- **AI/ML expansions** — model fine-tuning workflows, vector DB patterns (pgvector, Qdrant), RAG pipelines
- **Bug fixes** — incorrect code samples, outdated package versions, broken patterns
- **Evals** — test prompts and assertion sets that verify the skill works correctly

### Not a fit right now
- Completely different tech stacks not related to the existing ecosystem
- Changes to the core loop protocol without a thorough explanation
- Removing existing stack support without deprecation discussion

---

## How to Contribute

### 1. Fork and clone
```bash
git clone https://github.com/YOUR_USERNAME/build-loop
cd build-loop
```

### 2. Make your changes

**Adding a new reference file:**
- Place it in `references/`
- Name it clearly: `[domain]-pattern.md` for stack patterns, `[layer]-security.md` for security
- Follow the existing file format — sections with `##`, code blocks with language tags, full runnable examples
- Update the reference table in `SKILL.md` so Claude knows when to load it

**Editing SKILL.md:**
- Keep it under 300 lines — detailed content belongs in `references/`
- The loop phases (Plan/Execute/Verify/Iterate) should stay stable; add flags rather than new phases
- Run a sanity check: does the skill still activate correctly after your change?

**Updating an existing reference file:**
- Keep code examples runnable and up to date with current package versions
- If a pattern changed significantly (e.g. new major version of a framework), note the version explicitly

### 3. Test your change

Try the skill with at least two prompts that exercise your new content:
```
Build me a SaaS using [your new stack]
```
Verify the loop runs correctly and the reference file loads at the right time.

### 4. Open a pull request

Use the PR template. Include:
- What you added or changed
- Why it belongs in this skill
- Any prompts you tested with

---

## Code Style for Reference Files

```markdown
# File Title

Brief one-line description of what this covers.

---

## Section Name

Explanation in prose (keep it short — Claude reads this at runtime).

### Subsection

Always include **full, runnable** code examples. No pseudocode. No `// TODO`.

\```language
// Full path comment at top of every file example
// path/to/file.ts
actual runnable code here
\```

Keep examples current with the latest stable versions of all packages.
```

---

## Reporting Issues

Use GitHub Issues. Include:
- The prompt you used
- What the skill did
- What you expected it to do
- Which reference file seems to be the cause (if known)

---

## License

By contributing, you agree your work is licensed under the [MIT License](LICENSE).
