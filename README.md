# Skills

A collection of specialized skills for AI agents (Claude Code, Claude API).
Each skill provides domain-specific knowledge, patterns, and best practices
to extend agent capabilities for real-world engineering tasks.

---

## Available Skills

### [docker-expert](./docker-expert/SKILL.md)

Docker containerization expert with deep, practical knowledge of multi-stage builds,
image optimization, `.dockerignore` strategy, Docker Compose orchestration (dev and prod),
container security, resource limits, health checks, and production deployment patterns.

**Use for:**
- Dockerfile review and optimization (image size, layer caching, multi-stage)
- `.dockerignore` creation and audit
- Docker Compose configuration (dev/prod environments)
- Build cache management and cleanup
- Container security hardening
- Volume strategy (named vs anonymous vs bind mounts)
- Debugging container issues (OOM, MissingGreenlet, startup failures)

**Key patterns:**
- `pip wheel` multi-stage build for Python (removes gcc from runtime)
- Next.js standalone multi-stage (2.8GB dev → 200MB prod)
- Complete `.dockerignore` templates (Python and Node.js)
- Dev vs prod Docker Compose patterns with resource limits and healthchecks

**Resources:** [`resources/implementation-playbook.md`](./docker-expert/resources/implementation-playbook.md)
contains complete, copy-paste ready configurations including full project setup,
GitHub Actions CI/CD workflows, VPS deployment checklist, and registry integration.

---

## Skill Structure

Each skill follows this format:

```
<skill-name>/
├── SKILL.md                          # Main skill with patterns and knowledge
└── resources/
    └── implementation-playbook.md    # Complete code examples and configs
```

### `SKILL.md` sections:
- **When to use / not use** — clear scope boundaries
- **Diagnostic commands** — always start here before changing anything
- **Core knowledge** — root causes of common problems
- **Patterns** — copy-paste ready solutions
- **Debugging guide** — common issues and fixes
- **Checklist** — code review checklist

---

## How to Use

### With Claude Code
```
Use the .agent/skills/<skill-name>/SKILL.md skill
```

### With Claude API
Include the skill content in the system prompt or as a user message.

### Adding to your project
```bash
# Clone or copy the skill to your project
mkdir -p .agent/skills
cp -r docker-expert .agent/skills/
```

---

## Contributing

Skills should be:
- **Battle-tested** — based on real production experience, not documentation
- **Opinionated** — recommend the best approach, not all approaches
- **Actionable** — every pattern should be copy-paste ready
- **Honest** — include known limitations and failure modes
