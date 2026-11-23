# TNT Agent Plugins (`tnt.agentplugins`)

A repository of agent plugins providing specialized skills and grounding documentation for AI assistants (such as Google Antigravity and Gemini CLI).

## Overview

This repository hosts modular, reusable agent plugins maintained by Tripp'n Technology. Each plugin packages:
- **Grounding Documentation (`rules/AGENTS.md`)**: Always-on or contextual guidelines, architecture standards, conventions, and domain knowledge that ground agent behavior.
- **Agent Skills (`skills/<skill-name>/SKILL.md`)**: Progressive, on-demand procedural runbooks and workflows that teach agents how to perform specific tasks.
- **Reference Documentation (`references/`)**: Detailed manuals, design specifications, and deep-dive documentation loaded progressively by agents when needed.
- **Tooling & Integrations** (optional): Lifecycle hooks (`hooks.json`) and Model Context Protocol configurations (`mcp_config.json`).

---

## Repository Structure

```text
tnt.agentplugins/
├── .gitignore
├── README.md
└── plugins/
    ├── <plugin-name-1>/
    │   ├── plugin.json               # Required plugin manifest
    │   ├── rules/
    │   │   └── AGENTS.md             # Grounding rules & standards
    │   └── skills/
    │       └── <skill-name>/
    │           ├── SKILL.md          # Procedural instructions & frontmatter
    │           └── references/       # Detailed reference manuals
    └── <plugin-name-2>/
        ...
```

---

## How Plugins Work

### 1. Grounding Documentation (`rules/`)
Rules in `rules/AGENTS.md` (or `GEMINI.md`) are automatically applied whenever the plugin is active. They ground the agent with:
- Project architecture and domain model conventions.
- Mandatory coding standards, patterns, and anti-patterns.
- Security, validation, and testing requirements.

### 2. Skills (`skills/`)
Skills provide step-by-step instructions for specific multi-step workflows. They leverage **progressive disclosure**:
- The agent only reads the skill's name and description from the `SKILL.md` frontmatter at startup.
- Full instructions and `references/` are loaded only when a user prompt matches the skill's description.

---

## Using Plugins in Projects

### Option A: Register via `plugins.json` (Recommended)
You can point your project or global Antigravity / Gemini configuration to this repository's `plugins` folder.

In your project's `.agents/plugins.json` or your global config `~/.gemini/config/plugins.json`:

```json
{
  "entries": [
    {
      "path": "D:/Src/tnt.agentplugins/plugins"
    }
  ]
}
```

### Option B: Local Project Inclusion
Copy or symlink the specific plugin directory into your project's `.agents/plugins/`:

```text
my-project/
└── .agents/
    └── plugins/
        └── <plugin-name>/
```

---

## Creating a New Plugin

1. Create a directory under `plugins/<plugin-name>/`.
2. Add a `plugin.json` manifest:
   ```json
   {
     "name": "my-plugin",
     "author": "Tripp'n Technology",
     "description": "Short description of the plugin's purpose."
   }
   ```
3. Add grounding documentation in `rules/AGENTS.md`.
4. Add any skills in `skills/<skill-name>/SKILL.md` with appropriate YAML frontmatter (`name` and `description`).
5. Place large manuals or guides under `skills/<skill-name>/references/` to take advantage of progressive disclosure.
