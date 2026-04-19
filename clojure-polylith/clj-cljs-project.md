# Polylith: Mixed Clojure + ClojureScript Project

## Contents
- When to use this setup
- workspace.edn for mixed dialects
- Interface namespace: why `ifc` instead of `interface`
- Shared code with .cljc
- Component structure for mixed dialect
- shadow-cljs integration
- deps.edn + package.json layout
- Dev project setup
- Building CLJS artifacts
- Testing considerations
- Common pitfalls

---

## When to Use This Setup

Use a mixed Clj/Cljs Polylith workspace when:

- You have a backend (Clojure) and a frontend (ClojureScript) in the same repo
- You want to share data schemas, validation, utility logic, or domain models between frontend and backend via `.cljc` files
- You want Polylith's incremental testing and composable bricks to cover both targets

---

## workspace.edn for Mixed Dialects

```clojure
{:top-namespace "myapp"

 ;; REQUIRED for ClojureScript — both dialects
 :dialects #{"clj" "cljs"}

 ;; REQUIRED for ClojureScript — "interface" is a reserved CLJS keyword
 :interface-ns "ifc"

 :vcs {:name "git" :auto-add false}

 ;; Pass shadow-cljs version to component templates
 :template-data {:clojure-ver    "1.12.0"
                 :shadow-cljs-ver "^3.2.0"}

 :projects {"development"  {:alias "dev"}
            "backend"      {:alias "be"}
            "frontend"     {:alias "fe"}}}
```

`.cljc` files are always read regardless of `:dialects`. Adding `"cljs"` makes poly also parse `.cljs` files.

---

## Interface Namespace: `ifc` Instead of `interface`

`interface` is a **reserved keyword** in ClojureScript and an **invalid identifier** in Kotlin. In any workspace touching CLJS, set:

```clojure
;; workspace.edn
:interface-ns "ifc"
```

This renames the public API namespace for every brick:

| Before (`interface`) | After (`ifc`) |
|----------------------|---------------|
| `myapp.user.interface` | `myapp.user.ifc` |
| `myapp.auth.interface` | `myapp.auth.ifc` |

All `poly create component` commands will generate `ifc.clj` (or `ifc.cljc`) instead of `interface.clj`. Update any existing bricks manually if migrating.

---

## Shared Code with `.cljc`

`.cljc` files compile to both Clojure and ClojureScript. Use them for:

- Domain model definitions
- Data validation (Malli, Spec)
- Pure utility functions
- Constants and enums

```clojure
;; components/user/src/myapp/user/ifc.cljc
(ns myapp.user.ifc
  (:require [myapp.user.core :as core]))

(defn validate [user] (core/validate user))
(defn display-name [user] (core/display-name user))
```

```clojure
;; components/user/src/myapp/user/schema.cljc
(ns myapp.user.schema
  (:require [malli.core :as m]))

(def User
  [:map
   [:id   :uuid]
   [:name :string]
   [:email [:re #".+@.+"]]])
```

Use reader conditionals for platform-specific branches:

```clojure
(defn uuid []
  #?(:clj  (java.util.UUID/randomUUID)
     :cljs (random-uuid)))
```

---

## Component Structure for Mixed Dialect

A component that targets both platforms:

```
components/user/
├── src/
│   └── myapp/user/
│       ├── ifc.cljc       ← shared interface (cljc for both platforms)
│       ├── core.cljc      ← shared implementation
│       ├── server.clj     ← Clojure-only impl (DB access, etc.)
│       └── client.cljs    ← ClojureScript-only impl (browser APIs, etc.)
├── test/
│   └── myapp/user/
│       ├── ifc_test.clj   ← Clojure tests (poly runs these)
│       └── ifc_test.cljs  ← ClojureScript tests (run via shadow-cljs)
├── resources/
├── deps.edn               ← JVM deps
└── package.json           ← NPM deps (optional, for CLJS-specific libs)
```

Components that are purely backend (Clojure-only) can still use `.clj` files; only CLJS-touching components need `.cljc` or `.cljs`.

---

## shadow-cljs Integration

shadow-cljs is the standard build tool for ClojureScript in a Clojure deps.edn workspace.

### Root-level shadow-cljs.edn (or package.json config)

```clojure
;; shadow-cljs.edn at workspace root
{:source-paths ["components/ui/src"
                "components/user/src"
                "bases/frontend/src"
                "development/src"]

 :dependencies [[reagent "1.2.0"]
                [re-frame "1.4.0"]]

 :dev-http {8080 "public/"}

 :builds {:app {:target     :browser
                :output-dir "public/js"
                :asset-path "/js"
                :modules    {:main {:init-fn myapp.frontend.core/init}}
                :devtools   {:after-load myapp.frontend.core/reload}}

          :test {:target    :node-test
                 :output-to "target/node-tests.js"
                 :ns-regexp "-test$"}}}
```

### package.json at workspace root

