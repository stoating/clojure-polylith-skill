# Polylith Configuration

## Contents
- workspace.edn reference
- deps.edn patterns
- User config.edn
- Test configuration
- info output flags reference

---

## workspace.edn Reference

Full example with all common keys:

```clojure
{:vcs {:name "git"
       :auto-add false}        ; true = auto git add on poly create

 :top-namespace "se.example"   ; REQUIRED — all src namespaces must start with this

 :interface-ns "interface"     ; default; use "ifc" for ClojureScript workspaces

 :default-profile-name "default"  ; profile alias :+default merged automatically into dev

 :dialects #{"clj"}            ; add "cljs" for ClojureScript support

 :tag-patterns {:stable  "^stable-.*"    ; used by diff/test/info since:stable
                :release "^v[0-9].*"     ; used by since:release
                :my-key  "^my-prefix-.*"} ; custom; use via since:my-key

 :template-data {:clojure-ver "1.12.0"   ; passed to Selmer templates on create
                 :shadow-cljs-ver "^3.2.0"}

 :projects {"development"  {:alias "dev"}
            "backend"      {:alias "be"
                           :test {:setup-fn    project.backend.test-setup/start
                                  :teardown-fn project.backend.test-setup/stop
                                  :include ["brick-a" "brick-b"]  ; test only these
                                  :exclude ["slow-brick"]}        ; skip these
                           :necessary ["adapter"]                  ; suppress Warning 207
                           :keep-lib-versions [ring/ring]}}        ; skip on libs :update

 :bricks {"mycomp" {:keep-lib-versions [org.slf4j/slf4j-nop]}}

 :compact-views #{}             ; #{"deps" "libs"} for compact output by default

 :validations {:inconsistent-lib-versions {:type :warning   ; :error | :warning | :none
                                           :exclude []}}

 :custom {:anything "here"}}    ; safe namespace for user data; won't conflict with poly keys
```

### Key rules

- `:top-namespace` is required — every source file poly reads must start with it
- Project keys are the **full** project directory name, not an alias
- `:alias` should be short (1-2 chars) — shown in info/deps tables
- `:necessary` suppresses Warning 207 for bricks that look unused but are required at runtime

---

## deps.edn Patterns

### Root deps.edn (development project)

```clojure
{:aliases
 {:dev {:extra-paths ["development/src"]
        :extra-deps  {poly/user    {:local/root "components/user"}
                      poly/rest-api {:local/root "bases/rest-api"}
                      org.clojure/clojure {:mvn/version "1.12.0"}}}

  :test {:extra-paths ["components/user/test"
                       "bases/rest-api/test"]}

  ;; Profiles — must start with :+
  :+default {:extra-deps  {poly/user {:local/root "components/user"}}
             :extra-paths ["components/user/src"]}

  :+remote  {:extra-deps  {poly/user-remote {:local/root "components/user-remote"}}
             :extra-paths ["components/user-remote/src"]}}}
```

### Component / base deps.edn

```clojure
{:paths ["src" "resources"]
 :deps  {metosin/malli {:mvn/version "0.16.4"}}
 :aliases {:test {:extra-paths ["test"]
                  :extra-deps  {lambdaisland/kaocha {:mvn/version "1.91.1392"}}}}}
```

### Project deps.edn

```clojure
{:deps {poly/user     {:local/root "../../components/user"}
        poly/rest-api {:local/root "../../bases/rest-api"}
        org.clojure/clojure {:mvn/version "1.12.0"}}
 :aliases
 {:test    {:extra-paths ["../../components/user/test"
                          "../../bases/rest-api/test"]}
  :uberjar {:main se.example.rest-api.core}}}
```

Note: test paths are **not** auto-included from `:local/root` — list them explicitly.

---

## User config.edn

Location: `~/.config/polylith/config.edn`

```clojure
{:color-mode "dark"          ; "none" | "light" | "dark"
 :empty-character "."        ; character for empty table cells
 :thousand-separator ","     ; separator for LOC numbers
 :m2-dir "~/.m2"            ; Maven repo path for library size calculation

 ;; Workspace shortcuts for poly shell switch-ws
 :ws-shortcuts {:root-dir "/path/to/workspaces"
                :paths [{:dir  "/path/to/ws1"     :name "ws1"}
                        {:file "/path/to/ws.edn"  :name "saved-ws"}]}}
```

Override per-command: `poly info color-mode:none`

---

## Test Configuration

### Setup and teardown

Defined in workspace.edn per project:

```clojure
:projects {"backend" {:test {:setup-fn    project.backend.test-setup/start
                             :teardown-fn project.backend.test-setup/stop}}}
```

The setup/teardown namespace lives in the project's `test/` directory:

```clojure
;; projects/backend/test/project/backend/test_setup.clj
(ns project.backend.test-setup)

(defn start [] (println "Starting test environment..."))
(defn stop  [] (println "Tearing down test environment..."))
```

### Include / exclude bricks from testing

```clojure
:projects {"backend" {:test {:include ["user" "rest-api"]   ; only test these bricks
                             :exclude ["slow-integration"]}}} ; skip these bricks
```

### Custom test runner (e.g. Kaocha)

```clojure
;; workspace.edn
{:test {:create-test-runner my.custom/create-test-runner}}
```

Test runners receive an isolated classloader per project. Can be in-process (fast, memory accumulates) or external subprocess (isolated, memory freed after each project).

---

## info Output Flags

Each brick × project cell shows up to 4 flags:

```
s r t x
│ │ │ └─ marked for test execution
│ │ └─── test/ directory present
│ └───── resources/ directory present
└─────── src/ directory present
```

Examples:
- `srx-` = src + resources present, no test dir, not marked for test
- `s--x` = src only, marked for test
- `s-tx` = src + test present, marked for test

Change markers:
- `*` after brick name = changed since stable tag
- `+` = affected by a changed dependency (indirect change)
