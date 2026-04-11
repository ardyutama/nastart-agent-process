# nastart-agent-process

> AI agent methodology and planning context behind the [nastart](https://github.com/ardyutama/nastart) recipe costing app.

This repo documents how GitHub Copilot agents were used systematically — from initial idea to architecture decisions to working code — to build a production-quality full-stack application. It is the process layer; [nastart](https://github.com/ardyutama/nastart) is the result.

---

## What's Inside

| Folder | Contents |
|---|---|
| [`lessons/`](lessons/) | 8 lessons covering Clean Architecture, EF Core schema, CQRS with MediatR, validation, JWT auth, ingredient management, cost cascade, and recipe costing engine |
| [`business-flows/`](business-flows/) | Canonical architecture decisions (locked), auth flows, ingredient price management, recipe costing, invoice scanning, Telegram bot |
| [`personas/`](personas/) | Target user research — home bakers, F&B owners, head chefs, procurement, cost controllers |
| [`flows/`](flows/) | Feature-level user flows: auth, recipe builder, ingredient management, invoice OCR, Telegram |
| [`docs/plans/`](docs/plans/) | Design documents produced during brainstorming sessions |
| [`.github/skills/`](.github/skills/) | 20+ agent skill definitions — reusable instruction sets for brainstorming, TDD, planning, code review, security, and more |
| [`.github/context/`](.github/context/) | Phase session context, v1 constraints, project map — loaded by agents at session start |
| [`AGENTS.md`](AGENTS.md) | Agent rules file — loaded at the start of every coding session |
| [`CHECKPOINT.md`](CHECKPOINT.md) | Full conversation state export — used to resume sessions with complete context |

---

## How the Agent Workflow Operates

Each coding session follows a structured loop:

1. **Load context** — agent reads `AGENTS.md` + relevant lesson/context files
2. **Brainstorm** — `brainstorming` skill explores requirements before any code is written
3. **Plan** — `planning-and-task-breakdown` skill breaks work into ordered, verifiable increments
4. **Implement** — `incremental-implementation` skill; TDD enforced via `test-driven-development` skill
5. **Review** — `code-review-and-quality` skill checks correctness, readability, architecture, security, performance
6. **Document** — `documentation-and-adrs` skill records decisions in `docs/plans/`

Skills are composable — a session may invoke several in sequence. The `using-agent-skills` skill governs how skills are discovered and chained.

---

## Agent Skills

| Skill | Purpose |
|---|---|
| `brainstorming` | Turn ideas into designs before any code is written |
| `spec-driven-development` | Create specs when requirements are ambiguous |
| `planning-and-task-breakdown` | Break features into ordered, implementable tasks |
| `incremental-implementation` | Deliver changes in small, verifiable increments |
| `test-driven-development` | Write failing tests first; prove bugs before fixing |
| `code-review-and-quality` | Multi-axis review: correctness, readability, architecture, security, performance |
| `api-and-interface-design` | Design stable API and module boundaries |
| `security-and-hardening` | Harden code against OWASP Top 10 |
| `debugging-and-error-recovery` | Systematic root-cause debugging |
| `documentation-and-adrs` | Record decisions with context, constraints, and trade-offs |
| `frontend-ui-engineering` | Build production-quality Vue components |
| `ci-cd-and-automation` | Set up build, test, and deployment pipelines |
| `context-engineering` | Optimize agent context for quality output |
| + 8 more | `idea-refine`, `code-simplification`, `performance-optimization`, `deprecation-and-migration`, `shipping-and-launch`, `git-workflow-and-versioning`, `browser-testing-with-devtools`, `using-agent-skills` |

---

## Tech Stack (product)

| Layer | Technology |
|---|---|
| Backend API | .NET 10 Web API — Clean Architecture |
| AI / OCR / Bot | Python FastAPI |
| Frontend | Vue 3 + Vite + TypeScript |
| Database | PostgreSQL 16 + EF Core |
| Auth | JWT |
| File Storage | MiniStack (local) → Cloudflare R2 (VPS) → AWS S3 (future) |
| Deployment | Docker Compose → VPS |

---

## Storage Strategy

File storage uses S3-compatible APIs across all environments — **zero code changes** when moving between stages:

```
MiniStack (local dev) → Cloudflare R2 (VPS production) → AWS S3 (if scale demands)
       ↑                          ↑                               ↑
   same code                  same code                       same code
```

The boto3 client is identical in every environment — only the endpoint URL and credentials change via environment variables:

```python
# ai-service/config.py — one client definition, works everywhere
s3 = boto3.client("s3",
    endpoint_url=os.getenv("AWS_ENDPOINT_URL"),   # None = real AWS, string = emulator/R2
    aws_access_key_id=os.getenv("AWS_ACCESS_KEY_ID"),
    aws_secret_access_key=os.getenv("AWS_SECRET_ACCESS_KEY"),
    region_name=os.getenv("AWS_DEFAULT_REGION", "auto"))
```

| Environment | Tool | Config |
|---|---|---|
| Local dev | [MiniStack](https://github.com/ministackorg/ministack) — free MIT, 38 AWS services, <1s startup | `AWS_ENDPOINT_URL=http://localhost:4566` |
| VPS production | Cloudflare R2 — 10GB free, no egress fees, S3-compatible | `AWS_ENDPOINT_URL=https://<id>.r2.cloudflarestorage.com` |
| AWS migration | Real AWS S3 | Unset `AWS_ENDPOINT_URL` |

---

## See the Product

→ **[nastart](https://github.com/ardyutama/nastart)** — the recipe costing app built using this methodology
