# OpenCode Skills Collection

Curated skill definitions for OpenCode agent workflows, organized by domain.

## Overview

This repository contains reusable skill definitions that guide OpenCode agents through specific workflows. Each skill is a self-contained module in `.opencode/skills/<name>/SKILL.md` that can be loaded by OpenCode.

## Skill Categories

### DevSecOps (Free Infrastructure Discovery)
Skills for discovering zero-cost DevSecOps infrastructure:

| Skill | Purpose |
|-------|---------|
| `devsecops-free-discovery` | Orchestrates overall discovery workflow |
| `devsecops-free-cloud` | Free IaaS/PaaS compute (GCP, AWS, Azure, Oracle, IBM, Cloudflare) |
| `devsecops-free-cicd` | Free CI/CD pipelines, build systems, artifact repos |
| `devsecops-free-security` | Free SAST/DAST, vulnerability scanners, secrets, PKI |
| `devsecops-free-monitoring` | Free monitoring, logging, APM, observability |
| `devsecops-free-dns` | Free DNS, CDN, SSL, DDoS protection |
| `devsecops-free-storage` | Free object storage, databases, file hosting |
| `devsecops-free-auth` | Free authentication, authorization, identity services |

### Sources
Skills are derived from:
- [free-for-dev](https://github.com/ripienaar/free-for-dev) - Free tier infrastructure catalog

## Installation

### Symlink to workspace
```bash
ln -s /path/to/opencode-skills/.opencode/skills/* ~/.local/share/opencode/skills/
```

### Copy to workspace
```bash
cp -r .opencode/skills/* /path/to/your/workspace/.opencode/skills/
```

## Skill Structure

Each skill follows this format:

```
.skills/<skill-name>/
├── SKILL.md          # Required: Skill definition with YAML frontmatter
└── reference/        # Optional: Additional documentation
```

### SKILL.md Format

```yaml
---
name: skill-name
description: "Clear, specific description of what this skill does"
---

# Skill: Skill Name

## Goal
Single sentence objective.

## Use This Skill When
- Trigger condition

## Do Not Use This Skill When
- Anti-trigger condition

## Inputs
- Required inputs

## Steps
1. Step one

## Output
- Deliverables
```

## Creating New Skills

1. Create directory: `mkdir -p .opencode/skills/your-skill-name`
2. Write `SKILL.md` with YAML frontmatter
3. Follow the skill-authoring skill guidelines
4. Test loading with OpenCode

## Naming Conventions

- Use kebab-case: `my-skill-name`
- Domain prefix for categories: `devsecops-free-cloud`
- Action-oriented: `create-`, `deploy-`, `test-`

## License

MIT License - See LICENSE file for details.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add your skill following the template
4. Submit a pull request

## Related

- [OpenCode](https://github.com/anomalyco/opencode) - The agent framework
- [free-for-dev](https://github.com/ripienaar/free-for-dev) - Inspiration source