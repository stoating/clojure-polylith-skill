# Polylith Anti-Patterns

## Contents

- Dependency anti-patterns
- Project structure anti-patterns
- Interface anti-patterns
- Testing anti-patterns
- Profiles anti-patterns
- Git / tagging anti-patterns
- Library anti-patterns
- IDE / tooling anti-patterns
- Quick diagnostic checklist

---

## Dependency Anti-Patterns

### Requiring implementation namespaces instead of interfaces

```clojure
;; WRONG — bypasses the interface contract, breaks poly validation
(ns se.example.rest-api.core
  (:require [se.example.user.core :as user]))  ; ← impl namespace

;; RIGHT
(ns se.example.rest-api.core
  (:require [se.example.user.interface :as user]))  ; ← interface namespace
```

**Error 101** — Poly will catch this in `poly check`.

---

### Declaring cross-brick deps inside a brick's deps.edn

```clojure
;; WRONG — brick deps.edn lists another brick
;; components/rest-api/deps.edn
{:deps {poly/user {:local/root "../user"}}}  ; ← wrong place

;; RIGHT — put cross-brick wiring in the project's deps.edn
;; projects/backend/deps.edn
{:deps {poly/user    {:local/root "../../components/user"}
        poly/rest-api {:local/root "../../bases/rest-api"}}}
```

**Error 112** — Poly validates this. Bricks declare only third-party deps in their own `deps.edn`.

---

### Circular dependencies

```text
component/a requires interface of b
component/b requires interface of a
```

**Error 104.** Break the cycle by extracting shared logic into a new component that both `a` and `b` depend on.

---

## Project Structure Anti-Patterns

### Adding `src/` to a deployable project

```text
projects/backend/
├── src/           ← WRONG — all prod code must be in bricks
│   └── se/example/...
└── deps.edn
```

Source code in `projects/*/src/` is not tested the same way as brick code and undermines the Polylith model. Keep all production logic in components or bases.

---

### Forgetting to add test paths to a project

```clojure
;; WRONG — test paths from :local/root are NOT auto-included
{:deps {poly/user {:local/root "../../components/user"}}}

;; RIGHT
{:deps    {poly/user {:local/root "../../components/user"}}
 :aliases {:test {:extra-paths ["../../components/user/test"]}}}
```

Without explicit test paths, the project classloader won't see tests for those bricks.

---

### Not adding a brick to the development project

```clojure
;; After poly create component name:mycomp, you must also add it here:
;; root deps.edn
{:aliases
 {:dev {:extra-deps {poly/mycomp {:local/root "components/mycomp"}}  ; ← add this
        :extra-paths ["components/mycomp/test"]}}}                    ; ← and this
```

Without this, the REPL and dev tooling won't see the new component.

---

## Interface Anti-Patterns

### Putting implementation logic directly in the interface

```clojure
;; WRONG — logic belongs in impl namespaces
(ns se.example.user.interface)

(defn get-user [id]
  (jdbc/query db ["SELECT * FROM users WHERE id=?" id]))  ; ← not here

;; RIGHT — delegate to core
(ns se.example.user.interface
  (:require [se.example.user.core :as core]))

(defn get-user [id] (core/get-user id))
```

The interface should be a thin contract. This makes it easier to swap implementations.

---

### Mismatching signatures across components implementing the same interface

```clojure
;; components/user/src/.../interface.clj
(defn get-user [id])          ; 1 arg

;; components/user-remote/src/.../interface.clj
(defn get-user [id opts])     ; 2 args — WRONG
```

**Warning 201 / Error 103.** All components sharing an interface must have identical arities, argument order, and type hints.

---

### Using `interface` as namespace name with ClojureScript

`interface` is a reserved keyword in ClojureScript. In a CLJS workspace:

```clojure
;; WRONG — will break CLJS compilation
;; namespace: se.example.user.interface

;; RIGHT — set in workspace.edn
:interface-ns "ifc"

;; Then namespace becomes: se.example.user.ifc
```

See [clj-cljs-project.md](clj-cljs-project.md) for full CLJS setup.

---

## Testing Anti-Patterns

### Running `poly test` from outside the workspace root

Poly must be run from the workspace root (where `workspace.edn` lives). Otherwise it won't find the workspace or will analyse the wrong one.

---

### Relying on test ordering across bricks

Each project gets an isolated classloader. Tests within a brick run sequentially (by default) but brick test suites are independent. Don't assume global state from one brick test leaks into another.