```json
{
  "name": "myapp-workspace",
  "private": true,
  "devDependencies": {
    "shadow-cljs": "2.28.15"
  },
  "dependencies": {}
}
```

Per-component `package.json` files are merged by shadow-cljs for component-specific NPM deps:

```json
// components/ui/package.json
{
  "dependencies": {
    "react": "18.3.1",
    "react-dom": "18.3.1"
  }
}
```

---

## deps.edn Layout

### Root deps.edn (development project)

```clojure
{:aliases
 {:dev  {:extra-paths ["development/src"]
         :extra-deps  {poly/user     {:local/root "components/user"}
                       poly/auth     {:local/root "components/auth"}
                       poly/frontend {:local/root "bases/frontend"}
                       poly/backend  {:local/root "bases/backend"}
                       org.clojure/clojure       {:mvn/version "1.12.0"}
                       org.clojure/clojurescript {:mvn/version "1.11.132"}
                       thheller/shadow-cljs       {:mvn/version "2.28.15"}}}

  :test {:extra-paths ["components/user/test"
                       "components/auth/test"
                       "bases/backend/test"]}}}
```

### Component deps.edn (shared component)

```clojure
;; components/user/deps.edn
{:paths ["src" "resources"]
 :deps  {metosin/malli {:mvn/version "0.16.4"}}
 :aliases {:test {:extra-paths ["test"]
                  :extra-deps  {org.clojure/test.check {:mvn/version "1.1.1"}}}}}
```

### Backend project deps.edn

```clojure
;; projects/backend/deps.edn
{:deps {poly/user    {:local/root "../../components/user"}
        poly/auth    {:local/root "../../components/auth"}
        poly/backend {:local/root "../../bases/backend"}
        org.clojure/clojure {:mvn/version "1.12.0"}
        ring/ring-core      {:mvn/version "1.12.2"}}
 :aliases {:test    {:extra-paths ["../../components/user/test"
                                   "../../components/auth/test"
                                   "../../bases/backend/test"]}
           :uberjar {:main myapp.backend.core}}}
```

### Frontend project deps.edn

```clojure
;; projects/frontend/deps.edn
{:deps {poly/user     {:local/root "../../components/user"}
        poly/auth     {:local/root "../../components/auth"}
        poly/frontend {:local/root "../../bases/frontend"}
        org.clojure/clojurescript {:mvn/version "1.11.132"}
        thheller/shadow-cljs      {:mvn/version "2.28.15"}}
 :aliases {:test {:extra-paths ["../../components/user/test"
                                "../../bases/frontend/test"]}}}
```

---

## Dev Project Setup

Start shadow-cljs for frontend development alongside your Clojure REPL:

```bash
# Terminal 1 — Clojure REPL with all bricks
clj -M:dev

# Terminal 2 — shadow-cljs watch for frontend
npx shadow-cljs watch app

# Or via clj alias
clj -M:dev:shadow-cljs watch app
```

shadow-cljs picks up source paths from `shadow-cljs.edn` (or root `package.json`). Ensure all CLJS component `src/` paths are listed there.

---

## Building CLJS Artifacts

### Development build (watch mode)

```bash
npx shadow-cljs watch app
```

### Release build

```bash
npx shadow-cljs release app
```

### From a clj alias

```clojure
;; root deps.edn
{:aliases {:build-fe {:extra-deps {thheller/shadow-cljs {:mvn/version "2.28.15"}}
                      :main-opts  ["-m" "shadow.cljs.devtools.cli" "release" "app"]}}}
```

```bash
clj -M:dev:build-fe
```

---

## Testing Considerations

### Clojure tests (run by poly)

`poly test` runs only JVM (`.clj`) tests — it does not execute ClojureScript tests. Set up `clojure.test`-based tests under each component's `test/` using `.clj` or `.cljc` files.

```bash
poly test :all-bricks   # runs all JVM tests across all bricks
```

### ClojureScript tests (run by shadow-cljs)

CLJS tests are run separately via shadow-cljs:

```bash
# Node.js test runner
npx shadow-cljs compile test && node target/node-tests.js

# Or: watch + karma for browser tests
npx shadow-cljs watch test
```

### Shared .cljc test files

You can write tests in `.cljc` that run on both platforms. Poly will run them as Clojure tests; shadow-cljs will run them as ClojureScript tests.

```clojure
;; components/user/test/myapp/user/ifc_test.cljc
(ns myapp.user.ifc-test
  (:require #?(:clj  [clojure.test :refer [deftest is]]
               :cljs [cljs.test    :refer [deftest is] :include-macros true])
            [myapp.user.ifc :as user]))

(deftest validate-test
  (is (user/validate {:id #uuid "..." :name "Alice" :email "a@b.com"})))
```

### CI strategy

```bash
# JVM tests (poly handles incremental)
poly test :all

# CLJS tests (run separately)
npx shadow-cljs compile test
node target/node-tests.js
```
