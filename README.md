# pipeline-templates

Reusable GitHub Actions workflows, so a new repository does not start with a
copy-paste of the last one's CI.

```yaml
# .github/workflows/ci.yml in your repository
name: ci

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: janbalitaan/pipeline-templates/.github/workflows/python-ci.yml@main
```

That is a complete configuration. Every input has a default.

## Layout

```
python/
  setup/action.yml       install uv, install Python, sync
  gate/action.yml        lint, format, types, tests, complexity
  build/action.yml       build, validate, check the tag
.github/
  workflows/
    python-ci.yml        reusable workflow, wraps setup + gate
    python-publish.yml   reusable workflow, wraps setup + gate + build
    lint.yml             actionlint over the files above
  dependabot.yml         keeps the pinned actions current
```

That is the whole repository. There is no project here and no example to
copy — these are templates, and how to use them is this README.

The real logic is in `python/` as composite actions, which can be organized
into directories freely. The two files in `.github/workflows/` are thin
wrappers that add the matrix, the jobs and the permissions.

They have to be there: GitHub requires reusable workflows to sit **directly**
in `.github/workflows`, and
[subdirectories are not supported](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).
That is why they carry a `python-` prefix instead of living in a folder.

### Using the pieces directly

A repository that wants steps rather than a whole workflow can call the
composite actions on their own — useful when the standard matrix does not fit:

```yaml
jobs:
  my-own-shape:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: janbalitaan/pipeline-templates/python/setup@main
        with:
          python-version: '3.12'
      - uses: janbalitaan/pipeline-templates/python/gate@main
        with:
          coverage-min: 90
      - run: ./my-extra-step.sh
```

The difference between the two mechanisms is worth knowing: a **reusable
workflow** is called at job level and brings its own jobs, matrix and
permissions. A **composite action** is called at step level and runs inside
your job, so it cannot define a matrix — but it can live in any directory.

## Overriding

Inputs are the override mechanism. Set only what differs:

```yaml
jobs:
  ci:
    uses: janbalitaan/pipeline-templates/.github/workflows/python-ci.yml@main
    with:
      python-versions: '["3.12", "3.13"]'   # narrower matrix
      sync-args: --all-extras              # optional dependencies
      coverage-min: 85                     # add a coverage gate
      run-type-check: false                # project has no type checker yet
```

### `python-ci.yml`

| input | default | what it does |
| --- | --- | --- |
| `python-versions` | `["3.10","3.11","3.12","3.13"]` | JSON array for the test matrix |
| `source-dir` | `src` | where the package lives |
| `test-dir` | `tests` | empty string skips linting tests |
| `sync-args` | `` | extra `uv sync` arguments |
| `run-lint` | `true` | `ruff check` |
| `run-format-check` | `true` | `ruff format --check` |
| `run-type-check` | `true` | `ty check` |
| `max-complexity` | `15` | highest cognitive complexity per function; `0` skips |
| `run-complexity` | `true` | run the complexity check as its own job |
| `coverage-min` | `0` | `0` disables the gate |
| `working-directory` | `.` | where the project lives, for a monorepo |
| `runs-on` | `ubuntu-latest` | runner label |

### `python-publish.yml`

| input | default | what it does |
| --- | --- | --- |
| `environment` | `pypi` | the environment holding the trusted publisher |
| `repository-url` | `` | set to TestPyPI's URL to publish there |
| `run-gate` | `true` | re-run the checks before publishing |
| `verify-tag` | `true` | fail when the tag disagrees with the version |
| `source-dir` | `src` | where the package lives |
| `test-dir` | `tests` | empty string skips linting tests |
| `sync-args` | `` | extra `uv sync` arguments |
| `max-complexity` | `15` | highest cognitive complexity per function; `0` skips |
| `python-version` | `3.12` | interpreter for building |
| `working-directory` | `.` | where the project lives, for a monorepo |
| `runs-on` | `ubuntu-latest` | runner label |

### The composite actions

Called at step level, for a job whose shape the workflows do not fit.

| action | inputs |
| --- | --- |
| `python/setup` | `python-version`, `sync-args`, `cache`, `working-directory` |
| `python/gate` | `source-dir`, `test-dir`, `run-lint`, `run-format-check`, `run-type-check`, `run-tests`, `coverage-min`, `max-complexity`, `complexity-dir`, `working-directory` |
| `python/build` | `verify-tag` |

## Extending rather than overriding

An input cannot express a step that does not exist in the template. For that,
call the template and add your own job beside it:

```yaml
jobs:
  ci:
    uses: janbalitaan/pipeline-templates/.github/workflows/python-ci.yml@main

  docs:
    needs: ci          # only if the standard checks passed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make docs
```

`needs: ci` is the composition point. Anything the template does not do
belongs in a job of your own, not in a fork of the template.

## Pinning

`@main` tracks the template, so a fix reaches every repository without
touching any of them — and so does a breaking change. Pin a tag when that
matters:

```yaml
uses: janbalitaan/pipeline-templates/.github/workflows/python-ci.yml@v1
```

The trade is the usual one. `@main` means never updating CI by hand and
occasionally being surprised; a tag means the opposite.