---

### Skipping `poly test` and only running tests from the REPL

Running tests in the REPL uses the dev project classpath, which includes all bricks. Running `poly test` checks each deployable project in isolation with only its declared bricks — it can catch classpath leakage that REPL tests miss.

---

### Not providing a setup namespace for integration tests

If tests require a running DB, server, or other resource:

```clojure
;; workspace.edn
:projects {"backend" {:test {:setup-fn    project.backend.test-setup/start
                             :teardown-fn project.backend.test-setup/stop}}}
```

Without this, tests that assume the resource will fail inconsistently.

---

## Profiles Anti-Patterns

### Multiple implementations of the same interface active in dev simultaneously

```text
:+default includes poly/user
:+myprofile also includes poly/user-remote (implements same interface)

poly info +myprofile   ← both active = Error 108
```

Only one implementation of a given interface can be active in the dev project at a time. Use profiles to switch between them, never activate both.

---

### Profile path also present in dev

```clojure
;; WRONG — same path in both :dev and :+default
:dev      {:extra-paths ["components/user/src"]}
:+default {:extra-paths ["components/user/src"]}  ; ← duplicate
```

**Warning 203.** Remove from one of them.

---

## Git / Tagging Anti-Patterns

### No stable tag — `poly diff` compares to first commit

If no tag matching `:stable` pattern exists, poly uses the very first commit as the baseline. Everything looks changed.

```bash
# Fix: create a stable tag
git tag -f stable-myname
```

---

### Using a shared stable tag across developers

```bash
# WRONG — developer B's push moves the tag for developer A
git tag -f stable
git push --force origin stable
```

Each developer should use their own tag name:

```bash
git tag -f stable-lisa   # Lisa's baseline
git tag -f stable-john   # John's baseline
git tag -f stable-ci     # CI's baseline
```

---

### Forgetting `:git-add` after creating a brick

When `:vcs :auto-add false` (the default), `poly create` does not stage files:

```bash
poly create component name:mycomp :git-add   # ← add the flag
# or manually:
git add components/mycomp
```

Without git tracking, poly won't detect changes to the new brick.

---

## Library Anti-Patterns

### Different versions of the same library across bricks in the same project

```text
components/user/deps.edn       → ring/ring 1.9.6
components/rest-api/deps.edn   → ring/ring 2.0.0
```

Both end up on the same project classpath — one version wins arbitrarily. **Warning 301.**

Use `poly libs` to spot version matrix conflicts, then align versions.

---

### Not setting `:keep-lib-versions` for intentionally pinned libraries

```bash
poly libs :update   # updates everything, including pinned libs
```

If a library must stay at a specific version, declare it:

```clojure
;; workspace.edn
:projects {"backend" {:keep-lib-versions [ring/ring]}}
```

---

## IDE / Tooling Anti-Patterns

### Putting non-Clojure files in `src/`

```text
components/mycomp/src/
├── se/example/mycomp/
│   └── interface.clj
└── config.json    ← WRONG — poly tries to parse this as Clojure, Error 111
```

Non-Clojure files (JSON, images, SQL, EDN config) go in `resources/`. Files there are not namespace-checked.

---

### Expecting IDE to auto-discover `:local/root` dependencies

Not all IDEs support `:local/root` for source path resolution:

- **Cursive 1.13+**: supported — click "Import Changes" after adding a brick
- **Calva / CIDER**: supported via `clojure-lsp`
- **Older Cursive**: use `:extra-paths` as fallback (less ideal):
  ```clojure
  :extra-paths ["components/user/src" "components/user/resources"]
  ```

---

## Quick Diagnostic Checklist

When something doesn't work:

```text
[ ] Did you run poly check? Read all errors/warnings.
[ ] Is the brick in both the dev project AND the deployable project deps.edn?
[ ] Are test paths explicitly listed in the project :test alias?
[ ] Did you git add newly created files?
[ ] Is there a stable tag? (git tag -l | grep stable)
[ ] Are you requiring the interface namespace, not impl?
[ ] Is there a circular dependency? (poly deps, look for cycles)
[ ] Are library versions consistent? (poly libs)
[ ] For CLJS: is :interface-ns "ifc" set in workspace.edn?
[ ] Is the namespace in every source file starting with :top-namespace?
[ ] Are non-Clojure files in resources/, not src/?
```
