# Skills

My curated list of handcrafted skills.

## Index

| Skill | Description |
|-------|-------------|
| [big-review](big-review/) | Deep, evidence-first code review for local diffs and GitHub PRs |
| [nextjs-audit](nextjs-audit/) | Evidence-first Next.js (App Router, 14+) audit for framework invariants, security boundaries, caching, effects, and convention drift |

## Installation

### Claude Code

```bash
git clone git@github.com:martin2844/skills.git /tmp/skills-repo
cp -r /tmp/skills-repo/<skill-name> ~/.claude/skills/
```

### OpenCode

```bash
git clone git@github.com:martin2844/skills.git /tmp/skills-repo
cp -r /tmp/skills-repo/<skill-name> ~/.agents/skills/
```

### Kimi CLI

```bash
git clone git@github.com:martin2844/skills.git /tmp/skills-repo
cp -r /tmp/skills-repo/<skill-name> ~/.kimi/skills/
```

### Codex

```bash
git clone git@github.com:martin2844/skills.git /tmp/skills-repo
cp -r /tmp/skills-repo/<skill-name> ~/.codex/skills/
```

Replace `<skill-name>` with the skill you want (e.g. `big-review`). Restart your session after installing for the skill to be discovered.