## Publishing to PyPI

```yaml
name: publish

on:
  release:
    types: [published]

jobs:
  publish:
    uses: janbalitaan/pipeline-templates/.github/workflows/python-publish.yml@main
    permissions:
      id-token: write
    with:
      environment: pypi
```

`permissions: id-token: write` has to be granted by the caller — a reusable
workflow cannot grant itself more than the caller has. That token is how
trusted publishing works: GitHub mints a short-lived OIDC token, PyPI
exchanges it for an upload token, and **no API token is stored anywhere**, so
there is none to leak or rotate.

On the PyPI side, add a trusted publisher for the project under
*Publishing*: owner `janbalitaan`, the repository name, workflow filename
`publish.yml`, and environment `pypi`. Then create a GitHub environment of
the same name. A required reviewer on that environment means a release cannot
publish without a human.

Publishing runs on a **published release**, not on a tag. A tag pushed by
mistake can be deleted; a release is a deliberate act.

## How the templates get exercised

By the repositories that call them. There is no project here, so there is
nothing for a pipeline to run against — and a package invented to give one
something to do would be scaffolding testing scaffolding.

What this repository runs for itself is `actionlint`, which reads the
templates the way GitHub will and catches an unknown input, a malformed
expression, or a step referencing a job that does not exist. That is checking
the deliverable is well formed.

What it cannot catch is a template that is valid and wrong — a `gate` step
that runs the wrong command, say. That surfaces in the first repository to
call it. The mitigation is the pin: `@main` means a fix reaches everything
without touching anything, and a tag means the opposite.

```yaml
uses: janbalitaan/pipeline-templates/.github/workflows/python-ci.yml@v1
```

For a template you depend on in anger, pin the tag and move it deliberately.

### Keeping the pinned actions current

The templates pin `actions/checkout@v4`, `astral-sh/setup-uv@v5` and
`pypa/gh-action-pypi-publish@release/v1`. Major tags move, but not forever —
`setup-uv@v5` stops receiving fixes once v6 ships. Dependabot watches them
weekly, which matters more here than in most repositories: everything calling
these templates at `@main` inherits whatever is pinned here, so one stale
action is stale everywhere at once.

### Using the pieces directly

`working-directory` points the actions at a subdirectory, which is what a
monorepo needs:

```yaml
jobs:
  ci:
    uses: janbalitaan/pipeline-templates/.github/workflows/python-ci.yml@main
    with:
      working-directory: packages/my-library
```

## Why these choices

**No artifacts.** The publish workflow builds and uploads in one job.
Splitting it would mean passing the distributions between jobs as an
artifact, which consumes storage on every release and buys nothing — the
upload and download would be seconds apart on the same commit. Anyone
forking this into a private repository pays for that storage, so the default
is not to create it.

**`concurrency` in the caller.** A burst of commits otherwise queues a full
matrix per commit for results nobody reads. It belongs in the caller because
the right group key depends on the repository.

**`fail-fast: false`.** Knowing a failure is specific to one Python version
is most of the diagnosis. Cancelling the other legs throws that away to save
a minute.

**One complexity job, not one per version.** It measures source, so running
it four times reports the same number four times.

**Complexity is measured with [complexipy](https://pypi.org/project/complexipy/),
not a script.** A hand-written checker meant a copy of it in every repository
that wanted the gate — four identical files, each needing the same fix. It
also undercounted: a function the local script scored at 14 measured 16 under
complexipy, so the gate was quietly passing something over the limit.

**The gate runs again before publishing.** Shipping a version that fails its
own tests is worse than shipping late, and CI passing on some earlier commit
is not the same as this commit being sound.

## License

MIT

## Publishing to PyPI: use the actions, not the workflow

PyPI's trusted publishing **cannot name a reusable workflow as the
publisher** ([warehouse#11096](https://github.com/pypi/warehouse/issues/11096)).
It matches on the workflow that performs the upload, so calling
`python-publish.yml` makes the OIDC claim point at this repository instead of
yours, and the upload is refused:

```
invalid-publisher: valid token, but no corresponding publisher
  workflow_ref:     you/your-project/.github/workflows/publish.yml@refs/heads/main
  job_workflow_ref: janbalitaan/pipeline-templates/.github/workflows/python-publish.yml@refs/heads/main
```

No publisher configuration matches that second line.

Spell the job out in your own `publish.yml` and use the composite actions.
They run inside your workflow rather than replacing it, so the claim stays
yours and the reuse is preserved:

```yaml
name: publish

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    environment:
      name: pypi
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: janbalitaan/pipeline-templates/python/setup@main
        with:
          python-version: '3.12'
          sync-args: --all-extras
      - uses: janbalitaan/pipeline-templates/python/gate@main
        with:
          source-dir: src
          test-dir: tests
      - uses: janbalitaan/pipeline-templates/python/build@main
        with:
          verify-tag: 'true'
      - uses: pypa/gh-action-pypi-publish@release/v1
```

Configure the trusted publisher with **your** repository and `publish.yml`,
and an environment name matching the `environment:` above.

CI is unaffected — `python-ci.yml` is a reusable workflow and stays one,
because nothing about CI depends on an OIDC claim.
