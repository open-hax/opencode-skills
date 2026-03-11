# opencode-skills

Skill definitions for OpenCode agent workflows.

## Installation

Link or copy skills to your workspace:

```bash
# Option 1: Symlink to workspace .opencode/skills/
ln -s $(pwd)/.opencode/skills/* /path/to/workspace/.opencode/skills/

# Option 2: Symlink to user-level .agents/skills/
ln -s $(pwd)/.opencode/skills/* ~/.agents/skills/
```

## Available Skills

### DevSecOps Free Infrastructure

Skills for discovering free DevSecOps infrastructure:

- `devsecops-free-discovery` - Main orchestrator
- `devsecops-free-cloud` - Compute, storage, networking
- `devsecops-free-cicd` - CI/CD, build systems
- `devsecops-free-security` - SAST/DAST, secrets, PKI
- `devsecops-free-monitoring` - Monitoring, logging, APM
- `devsecops-free-dns` - DNS, CDN, SSL, DDoS
- `devsecops-free-storage` - Object storage, databases
- `devsecops-free-auth` - Authentication, authorization

## Usage

Invoke skills through OpenCode:

```
Use devsecops-free-discovery to find free infrastructure for my project
```

Or use the discovery skill as an orchestrator that calls sub-skills.

## Repository Structure

```
opencode-skills/
├── README.md              # Documentation
├── AGENTS.md              # This file - installation guide
├── .opencode/
│   └── skills/
│       ├── devsecops-free-auth/
│       │   └── SKILL.md
│       ├── devsecops-free-cicd/
│       │   └── SKILL.md
│       └── ...
└── LICENSE
```

## Adding New Skills

1. Create `.opencode/skills/<skill-name>/SKILL.md`
2. Add YAML frontmatter with `name` and `description`
3. Follow the skill-authoring conventions
4. Document the skill in this file