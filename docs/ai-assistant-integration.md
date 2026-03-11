# AI Assistant Integration

Guide for integrating compute skills with various AI coding assistants.

## Claude Code

### Marketplace (Recommended)

```bash
claude install-skill agent-skills-for-compute
```

### Manual

Clone the repository and add to your project's `.claude/skills/`:

```bash
git clone https://github.com/dtunai/agent-skills-for-compute.git
cp -r agent-skills-for-compute/skills/* .claude/skills/
```

## Cursor

### Remote Rule

Add the GitHub repository URL as a remote rule in Cursor settings:

1. Open Settings > Rules
2. Add remote rule: `https://github.com/dtunai/agent-skills-for-compute`
3. Skills are automatically available in conversations

### Local Clone

```bash
# Project-level
git clone https://github.com/dtunai/agent-skills-for-compute.git .cursor/skills/compute

# User-level
git clone https://github.com/dtunai/agent-skills-for-compute.git ~/.cursor/skills/compute
```

## OpenAI Codex

Add as a reference in your Codex configuration:

```bash
git clone https://github.com/dtunai/agent-skills-for-compute.git
# Point Codex to the skills/ directory
```

## Gemini CLI

```bash
# Clone to Gemini skills directory
git clone https://github.com/dtunai/agent-skills-for-compute.git ~/.gemini/skills/compute
```

## OpenCode

```bash
# Add as skill source
git clone https://github.com/dtunai/agent-skills-for-compute.git ~/.opencode/skills/compute
```

## Skill Discovery

All skills follow the [agentskills.io specification](https://agentskills.io/specification), making them compatible with any AI assistant that supports the standard.

The `.claude-plugin/marketplace.json` file describes available skills for automated discovery.
