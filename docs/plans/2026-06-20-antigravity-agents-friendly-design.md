# Design: Making comfyui-mcp Antigravity Friendly

**Date**: 2026-06-20  
**Author**: Antigravity AI Assistant  
**Status**: Approved  

---

## 1. Goal

The goal of this design is to add full compatibility for Google Antigravity agents to the `comfyui-mcp` repository while keeping the existing Claude Code plugin compatibility fully functional. This is achieved via a dual-support layout where Antigravity assets are dynamically generated and synced from the original Claude Code source files.

Users should be able to choose their preferred agent environment (Claude Code or Antigravity) during installation.

---

## 2. Architecture & Directories

We will introduce two new target folders at the repository root:
*   `.agents/`: Holds workspace-scoped agent assets.
    *   `skills/`: Workspace-scoped skills (`SKILL.md` files).
    *   `hooks.json`: Interceptor configuration for tool execution.
    *   `mcp_config.json`: The Model Context Protocol server registrations.
*   `.gemini/`: Holds configuration and command files.
    *   `commands/`: Holds TOML definitions for custom slash commands.

Additionally, a `GEMINI.md` file will be created at the root of the project to serve as the development guide for Antigravity agents (mirroring `CLAUDE.md`).

These directories **will not be committed** to git. Instead, they will be added to `.gitignore` and generated automatically during `npm install` to avoid maintenance drift.

---

## 3. Component Details

### 3.1. MCP Config (`.agents/mcp_config.json`)
Registers `comfyui`, `civitai`, and `huggingface` MCP servers.

**Note:** Uses `HUGGINGFACE_TOKEN` (no underscore between HUGGING and FACE) to match the existing `.env.example` convention.

```json
{
  "mcpServers": {
    "comfyui": {
      "command": "node",
      "args": ["dist/index.js"],
      "env": {
        "CIVITAI_API_TOKEN": ""
      }
    },
    "civitai": {
      "url": "https://mcp.civitai.com/mcp",
      "headers": {
        "Authorization": "Bearer ${CIVITAI_API_TOKEN:-}"
      }
    },
    "huggingface": {
      "url": "https://huggingface.co/mcp",
      "headers": {
        "Authorization": "Bearer ${HUGGINGFACE_TOKEN:-}"
      }
    }
  }
}
```

### 3.2. Event Hooks (`.agents/hooks.json`)
Dynamically converted from `plugin/hooks/hooks.json` with two transformations:
1. **Matcher rewrite**: `mcp__plugin_comfy_comfyui__` → `mcp__comfyui__`
2. **Path rewrite**: `${CLAUDE_PLUGIN_ROOT}/hooks/` → `./plugin/hooks/` and `${CLAUDE_PLUGIN_ROOT}/scripts/` → `./plugin/scripts/`

### 3.3. Sync Script (`scripts/sync-agents.mjs`)
A Node.js compilation and synchronization script that converts:
1.  **Skills**: Reads `plugin/skills/*/SKILL.md`, extracts and reformats frontmatter to include only `name` and `description`, and writes to `.agents/skills/<skill-name>/SKILL.md`.
2.  **Agents**: Reads `plugin/agents/*.md`, converts them into specialized Antigravity skills at `.agents/skills/comfy-<agent-name>/SKILL.md`.
3.  **Commands**: Reads `plugin/commands/*.md`, parses the frontmatter, and creates `.gemini/commands/comfy-<command-name>.toml` where the prompt body is placed inside `prompt = """..."""` and argument placeholders are translated.
4.  **Hooks**: Reads `plugin/hooks/hooks.json`, rewrites matcher prefixes and `${CLAUDE_PLUGIN_ROOT}` path references, writes to `.agents/hooks.json`.

All generated files also replace `${CLAUDE_PLUGIN_ROOT}` references in skill/command bodies with workspace-relative `./plugin` paths.

---

## 4. Integration with Build Pipeline

To prevent developers from having to manually sync changes, the script will run automatically:
*   Added script: `"sync-agents": "node scripts/sync-agents.mjs"`
*   `postinstall` pipeline: Runs sync after packages are installed.
*   `build` pipeline: Runs sync during compilation.

---

## 5. Documentation Updates

### 5.1. `GEMINI.md` (NEW)
Root-level development guide for Antigravity agents, mirroring `CLAUDE.md` with:
- Local testing instructions using the Antigravity environment
- MCP server configuration references
- Beads issue tracker integration

### 5.2. `README.md` (MODIFY)
Add a new "Agent Environment Setup" section that:
- Explains both Claude Code and Antigravity are supported
- Provides step-by-step installation instructions for each environment
- Links to `CLAUDE.md` and `GEMINI.md` for agent-specific development notes

### 5.3. `CHANGELOG.md` (MODIFY)
Add an entry documenting the Antigravity agent compatibility feature.

### 5.4. `ROADMAP.md` (MODIFY)
Add future items for continued Antigravity integration improvements.

---

## 6. Verification Plan

*   **Static Review**: Ensure JavaScript/TypeScript scripts compile without errors.
*   **Manual Trigger**: Run `npm run sync-agents` locally and verify that:
    1.  `.agents/skills/` contains directories matching the source folders.
    2.  `.gemini/commands/` contains `.toml` files (e.g., `comfy-gen.toml`).
    3.  `.agents/hooks.json` matches the tool prefixes for Antigravity and uses `./plugin/` paths.
    4.  `.agents/mcp_config.json` correctly registers the servers with `HUGGINGFACE_TOKEN`.
    5.  No references to `${CLAUDE_PLUGIN_ROOT}` remain in generated files.
*   **Documentation Review**: Verify README, CHANGELOG, and ROADMAP updates are accurate.
