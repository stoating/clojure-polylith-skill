# Polylith Core Concepts

## Contents
- Workspace
- Components
- Bases
- Projects (development vs deployable)
- Interfaces
- Directory structure
- Namespace conventions
- Dependency rules

---

## Workspace

The root container for everything. Configured by `workspace.edn` at the repo root. Managed by git — poly reads git history to determine what changed and what needs testing.

- One repo, one workspace
- All components, bases, and projects live inside it
- Profiles let different combinations of bricks be active in development

---

## Components

Reusable building blocks with business logic. Live in `components/COMP-NAME/`.

- Have an **interface** namespace — the only namespace other bricks are allowed to require
- Implementation lives in sibling namespaces: `core`, `helper`, etc.
- The interface delegates to implementation; callers never see the impl
- Multiple components can implement the same interface (for different contexts/environments)

Structure:
```
components/mycomp/
├── src/
│   └── TOP_NS/mycomp/
│       ├── interface.clj   ← public API (required)
│       └── core.clj        ← implementation
├── test/
│   └── TOP_NS/mycomp/
│       └── interface_test.clj
├── resources/
└── deps.edn
```

---

## Bases

Entry points to deployable artifacts (REST API, CLI, Lambda, etc.). Live in `bases/BASE-NAME/`.

- **No interface** — bases consume, they don't expose
- Bridge between the outside world and components
- Can require component interfaces and other base namespaces from `:src`
- From `:test`, can require anything

Structure mirrors components but without `interface.clj`.

---

## Projects

### Development Project

The special project used for local REPL work.

- Configured in root `deps.edn` under `:dev` alias
- Merges all components/bases plus optional profiles
- `development/src/` holds dev-only code (e.g. `dev.lisa` — not under top-ns)
- Where you run the REPL and iterate interactively

### Deployable Projects

Live in `projects/PROJECT-NAME/`. Each is a separate deployable artifact.

- Own `deps.edn` listing which bricks to include via `:local/root`
- **No `src/` directory** — all production code must live in bricks
- Optional `test/` and `resources/` for project-level test setup
- Poly runs tests in isolation, one classloader per project

```
projects/myproject/
├── deps.edn        ← lists bricks via :local/root
├── test/           ← optional: setup/teardown namespaces
└── resources/      ← optional: project-level resources
```

---

## Interfaces

The public API contract of a component.

### Location

Single file:
```
components/mycomp/src/TOP_NS/mycomp/interface.clj
```

Or split into sub-namespaces (directory):
```
components/mycomp/src/TOP_NS/mycomp/interface/
├── core.clj        → TOP_NS.mycomp.interface
└── spec.clj        → TOP_NS.mycomp.interface.spec
```

### Valid interface contents

- `(defn name [args] ...)` — function definitions
- `(defmacro name [args] ...)` — macro definitions
- `(def name value)` — value definitions

Best practice: interface functions should be thin wrappers delegating to impl namespaces.

### Consistency validation

When multiple components implement the same interface, poly validates:
- All function/macro arities match (Error 201 if not)
- Argument order matches
- Type hints match
- `defn` vs `defmacro` matches

---

## Directory Structure

```
workspace-root/
├── workspace.edn          ← workspace configuration
├── deps.edn               ← development project deps + profiles
├── components/
│   ├── user/
│   │   ├── src/se/example/user/
│   │   │   ├── interface.clj
│   │   │   └── core.clj
│   │   ├── test/se/example/user/
│   │   │   └── interface_test.clj
│   │   ├── resources/
│   │   └── deps.edn
│   └── ...
├── bases/
│   ├── rest-api/
│   │   ├── src/se/example/rest_api/
│   │   │   └── core.clj
│   │   ├── test/
│   │   ├── resources/
│   │   └── deps.edn
│   └── ...
├── projects/
│   ├── backend/
│   │   ├── deps.edn
│   │   ├── test/
│   │   └── resources/
│   └── ...
└── development/
    └── src/
        └── dev/
            └── lisa.clj   ← dev-only, not under top-ns
```

---

## Namespace Conventions

Format: `TOP-NAMESPACE.BRICK-NAME.PART`

| Context | Namespace example |
|---------|------------------|
| Component interface | `se.example.user.interface` |
| Component impl | `se.example.user.core`, `se.example.user.helper` |
| Sub-interface | `se.example.user.interface.spec` |
| Base | `se.example.rest-api.core` |
| Dev code | `dev.lisa` (NOT under top-ns) |
| Project test setup | `project.backend.test-setup` |

**Dashes in brick names** become underscores in file paths:
- Brick `user-api` → directory `user_api/` → namespace `se.example.user_api.interface`

**All source files** under `src/` must start with `:top-namespace` (poly validates this, Error 205 if not). Files in `resources/` and `test-resources/` are excluded from this check.

---

## Dependency Rules

### From component `:src`
- ✅ May require: `TOP_NS.other-comp.interface` (and sub-namespaces)
- ❌ Must NOT require: `TOP_NS.other-comp.core` or any impl namespace → **Error 101**

### From component `:test`
- ✅ May require anything

### From base `:src`
- ✅ May require: component interface namespaces, other base namespaces
- ❌ Must NOT require: component impl namespaces → **Error 101**

### From base `:test`
- ✅ May require anything

### Cross-brick wiring
Do NOT declare component→component deps in a brick's `deps.edn`. Wire bricks together only in **project** `deps.edn`. Poly infers interface dependencies automatically.
