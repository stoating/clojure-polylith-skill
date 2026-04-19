# Polylith Workflows

## Contents
- Create a new component
- Wire a component into a project
- Create a base
- Multiple implementations via profiles
- Incremental testing with stable tags
- Building deployable artifacts

---

## Create a New Component

1. **Create the component:**
   ```bash
   poly create component name:mycomp
   ```

2. **Add to the development project** (root `deps.edn`):
   ```clojure
   {:aliases
    {:dev {:extra-deps {poly/mycomp {:local/root "components/mycomp"}}
           :extra-paths ["components/mycomp/test"]}}}
   ```

3. **Implement the interface** — delegate to impl:
   ```clojure
   ;; components/mycomp/src/se/example/mycomp/interface.clj
   (ns se.example.mycomp.interface
     (:require [se.example.mycomp.core :as core]))

   (defn do-thing [x] (core/do-thing x))
   ```

4. **Add the implementation:**
   ```clojure
   ;; components/mycomp/src/se/example/mycomp/core.clj
   (ns se.example.mycomp.core)

   (defn do-thing [x] ...)
   ```

5. **Write tests:**
   ```clojure
   ;; components/mycomp/test/se/example/mycomp/interface_test.clj
   (ns se.example.mycomp.interface-test
     (:require [clojure.test :refer [deftest is]]
               [se.example.mycomp.interface :as mycomp]))

   (deftest do-thing-test
     (is (= expected (mycomp/do-thing input))))
   ```

6. **Validate:**
   ```bash
   poly check
   poly info
   ```

---

## Wire a Component into a Project

1. **Add brick to project `deps.edn`:**
   ```clojure
   ;; projects/myproject/deps.edn
   {:deps {poly/mycomp {:local/root "../../components/mycomp"}
           poly/mybase {:local/root "../../bases/mybase"}
           org.clojure/clojure {:mvn/version "1.12.0"}}
    :aliases {:test {:extra-paths ["../../components/mycomp/test"
                                   "../../bases/mybase/test"]}}}
   ```
   Note: test paths must be listed manually — they are not auto-included from `:local/root`.

2. **Require the component interface** from the base or another component:
   ```clojure
   ;; bases/mybase/src/se/example/mybase/core.clj
   (ns se.example.mybase.core
     (:require [se.example.mycomp.interface :as mycomp]))  ; ← interface only, never impl

   (defn handle [req]
     (mycomp/do-thing (:body req)))
   ```

3. **Run project tests:**
   ```bash
   poly test project:myproject
   ```

---

## Create a Base

```bash
poly create base name:rest-api
```

Add to development project:
```clojure
;; root deps.edn
{:aliases
 {:dev {:extra-deps {poly/rest-api {:local/root "bases/rest-api"}}
        :extra-paths ["bases/rest-api/test"]}}}
```

Bases have no interface — they are the outermost shell. They require components via interfaces and expose the entry point (e.g. `-main`, ring handler, lambda handler).

---

## Multiple Implementations via Profiles

Use case: `user` component in development (in-memory), `user-remote` in production (calls a service). Both implement `se.example.user.interface`.

1. **Create both components** with the same interface:
   ```bash
   poly create component name:user
   poly create component name:user-remote interface:user
   ```

2. **Define profiles in root `deps.edn`:**
   ```clojure
   {:aliases
    {:dev {...}
     :+default {:extra-deps {poly/user {:local/root "components/user"}}
                :extra-paths ["components/user/src"]}
     :+remote  {:extra-deps {poly/user-remote {:local/root "components/user-remote"}}
                :extra-paths ["components/user-remote/src"]}}}
   ```
   Profile aliases must start with `:+`.

3. **Activate profiles** (in poly shell or appended to commands):
   ```bash
   poly info             # uses :+default automatically
   poly info +remote     # activates :+remote profile
   poly info +           # deactivate all profiles
   ```

4. **Check active profiles in dev:**
   ```bash
   poly info   # profile column shows which bricks are active
   ```

5. **In production projects:** Simply include the correct implementation component in the project's `deps.edn`. Different projects can use different implementations without conflict.

---

## Incremental Testing with Stable Tags

Poly determines what to test by comparing the current state to a **stable tag** in git.

### Create a stable tag

```bash
git tag -f stable-lisa   # each developer tags their own baseline
```

Convention: each developer uses their own tag name (`stable-lisa`, `stable-john`). CI uses `stable-ci` or `stable-build-123`.

### Run tests incrementally

```bash
poly test   # only tests bricks changed since stable-lisa
```

### What counts as "changed"

A brick is considered changed if:
- Any of its files changed since the stable tag, OR
- Any brick it depends on (transitively) changed

### CI workflow

```bash
# After a successful CI run:
git tag -f stable-ci
git push origin stable-ci --force
```

### Check what changed

```bash
poly diff             # files changed since stable tag
poly info since:stable  # bricks changed (marked with *)
```

### Override the baseline

```bash
poly test since:HEAD~1    # test against parent commit
poly test since:abc1234   # test against specific SHA
poly test :all            # bypass — test everything
```

---

## Building Deployable Artifacts

Polylith doesn't prescribe a build tool — use `tools.build` or anything that works with `clj`.

### Example uberjar build

```clojure
;; build.clj at workspace root (or per-project)
(ns build
  (:require [clojure.tools.build.api :as b]))

(def basis (b/create-basis {:project "projects/backend/deps.edn"}))
(def class-dir "target/classes")
(def uber-file "target/backend.jar")

(defn uber [_]
  (b/copy-dir {:src-dirs ["src" "resources"] :target-dir class-dir})
  (b/compile-clj {:basis basis :src-dirs ["src"] :class-dir class-dir})
  (b/uber {:class-dir class-dir :uber-file uber-file :basis basis
           :main 'se.example.rest-api.core}))
```

```bash
clojure -T:build uber
```

### Project alias for uberjar

```clojure
;; projects/backend/deps.edn
{:deps {...}
 :aliases {:uberjar {:main se.example.rest-api.core}
           :build   {:deps {io.github.clojure/tools.build {:git/tag "v0.10.5" :git/sha "2a21b7a"}}
                     :ns-default build}}}
```
