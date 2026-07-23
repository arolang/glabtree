# glabtree

Mirror a GitLab group's entire subgroup/repository tree onto the local
filesystem and clone every repository over SSH — using ARO's embedded git.

## Usage

```bash
export GITLAB_TOKEN=glpat-xxxxxxxxxxxx        # prompted (hidden) if unset
aro run ./glabtree --group 1234 --url https://gitlab.com
```

| Input | Source | Fallback |
|-------|--------|----------|
| Group id | `--group <id>` (numeric) | required |
| GitLab base URL | `--url <url>` | `$GITLAB_URL`, then prompt |
| Access token | `$GITLAB_TOKEN` | hidden prompt |

The tree is created under the **current working directory**, mirroring each
group's `full_path` exactly as it appears on GitLab. Empty subgroups still
appear as directories.

## What it does

1. Resolves configuration (parameter → environment → prompt).
2. Fetches the root group to learn its full path (typed API response).
3. Recursively walks every subgroup and repository via the GitLab REST API,
   following pagination through the `x-next-page` header.
4. Stores each discovered repository into `repo-repository` (`parallel for
   each` for concurrency), which drives the reactive UI and the checkout.
5. A **repository observer** clones each newly-found repository over SSH with
   ARO's embedded git, then marks it cloned.
6. A second **repository observer** repaints a live square grid on every
   change: `◻︎` for a discovered repo, `◼︎` once it is cloned.

```
  glabtree · mirroring GitLab group tree

  7 / 12 repositories cloned

  ◼︎ ◼︎ ◼︎ ◼︎ ◼︎ ◼︎ ◼︎ ◻︎ ◻︎ ◻︎ ◻︎ ◻︎

  ◻︎ found   ◼︎ cloned
```

Custom `GroupDiscovered` / `RepositoryCloned` events are appended to
`glabtree-audit.log`.

## ARO features demonstrated

- **Command-line parameters** (`--group`, `--url`) and **environment** reads,
  with an interactive **hidden prompt** fallback.
- **Contract-first typed objects**: every object it passes around
  (`GitlabGroup`, `RepoRecord`, `GroupEvent`, `RepoEvent`) is an OpenAPI
  component in `openapi.yaml`, pulled in one validated step with
  `Extract the <x: Schema> from ...` instead of field-by-field.
- **Events & observers**: repository-change events fan out to a guarded
  checkout observer and a render observer; custom domain events feed an audit
  log.
- **User-defined actions** (`Application.MirrorGroup`, …) with **recursion**
  for tree traversal and pagination.
- **Parallelism**: `parallel for each` with a bounded concurrency cap.
- **Embedded git** (`Clone` over SSH), **filesystem** (`Make`), and the
  **reactive template engine** (live in-place `Render`).

## Files

| File | Purpose |
|------|---------|
| `main.aro` | Config actions, recursive traversal actions, `Application-Start`, `GroupDiscovered` handler |
| `sources/observers.aro` | Checkout observer, reactive grid observer, `RepositoryCloned` handler |
| `templates/grid.screen` | Reactive square-grid template |
| `openapi.yaml` | Typed component schemas (no HTTP server; `paths` empty) |
| `aro.yaml` | Application manifest |
| `plan.md` | The prompt that generates this application |

## Requirements

- **aro ≥ 0.11.2** (see `aro.yaml`).
- Clones use **SSH**, so an SSH key with access to the target GitLab must be
  available (agent or `~/.ssh/id_ed25519` / `~/.ssh/id_rsa`).
- Authenticated SSH cloning relies on a credentials callback added to ARO's
  embedded `GitService.clone` (SSH agent / default keys + optional HTTPS
  token). See the note in the workspace root if running against an older
  runtime.
