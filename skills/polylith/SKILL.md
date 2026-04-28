---
name: polylith
description: Use when working with Polylith — a Clojure/ClojureScript architecture tool. Activate when the user asks about bricks, components, bases, projects, interfaces, poly commands (check, info, diff, deps, test, libs, create, ws, shell), workspace.edn configuration, profiles, stable tagging, incremental testing, or structuring a Clojure/ClojureScript monorepo with Polylith.
version: 1.0.0
---

# Polylith

Polylith is a Clojure architecture tool for building systems from reusable, composable **bricks** (components and bases) in a single git repository. One repo, many deployable artifacts, each brick tested and composed independently.

## Quick Decision: What do you need?

| Task | Go to |
|------|-------|
| Understand workspace, components, bases, projects, interfaces | [core-concepts.md](core-concepts.md) |
| Run `poly check / info / diff / deps / test / libs / create / ws` | [commands.md](commands.md) |
| Create a component, wire it into a project, use profiles, tag stable | [workflows.md](workflows.md) |
| workspace.edn, deps.edn patterns, test config, user config | [configuration.md](configuration.md) |
| Mixed Clojure + ClojureScript project in one Polylith workspace | [clj-cljs-project.md](clj-cljs-project.md) |
| Something isn't working / weird behavior / things to avoid | [anti-patterns.md](anti-patterns.md) |

## Core Mental Model

Every brick (component or base) is self-contained with its own `src/`, `test/`, `resources/`, and `deps.edn`. **Components** expose a public interface namespace; other bricks may only require that interface, never the implementation. **Bases** are entry points (APIs, CLIs) — they consume components but have no interface themselves. **Projects** declare which bricks to bundle into a deployable artifact. The **development project** (root `deps.edn`) merges everything for REPL work.

Poly uses git to track what changed since a stable tag and only tests affected bricks — making a large monorepo feel fast.
