# Feel Physics Claude Code Marketplace

Claude Code / Codex CLI plugin marketplace by [Feel Physics](https://github.com/feel-physics).

## Plugins

| Plugin | Description | Skills |
|--------|-------------|--------|
| [obsidian](plugins/obsidian/) | Obsidian-related skills | show-dotfiles |
| [news](plugins/news/) | AI news tracking | check-ai-trusted-posts |
| [rails](plugins/rails/) | Rails development helpers | adding-undo-to-rails-crud, verify-local-rails-runtime |

## Install

### Claude Code

```bash
# 1. Add marketplace
/plugin marketplace add https://github.com/feel-physics/claude-code-marketplace.git

# 2. Install a plugin
/plugin install obsidian@feel-physics
/plugin install news@feel-physics
/plugin install rails@feel-physics
```

### Codex CLI

```bash
# Via $skill-installer
$skill-installer https://github.com/feel-physics/claude-code-marketplace

# Or manually copy skills to your project or home directory
cp -r plugins/rails/skills/* .agents/skills/       # per-repo
cp -r plugins/rails/skills/* ~/.agents/skills/      # per-user
```

Codex CLI reads skills from `.agents/skills/` (repo), `$HOME/.agents/skills/` (user), or `/etc/codex/skills/` (system).

## License

MIT
