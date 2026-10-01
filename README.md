# Skills

My curated list of handcrafted skills.

## Index

| Skill | Description |
|-------|-------------|
| [big-review](big-review/) | Deep, evidence-first code review for local diffs and GitHub PRs |
| [nextjs-audit](nextjs-audit/) | Evidence-first Next.js (App Router, 14+) audit for framework invariants, security boundaries, caching, effects, and convention drift |
| [save-session](save-session/) | Save searchable Markdown session summaries by project under `~/Documents/sessions/`, with a shared index for finding past work |

## Installation

Clone the repository once and choose a skill from the index:

```bash
git clone https://github.com/martin2844/skills.git
cd skills
skill_name=save-session
```

Then run the commands for your tool below. Copy the whole skill folder so any supporting references are included.

### Claude Code

```bash
mkdir -p ~/.claude/skills
cp -R "$skill_name" ~/.claude/skills/
```

See [Claude Code skill locations](https://code.claude.com/docs/en/skills#choose-where-skills-load).

### OpenCode

```bash
mkdir -p ~/.agents/skills
cp -R "$skill_name" ~/.agents/skills/
```

See [OpenCode skill locations](https://opencode.ai/docs/skills/#place-files).

### Kimi CLI

```bash
mkdir -p "${KIMI_CODE_HOME:-$HOME/.kimi-code}/skills"
cp -R "$skill_name" "${KIMI_CODE_HOME:-$HOME/.kimi-code}/skills/"
```

Kimi also discovers `~/.agents/skills/`. See [Kimi skill locations](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/skills.html#skill-locations).

### Codex

```bash
mkdir -p ~/.agents/skills
cp -R "$skill_name" ~/.agents/skills/
```

See [Codex skill locations](https://developers.openai.com/codex/skills/). OpenCode and Codex share this directory, so one copy is enough if you use both.

Start a new session if the installed skill does not appear.

## Using save-session

Ask your agent to "save-session for my-project" or "find saved sessions about authentication". Summaries are written in the conversation's language and stored under `~/Documents/sessions/<project>/`, with links in `~/Documents/sessions/INDEX.md`. The instructions are in Spanish; no additional packages or services are required.
