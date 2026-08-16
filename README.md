# rexplx reusable CI workflows

This repository contains small, public, callable GitHub Actions workflows for
repositories owned by `rexplx`. They standardize baseline Node, Python, and
shell checks without accepting arbitrary command strings or secrets.

The repository's own `.github/workflows/validate.yml` caller executes all three
workflows on pushes to `main` and pull requests. Its fixtures cover both Node
dependency-install branches and assert that every requested Node gate completed;
exercise requirements and pyproject-through-auto Python installation; prove the
shell working-directory excludes an intentionally bad out-of-scope script; and
exercise the explicit no-shell-files pass behavior.

## Call a workflow

Caller repositories keep a small workflow file. Pin `@main` during initial
adoption, then prefer a reviewed release tag or commit SHA for stable consumers.

### Node

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  node:
    uses: rexplx/ci-workflows/.github/workflows/node.yml@main
    with:
      node-version: "22"
      working-directory: .
      run-build: true
      run-tests: true
      run-audit: true
      audit-level: high
```

The Node workflow runs `npm ci` when `package-lock.json` or
`npm-shrinkwrap.json` exists and otherwise runs `npm install`. Requested build,
test, and audit gates execute fixed commands and fail if the corresponding
contract is not satisfied; they are never silently skipped. It exposes
`install_mode`, `build_completed`, `test_completed`, and `audit_completed`
outputs so callers can assert that requested gates actually ran.

### Python

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  python:
    uses: rexplx/ci-workflows/.github/workflows/python.yml@main
    with:
      python-version: "3.12"
      working-directory: .
      dependency-source: auto
      install-dependencies: true
      run-tests: true
```

`dependency-source: auto` installs `requirements.txt` when present, otherwise
installs the local `pyproject.toml` project. Use `requirements` or `pyproject`
to require one source explicitly. Dependency installation and pytest have
explicit boolean opt-outs for repositories that provide equivalent required
jobs elsewhere. The `dependency_source` output records the completed install
path for caller-side contract checks.

### Shell

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  shell:
    uses: rexplx/ci-workflows/.github/workflows/shell.yml@main
    with:
      working-directory: .
      no-shell-files: fail
```

The shell workflow runs `bash -n` and ShellCheck over tracked `.sh` files. It
fails when none exist by default; `no-shell-files: pass` is the explicit opt-out
for a caller that intentionally shares the workflow without shell files.

## Scope and boundaries

- These workflows are `workflow_call` building blocks. Caller repositories own
  their `pull_request`, `push`, scheduling, branch-protection, and required-check
  policies.
- Each workflow grants only `contents: read`, consumes no secrets, and validates
  its repository-relative working directory before use.
- Inputs select versions, fixed gates, or documented modes. They never accept a
  shell command, script body, package argument, or pytest argument string.
- Standard command contracts are intentional: Node callers provide `build` and
  `test` package scripts; Python callers make pytest available through their
  declared dependencies; shell callers rely on ShellCheck being present on the
  GitHub-hosted Ubuntu runner.
- The workflows do not publish packages, deploy services, mutate repositories,
  approve fork runs, or enforce organization-wide rulesets.
- Third-party forks remain governed by their upstream repository owners. Submit
  a repository-specific workflow upstream rather than treating this foundation
  as an enforcement mechanism.
