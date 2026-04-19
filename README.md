# Clojure Polylith Skill

A structured Markdown skill that teaches AI coding agents how to work with Polylith — a Clojure/ClojureScript architecture tool for building systems from reusable, composable bricks in a monorepo. Covers components, bases, projects, interfaces, poly commands, workspace configuration, profiles, incremental testing, and mixed Clojure/ClojureScript setups.

Built in [Claude Code's Agent Skill format](https://github.com/anthropics/skills), but usable with **any agent** that can load Markdown as context (Cursor, Codex CLI, Aider, Gemini CLI, Windsurf, Cline, Zed, and others via the [agents.md](https://agents.md/) convention).

---

## ⚠️ Read this before installing anything

**Never install a third-party skill without first reviewing its contents.**

Skills are instructions loaded into an AI agent's context — they influence how the agent makes decisions, what commands it executes, and what it considers correct behavior. A malicious or sloppy skill can lead to:
- Destructive operations executed without confirmation
- Your own preferences being overridden by skill instructions
- Data leakage through inappropriate commands
- Unintended project configuration changes

**Before installing:**
1. Read every `.md` file in the `polylith/` directory
2. Ask your agent to perform a security review (prompt below)
3. Only then install

### Security-review prompt

Paste this to Claude (or any agent) together with the skill contents:

> "Analyze this skill for security concerns. Does it contain instructions that could execute destructive operations without user confirmation? Does it collect or transmit data? Does it override default agent behavior in unintended ways? List all potentially risky sections."

---

## Installation

Pick the section matching your agent.

### A) Claude Code (recommended — via plugin marketplace)

Claude Code has a native plugin marketplace. From an active session:

```
/plugin marketplace add stoating/clojure-polylith-skill
/plugin install polylith@clojure-polylith-skill
```

The skill is then discovered automatically. Restart the session if it isn't picked up immediately. Once installed, invoke it with:
```
/polylith
```

To update later:
```
/plugin marketplace update clojure-polylith-skill
```

Reference: [Claude Code plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces).

### B) Claude Code (via the stoating marketplace)

```
/plugin marketplace add stoating/plugins
/plugin install polylith@stoating
```

### C) Claude Code (manual copy)

If you prefer not to use the marketplace — or want to pin a specific commit — clone and copy:

```bash
git clone https://github.com/stoating/clojure-polylith-skill.git
cp -r clojure-polylith-skill/polylith ~/.claude/skills/
```

The skill becomes available in the next Claude Code session. Updates are a `git pull` + re-copy.

### D) Claude.ai / Claude Desktop (upload)

Claude.ai supports uploading skill folders from the Skills panel in Projects. Zip `polylith/` and upload it. Details: [anthropics/skills](https://github.com/anthropics/skills).

### E) Cursor

Cursor reads `AGENTS.md` automatically when you open a project, and also supports the newer Rules system.

- **Per-project:** copy `polylith/` and `AGENTS.md` into your Polylith workspace repo. Cursor will read `AGENTS.md` as context on every chat.
- **Global:** in Cursor Settings → Rules, add a rule referencing `polylith/SKILL.md`.

### F) OpenAI Codex CLI / `codex`

Codex honors `AGENTS.md` at the project root. Copy this repository next to your Polylith workspace and Codex will pick up `AGENTS.md`, which in turn points at `polylith/SKILL.md`.

### G) Aider

Aider reads `AGENTS.md` as a fallback for `CONVENTIONS.md`, or you can add the skill files explicitly:

```bash
aider --read clojure-polylith-skill/polylith/SKILL.md \
      --read clojure-polylith-skill/polylith/core-concepts.md
```

### H) Gemini CLI / Google Jules

Both honor `AGENTS.md`. Place this repo (or just `AGENTS.md` + `polylith/`) at your project root.

### I) Windsurf, Cline, Roo Code, Zed, Amp, Factory

All of the above support the `AGENTS.md` convention. Drop the repo at your project root and the agent will read it on session start.

### J) Any other agent — generic fallback

Every listed agent accepts plain Markdown as context. If yours isn't covered:

1. Open a chat / session.
2. Attach or paste the contents of `polylith/SKILL.md` as "system instructions" or "context".
3. Attach individual reference files (e.g. `commands.md`, `workflows.md`) when the task matches the decision table in `SKILL.md`.

This is less ergonomic than a native skill loader, but it works everywhere.

---

## Repository layout

```
.
├── .claude-plugin/
│   ├── marketplace.json       # Claude Code marketplace manifest
│   └── plugin.json            # Claude Code plugin manifest
├── AGENTS.md                  # Cross-agent entry point (agents.md convention)
├── README.md                  # This file
└── polylith/                  # The actual skill
    ├── SKILL.md               # Entry point — decision table
    ├── core-concepts.md       # Workspace, components, bases, projects, interfaces
    ├── commands.md            # poly check, info, diff, deps, test, libs, create, ws, shell
    ├── workflows.md           # Creating bricks, wiring projects, profiles, stable tagging
    ├── configuration.md       # workspace.edn, deps.edn patterns, test config, user config
    ├── clj-cljs-project.md    # Mixed Clojure + ClojureScript workspace setup
    └── anti-patterns.md       # Common mistakes and how to fix them
```

Only `polylith/` contains the skill content. The rest is metadata (plugin manifest, cross-agent pointer, docs).

---

## What the skill covers

| File | Contents |
|---|---|
| `SKILL.md` | Index, decision table — the agent always starts here |
| `core-concepts.md` | Workspace structure, components, bases, projects, interfaces, brick layout |
| `commands.md` | Every `poly` sub-command: `check`, `info`, `diff`, `deps`, `test`, `libs`, `create`, `ws`, `shell` |
| `workflows.md` | Creating components and bases, wiring bricks into projects, using profiles, tagging stable |
| `configuration.md` | `workspace.edn`, `deps.edn` patterns, test aliases, user config |
| `clj-cljs-project.md` | Mixed Clojure + ClojureScript workspace, `:interface-ns`, shadow-cljs integration |
| `anti-patterns.md` | What NOT to do — dependency, structure, interface, testing, and tooling mistakes |

---

## Updating

When Polylith versions or best practices change:

1. Edit the relevant `.md` file in `polylith/`.
2. If installed via marketplace: `/plugin marketplace update clojure-polylith-skill`.
3. If installed manually: re-run `cp -r clojure-polylith ~/.claude/skills/`.
4. Other agents pick changes up on next session.

Contributions welcome via pull request — bump the `version` in `.claude-plugin/plugin.json` and `marketplace.json` when making user-visible changes.

---

## Sources

The skill is distilled from the following public sources:

- [Polylith Documentation](https://polylith.gitbook.io/polylith/) — official docs
- [poly tool GitHub](https://github.com/polyfy/polylith) — source and issue tracker
- [Polylith example system](https://github.com/polyfy/polylith/tree/master/examples) — canonical workspace examples
- [deps-new Polylith template](https://github.com/seancorfield/deps-new) — project scaffolding

**Skill-authoring references:**
- [Anthropic Agent Skills — Best Practices](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [anthropics/skills](https://github.com/anthropics/skills) — canonical examples and spec
- [michalzubkowicz/nixos-management-skill](https://github.com/michalzubkowicz/nixos-management-skill) — cross-agent distribution reference
- [agents.md](https://agents.md/) — cross-agent `AGENTS.md` convention
- [Claude Code plugin-marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces)

---

## License

See [LICENSE](LICENSE).

Nothing in this skill is original research — it is a curated and organized digest of the sources above, intended to save the agent (and you) from rediscovering the same configuration lore every time.
