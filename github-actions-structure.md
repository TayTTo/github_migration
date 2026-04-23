# GitHub Actions — Folder Structure & Best Practices

## Recommended Structure

```
.github/
├── actions/                        # Custom / composite actions
│   ├── setup-env/
│   │   └── action.yml
│   ├── docker-build/
│   │   └── action.yml
│   └── notify-slack/
│       └── action.yml
│
├── workflows/                      # All workflow files
│   ├── ci.yml                      # Lint, test, build (PR + push)
│   ├── cd.yml                      # Deploy (on merge to main)
│   ├── release.yml                 # Tag → GitHub Release
│   ├── codeql.yml                  # Security scanning
│   │
│   └── reusable/                   # Reusable workflow definitions
│       ├── test.yml
│       ├── build.yml
│       └── deploy.yml
│
└── CODEOWNERS                      # Who reviews workflow changes
```

---

## Rules

### Workflows (`workflows/`)
- One file per concern — `ci.yml`, `cd.yml`, `release.yml`, not one giant file
- Prefix reusable workflows clearly — put them in `workflows/reusable/` so callers are easy to find
- Name files after what they **do**, not when they run (`deploy.yml` not `on-push-main.yml`)

### Actions (`actions/`)
- Each action lives in its **own folder** — the folder name is the action's reference name
- Every action folder must have an `action.yml` (not `action.yaml` — stick to one convention)
- Optional: add a `README.md` inside the action folder documenting inputs/outputs

### Naming conventions
| Thing | Convention | Example |
|---|---|---|
| Workflow files | `kebab-case.yml` | `ci.yml`, `code-review.yml` |
| Action folders | `kebab-case/` | `setup-env/`, `docker-build/` |
| Job IDs | `snake_case` | `run_tests`, `build_image` |
| Step names | Sentence case | `Run unit tests` |
| Secrets | `SCREAMING_SNAKE_CASE` | `NPM_TOKEN`, `AWS_ROLE_ARN` |

---

## action.yml Template

```yaml
# .github/actions/setup-env/action.yml
name: Setup Environment
description: Install Node, restore cache, install deps

inputs:
  node-version:
    description: Node.js version to use
    required: false
    default: "20"

outputs:
  cache-hit:
    description: Whether the cache was restored
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Restore cache
      id: cache
      uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

    - name: Install dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      run: npm ci
      shell: bash
```

---

## Reusable Workflow Template

```yaml
# .github/workflows/reusable/test.yml
name: Reusable — Test

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: "20"
      working-directory:
        required: false
        type: string
        default: "."
    secrets:
      NPM_TOKEN:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@v4

      - uses: ./.github/actions/setup-env
        with:
          node-version: ${{ inputs.node-version }}

      - name: Run tests
        run: npm test
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## Caller Workflow Template

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  test:
    uses: ./.github/workflows/reusable/test.yml
    with:
      node-version: "20"
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

  build:
    needs: test
    uses: ./.github/workflows/reusable/build.yml
```

---

## What Goes Where

| File/Folder | Put here |
|---|---|
| `.github/workflows/*.yml` | Triggered workflows (ci, cd, release, cron) |
| `.github/workflows/reusable/` | `workflow_call` workflows consumed by other workflows |
| `.github/actions/*/action.yml` | Composite actions shared across multiple workflows |
| `scripts/` *(repo root)* | Shell scripts called by steps — keep logic out of YAML |
| `.github/CODEOWNERS` | Require review from platform/DevOps team on any `.github/` change |
