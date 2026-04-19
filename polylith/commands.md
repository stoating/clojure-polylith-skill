# Polylith CLI Commands

## Contents
- Information & validation
- Creation
- Dependencies
- Libraries
- Testing
- Shell
- Workspace data

---

## Information & Validation

### `poly check`

Validates workspace integrity. Exits non-zero on errors; warnings don't affect exit code.

```bash
poly check         # validate workspace
poly check :dev    # also validate development project
```

**Key error codes:**

| Code | Meaning |
|------|---------|
| 101 | Illegal namespace dep — required impl instead of interface |
| 102 | Duplicate definition |
| 103 | Missing interface definition (components implementing same interface disagree) |
| 104 | Circular dependency between bricks |
| 105 | Illegal name sharing between a base and a component |
| 106 | Multiple components with same interface in a project |
| 107 | Missing component in project for a required interface |
| 108 | Multiple same-interface components in dev project (use profiles) |
| 109 | Invalid test runner config FQN |
| 110 | Invalid deps.edn syntax |
| 111 | Unreadable namespace (fix syntax or move file to resources/) |
| 112 | Illegal brick dep in brick's deps.edn (put cross-brick deps in project) |

**Key warning codes:**

| Code | Meaning |
|------|---------|
| 201 | Mismatching function signatures across same-interface implementations |
| 202 | Missing path referenced in project |
| 203 | Path exists in both dev and a profile |
| 205 | File namespace doesn't start with `:top-namespace` |
| 207 | Component in project appears unnecessary (suppress via `:necessary` in workspace.edn) |
| 301 | Inconsistent library versions across bricks/projects |

---

### `poly info`

Overview of the workspace: bricks, projects, profiles, change flags.

```bash
poly info                    # standard overview
poly info :loc               # add lines-of-code counts
poly info :dialect           # show per-brick source dialect (j=clj, c=cljc, s=cljs)
poly info since:stable       # show changes since stable tag
poly info project:myproject  # filter to one project
poly info brick:mycomp       # filter to one brick
```

**Output flags per brick per project:**

| Flag | Meaning |
|------|---------|
| `s` | `src/` directory present |
| `r` | `resources/` directory present |
| `t` | `test/` directory present |
| `x` | Marked for test execution |

`*` = changed since stable tag. `+` = affected by a changed dependency.

---

### `poly diff`

Shows files changed since the last stable point in time (determined by git tags).

```bash
poly diff                      # since latest stable tag
poly diff since:release        # since latest release tag (^v[0-9].*)
poly diff since:previous-release
poly diff since:HEAD
poly diff since:HEAD~1
poly diff since:abc1234        # specific git SHA
poly diff since:my-tag-key     # custom key defined in workspace.edn :tag-patterns
```

Default stable tag pattern: `^stable-.*`
Default release tag pattern: `^v[0-9].*`

---

## Creation

```bash
# New workspace
poly create workspace name:NAME top-ns:TOP-NS [:commit] [branch:BRANCH] [dialects:clj:cljs]

# New component
poly create component name:NAME [interface:IFACE] [dialect:clj|cljs] [:git-add]

# New base
poly create base name:NAME [dialect:clj|cljs] [:git-add]

# New project
poly create project name:NAME [:git-add]
```

Notes:
- `interface:IFACE` — share an existing interface with another component (different implementation)
- `dialect` — defaults to the first dialect in workspace.edn `:dialects`
- `:git-add` — manually stage created files (use when `:vcs :auto-add` is false)

---

## Dependencies

### Workspace-level (interface view)

```bash
poly deps              # brick → interface matrix
poly deps :swap-axes   # flip rows/columns
```

In the matrix: `x` = src dep, `t` = test-only, `+` = indirect, `-` = indirect test.

### Brick-level

```bash
poly deps brick:mycomp              # deps for one brick
poly deps brick:mycomp out:deps.edn # export to file
```

### Project-level (brick view, more concrete than interface view)

```bash
poly deps project:myproject
poly deps project:myproject brick:mycomp
poly deps project:myproject out:deps.edn
```

---

## Libraries

```bash
poly libs              # all libraries with version matrix
poly libs :compact     # compact output
poly libs :outdated    # show available upgrades
poly libs :update      # update all deps.edn / package.json files

# Filter / target specific libraries
poly libs libraries:metosin/malli
poly libs libraries:metosin/malli:zprint/zprint   # update specific ones
```

Libraries include Maven, local `:local/root`, git, and npm deps.
Respects `:keep-lib-versions` config per brick or project in workspace.edn.

---

## Testing

```bash
poly test                    # changed bricks only (since stable tag), deployable projects
poly test :all               # all brick tests + all project tests
poly test :all-bricks        # all brick tests only
poly test :dev               # include development project tests
poly test :project           # changed bricks + project-specific tests
poly test project:myproject  # tests for specific project only
poly test brick:mycomp       # tests for specific brick only
```

Poly creates an isolated classloader per project — tests run in project context, not just dev.

`since:SINCE` can be appended to any test command to change the baseline:
```bash
poly test since:HEAD~1
```

---

## Shell

```bash
poly shell          # interactive shell with tab-completion
poly shell :tap     # auto-open Portal window for tap> output
poly shell :all     # suggest all args including rarely used ones
```

Inside the shell, all `poly` subcommands work without the `poly` prefix.
Switch workspace: `switch-ws dir:/path/to/other-ws` or `switch-ws via:shortcut-name` (from `~/.config/polylith/config.edn`).

---

## Workspace Data

The entire workspace model is queryable as EDN.

```bash
poly ws                                         # dump full workspace
poly ws get:keys                                # top-level keys
poly ws get:components:count                    # count components
poly ws get:components:mycomp:lines-of-code     # navigate nested path
poly ws get:settings:top-namespace              # read a setting
poly ws get:libraries:org.clojure/clojure       # library info
poly ws out:workspace.edn                       # export to file
```

Top-level keys: `:bases`, `:components`, `:projects`, `:interfaces`, `:libraries`, `:profiles`, `:changes`, `:messages`, `:settings`, `:configs`, `:paths`, `:ws-dir`.

Can also be consumed programmatically via the `polylith/clj-poly` library.
