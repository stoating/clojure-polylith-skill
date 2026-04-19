# Agent instructions

This repository contains the **`polylith`** skill — a structured set of Markdown files that teach an AI coding agent how to work with Polylith, the Clojure/ClojureScript architecture tool for building systems from reusable, composable bricks in a monorepo.

**Any agent that understands this `AGENTS.md` convention should:**

1. Treat `clojure-polylith/SKILL.md` as the entry point — it contains a decision table pointing to the right reference file for the task.
2. Load reference files on demand based on that table:
   - `core-concepts.md` — workspace, components, bases, projects, interfaces, brick structure
   - `commands.md` — `poly check`, `poly info`, `poly diff`, `poly deps`, `poly test`, `poly libs`, `poly create`, `poly ws`, `poly shell`
   - `workflows.md` — creating components, wiring bricks into projects, using profiles, tagging stable
   - `configuration.md` — `workspace.edn`, `deps.edn` patterns, test config, user config
   - `clj-cljs-project.md` — mixed Clojure + ClojureScript projects in one Polylith workspace
   - `anti-patterns.md` — common mistakes and things to avoid
3. Never suggest reaching across brick boundaries via implementation namespaces — components must only be consumed through their public interface namespace.
4. Clarify whether the user is working in the development project (REPL/dev) or building a deployable project artifact before suggesting `deps.edn` or alias changes.

This file follows the [agents.md](https://agents.md/) convention and is honored by OpenAI Codex CLI, Cursor, Aider, Zed, Amp, Gemini CLI, Google Jules, Windsurf, Factory, RooCode, and many others.

For Claude Code, the richer native format is `.claude-plugin/` + `clojure-polylith/SKILL.md`.
