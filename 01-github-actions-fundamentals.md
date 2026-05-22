# GitHub Actions - Complete Guide from Scratch to Advanced

---

## Chapter 1: What is GitHub Actions?

### The Problem It Solves

Without automation, every time a developer pushes code, someone must manually:
1. Run tests
2. Check code quality (linting)
3. Build the application
4. Deploy to servers
5. Notify the team

This is slow, error-prone, and doesn't scale.

### What is CI/CD?

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  CI (Continuous Integration)         CD (Continuous Deployment)    │
│  ─────────────────────────           ────────────────────────     │
│                                                                    │
│  Developer pushes code               After CI passes:             │
│       │                                    │                      │
│       ▼                                    ▼                      │
│  ┌──────────┐                        ┌──────────┐                │
│  │Run Tests │                        │  Deploy  │                │
│  └────┬─────┘                        │to Staging│                │
│       │                              └────┬─────┘                │
│       ▼                                   │                      │
│  ┌──────────┐                             ▼                      │
│  │  Lint    │                        ┌──────────┐                │
│  └────┬─────┘                        │  Deploy  │                │
│       │                              │to Prod   │                │
│       ▼                              └──────────┘                │
│  ┌──────────┐                                                    │
│  │  Build   │                                                    │
│  └──────────┘                                                    │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### GitHub Actions is:

- GitHub's built-in CI/CD platform
- Automates workflows triggered by GitHub events (push, PR, schedule, etc.)
- Free for public repos, generous free tier for private repos
- Runs on GitHub-hosted virtual machines (or your own servers)
- Defined in YAML files in your repository

---

## Chapter 2: Core Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│                        WORKFLOW                                   │
│                  (.github/workflows/ci.yml)                       │
│                                                                   │
│  Trigger: on push to main                                        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    JOB: build                             │    │
│  │                  (runs-on: ubuntu-latest)                 │    │
│  │                                                           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │    │
│  │  │  Step 1  │→ │  Step 2  │→ │  Step 3  │→ │ Step 4 │  │    │
│  │  │ Checkout │  │ Install  │  │Run Tests │  │ Build  │  │    │
│  │  │   code   │  │  deps    │  │          │  │        │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              │ depends on (needs)                 │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    JOB: deploy                            │    │
│  │                  (runs-on: ubuntu-latest)                 │    │
│  │                                                           │    │
│  │  ┌──────────┐  ┌──────────┐                             │    │
│  │  │  Step 1  │→ │  Step 2  │                             │    │
│  │  │ Download │  │  Deploy  │                             │    │
│  │  │ artifact │  │ to prod  │                             │    │
│  │  └──────────┘  └──────────┘                             │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Hierarchy

| Concept | What it is | Analogy |
|---------|-----------|---------|
| **Workflow** | A YAML file defining the entire automation | A recipe book |
| **Event/Trigger** | What starts the workflow | "When the oven reaches 350°F..." |
| **Job** | A set of steps that run on one machine | One dish to prepare |
| **Step** | One task within a job | One instruction in the recipe |
| **Action** | A reusable pre-built step | A kitchen appliance |
| **Runner** | The machine that executes a job | The kitchen |

### Where Do Workflow Files Live?

```
my-project/
├── .github/
│   └── workflows/
│       ├── ci.yml           ← Runs tests on every push
│       ├── deploy.yml       ← Deploys on release
│       └── nightly.yml      ← Scheduled nightly build
├── src/
├── package.json
└── README.md
```

---

## Chapter 3: Your First Workflow

### 3.1 The Absolute Simplest Workflow

Create `.github/workflows/hello.yml`:

```yaml
name: Hello World

on: push

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Say Hello
        run: echo "Hello, GitHub Actions!"
```

**Breaking it down:**
- `name:` — Display name in GitHub UI
- `on: push` — Run this workflow on every push to any branch
- `jobs:` — Define one or more jobs
- `greet:` — Job ID (you choose the name)
- `runs-on: ubuntu-latest` — Run on a GitHub-hosted Ubuntu VM
- `steps:` — List of things to do
- `run:` — Execute a shell command

Push this file, then go to your repo → "Actions" tab → see it run!

### 3.2 Real First Workflow: Node.js CI

```yaml
name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      # Step 1: Check out your code
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up Node.js
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      # Step 3: Install dependencies
      - name: Install dependencies
        run: npm ci

      # Step 4: Run linter
      - name: Lint
        run: npm run lint

      # Step 5: Run tests
      - name: Run tests
        run: npm test

      # Step 6: Build
      - name: Build
        run: npm run build
```

**Key things to notice:**
- `uses:` runs a pre-built Action (from GitHub Marketplace)
- `actions/checkout@v4` checks out your repository code
- `actions/setup-node@v4` installs Node.js
- `with:` passes parameters to an Action
- `run:` executes shell commands

---

## Chapter 4: Triggers (Events) — When Workflows Run

### 4.1 Push and Pull Request

```yaml
on:
  push:
    branches: [main, develop]        # Only these branches
    paths:                            # Only when these files change
      - 'src/**'
      - 'package.json'
    paths-ignore:                     # Ignore these paths
      - '**.md'
      - 'docs/**'

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

### 4.2 Schedule (Cron Jobs)

```yaml
on:
  schedule:
    - cron: '0 2 * * *'    # Every day at 2:00 AM UTC
    - cron: '0 */6 * * *'  # Every 6 hours

# Cron syntax: minute hour day-of-month month day-of-week
#              0      2    *             *     *
#              ┬      ┬    ┬             ┬     ┬
#              │      │    │             │     └─ day of week (0-6, Sun=0)
#              │      │    │             └─────── month (1-12)
#              │      │    └───────────────────── day of month (1-31)
#              │      └────────────────────────── hour (0-23)
#              └───────────────────────────────── minute (0-59)
```

### 4.3 Manual Trigger (workflow_dispatch)

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy to which environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Enable debug mode'
        required: false
        type: boolean
        default: false
```

Access inputs in jobs:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ github.event.inputs.environment }}"
```

### 4.4 Other Triggers

```yaml
on:
  # When a release is published
  release:
    types: [published]

  # When an issue is opened
  issues:
    types: [opened, labeled]

  # When a PR is merged (use pull_request with closed + merge check)
  pull_request:
    types: [closed]

  # When called by another workflow
  workflow_call:

  # When a tag is pushed
  push:
    tags:
      - 'v*'    # v1.0.0, v2.1.3, etc.
```

### Complete Trigger Reference

| Event | When |
|-------|------|
| `push` | Code pushed to branch |
| `pull_request` | PR opened, updated, or reopened |
| `schedule` | Cron schedule |
| `workflow_dispatch` | Manual button click |
| `release` | Release created/published |
| `issues` | Issue opened, closed, labeled |
| `workflow_call` | Called by another workflow |
| `workflow_run` | After another workflow completes |
| `repository_dispatch` | External API call |

---

## Chapter 5: Jobs — Parallel and Sequential Execution

### 5.1 Multiple Jobs (Run in Parallel by Default)

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
```

These 3 jobs run simultaneously (faster!).

### 5.2 Sequential Jobs (needs)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  build:
    needs: test              # Wait for test to pass
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

  deploy:
    needs: [test, build]     # Wait for BOTH
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

```
Execution order:
  test ──────► build ──────► deploy
              (waits for test)  (waits for both)
```

### 5.3 Conditional Jobs

```yaml
jobs:
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'    # Only on main branch
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production..."

  notify-failure:
    needs: [test, build]
    if: failure()                           # Only if previous jobs failed
    runs-on: ubuntu-latest
    steps:
      - run: echo "Something failed!"
```

### 5.4 Matrix Strategy (Test Multiple Versions)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

This creates 9 jobs (3 Node versions × 3 OS = 9 combinations), all running in parallel!

```
┌─────────────────────────────────────────────────┐
│  Matrix creates these jobs:                      │
│                                                  │
│  Node 18 + Ubuntu    Node 18 + Windows    ...   │
│  Node 20 + Ubuntu    Node 20 + Windows    ...   │
│  Node 22 + Ubuntu    Node 22 + Windows    ...   │
└─────────────────────────────────────────────────┘
```

**Exclude specific combinations:**
```yaml
strategy:
  matrix:
    node-version: [18, 20]
    os: [ubuntu-latest, windows-latest]
    exclude:
      - node-version: 18
        os: windows-latest
```

**Include extra combinations:**
```yaml
strategy:
  matrix:
    node-version: [18, 20]
    include:
      - node-version: 20
        experimental: true
```

---

## Chapter 6: Steps — Actions vs Shell Commands

### 6.1 Shell Commands (run)

```yaml
steps:
  # Single-line command
  - run: npm test

  # Multi-line commands
  - name: Build and verify
    run: |
      npm run build
      ls -la dist/
      echo "Build size: $(du -sh dist/)"

  # Use a different shell
  - name: PowerShell step
    shell: pwsh
    run: Write-Host "Hello from PowerShell"

  # Set working directory
  - name: Build frontend
    run: npm run build
    working-directory: ./frontend
```

### 6.2 Using Pre-built Actions (uses)

```yaml
steps:
  # From GitHub Marketplace
  - uses: actions/checkout@v4

  # With parameters
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'

  # From another repository
  - uses: owner/repo-name@v1

  # From a local file in your repo
  - uses: ./.github/actions/my-custom-action

  # Specific commit SHA (most secure)
  - uses: actions/checkout@8ade135a41bc03ea155e62e844d188df1ea18608
```

### 6.3 Step Outputs

```yaml
steps:
  - name: Get version
    id: version
    run: echo "tag=$(cat package.json | jq -r .version)" >> $GITHUB_OUTPUT

  - name: Use version
    run: echo "Version is ${{ steps.version.outputs.tag }}"
```

---

## Chapter 7: Environment Variables and Secrets

### 7.1 Environment Variables

```yaml
# Workflow-level env vars
env:
  NODE_ENV: production
  APP_NAME: my-app

jobs:
  build:
    runs-on: ubuntu-latest
    # Job-level env vars
    env:
      DATABASE_URL: postgres://localhost:5432/test

    steps:
      - name: Print env
        # Step-level env vars
        env:
          STEP_VAR: hello
        run: |
          echo "App: $APP_NAME"
          echo "DB: $DATABASE_URL"
          echo "Step: $STEP_VAR"
```

### 7.2 Secrets (Sensitive Data)

**Setting up secrets:**
1. Go to repo → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Add name (e.g., `AWS_ACCESS_KEY_ID`) and value

**Using secrets in workflows:**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to AWS
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws s3 sync ./dist s3://my-bucket

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /app
            git pull
            npm install
            pm2 restart all
```

**Secret scopes:**
| Scope | Available to |
|-------|-------------|
| Repository secrets | All workflows in that repo |
| Environment secrets | Only jobs using that environment |
| Organization secrets | All repos in the org (or selected) |

### 7.3 GitHub Context Variables

```yaml
steps:
  - run: |
      echo "Repo: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run #: ${{ github.run_number }}"
```

---

## Chapter 8: Caching Dependencies (Speed Up Workflows)

### The Problem

Every workflow run installs dependencies from scratch. For a Node.js project with 500 packages, `npm ci` takes 30-60 seconds. Over hundreds of runs, that adds up.

### Solution: Cache

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js with cache
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'           # Automatically caches node_modules!

      - run: npm ci              # Uses cache if package-lock.json unchanged
      - run: npm test
```

### Manual Cache (for more control)

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache node_modules
    uses: actions/cache@v4
    id: cache-deps
    with:
      path: node_modules
      key: deps-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
      restore-keys: |
        deps-${{ runner.os }}-

  - name: Install dependencies
    if: steps.cache-deps.outputs.cache-hit != 'true'
    run: npm ci
```

**How it works:**
1. First run: No cache → installs dependencies → saves cache
2. Second run: Cache found (package-lock unchanged) → skips install → saves minutes!
3. When package-lock changes: Cache miss → fresh install → new cache saved

### Cache for Other Languages

```yaml
# Python
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}

# Go
- uses: actions/cache@v4
  with:
    path: ~/go/pkg/mod
    key: go-${{ runner.os }}-${{ hashFiles('go.sum') }}

# Maven (Java)
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: maven-${{ runner.os }}-${{ hashFiles('pom.xml') }}
```

---

## Chapter 9: Artifacts (Share Data Between Jobs)

### The Problem

Jobs run on separate machines. If the `build` job creates a `dist/` folder, the `deploy` job can't access it directly.

### Solution: Upload and Download Artifacts

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build

      # Upload the build output
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 5      # Auto-delete after 5 days

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # Download the artifact from the build job
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      - name: Deploy
        run: |
          ls dist/
          echo "Deploying build..."
```

### Use Cases for Artifacts
- Pass build output from build job to deploy job
- Save test reports for later review
- Store compiled binaries
- Upload coverage reports

### Upload Test Results

```yaml
- name: Run tests with coverage
  run: npm test -- --coverage

- name: Upload coverage report
  uses: actions/upload-artifact@v4
  if: always()                    # Upload even if tests fail
  with:
    name: coverage-report
    path: coverage/
```

---

## Chapter 10: Environments and Deployment Protection

### What are Environments?

Environments represent deployment targets (staging, production). They can have:
- Secrets specific to that environment
- Protection rules (required reviewers, wait timers)
- Deployment logs

### Setup

1. Go to repo → Settings → Environments
2. Create environments: `staging`, `production`
3. Add protection rules to `production`:
   - Required reviewers: 2 people
   - Wait timer: 5 minutes

### Using Environments in Workflows

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - name: Deploy to staging
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}  # Staging-specific secret
        run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production                    # Requires approval!
      url: https://www.myapp.com
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - name: Deploy to production
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}  # Production-specific secret
        run: ./deploy.sh production
```

**What happens:**
1. `deploy-staging` runs automatically
2. `deploy-production` pauses and waits for manual approval
3. Designated reviewers get a notification
4. After approval → deploys to production

---

## Chapter 11: Reusable Workflows (DRY)

### The Problem

You have 10 microservices, all with the same CI workflow. Copy-pasting 10 files means 10 places to update.

### Solution: Reusable Workflows (workflow_call)

#### Define a reusable workflow:

`.github/workflows/reusable-node-ci.yml`:
```yaml
name: Reusable Node.js CI

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '20'
      working-directory:
        required: false
        type: string
        default: '.'
    secrets:
      NPM_TOKEN:
        required: false

jobs:
  ci:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
          cache-dependency-path: ${{ inputs.working-directory }}/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

#### Call it from another workflow:

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  frontend:
    uses: ./.github/workflows/reusable-node-ci.yml
    with:
      node-version: '20'
      working-directory: './frontend'

  backend:
    uses: ./.github/workflows/reusable-node-ci.yml
    with:
      node-version: '20'
      working-directory: './backend'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

#### Call from a different repository:

```yaml
jobs:
  ci:
    uses: my-org/shared-workflows/.github/workflows/node-ci.yml@main
    with:
      node-version: '20'
```

---

## Chapter 12: Composite Actions (Custom Reusable Steps)

### What are Composite Actions?

A group of steps packaged as a single reusable action. Like creating your own `uses:` action.

### Create a Composite Action

`.github/actions/setup-and-build/action.yml`:
```yaml
name: 'Setup and Build'
description: 'Install dependencies and build the project'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'

outputs:
  build-size:
    description: 'Size of build output'
    value: ${{ steps.size.outputs.size }}

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Build
      shell: bash
      run: npm run build

    - name: Get build size
      id: size
      shell: bash
      run: echo "size=$(du -sh dist/ | cut -f1)" >> $GITHUB_OUTPUT
```

### Use it in workflows:

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Build project
    uses: ./.github/actions/setup-and-build
    with:
      node-version: '20'

  - name: Show size
    run: echo "Build is ${{ steps.build.outputs.build-size }}"
```

---

## Chapter 13: Services (Databases in CI)

### The Problem

Your tests need a real database (not a mock). How do you run PostgreSQL or Redis during CI?

### Solution: Service Containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - name: Run tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: npm test
```

Services start before your steps run and are health-checked. Your app connects to them via `localhost`.

---

## Chapter 14: Conditional Execution (if)

### Step-Level Conditions

```yaml
steps:
  - name: Run only on main
    if: github.ref == 'refs/heads/main'
    run: echo "On main branch"

  - name: Run only on PRs
    if: github.event_name == 'pull_request'
    run: echo "This is a PR"

  - name: Run even if previous step failed
    if: always()
    run: echo "Always runs"

  - name: Run only on failure
    if: failure()
    run: echo "Something failed!"

  - name: Run only on success
    if: success()
    run: echo "Everything passed!"

  - name: Check commit message
    if: contains(github.event.head_commit.message, '[skip ci]')
    run: echo "Skipping..."
```

### Useful Conditions

```yaml
# Only on specific branch
if: github.ref_name == 'main'

# Only on tags
if: startsWith(github.ref, 'refs/tags/')

# Only on specific actor
if: github.actor == 'dependabot[bot]'

# Combine conditions
if: github.ref_name == 'main' && github.event_name == 'push'

# Check if a secret exists
if: secrets.DEPLOY_KEY != ''

# Skip for draft PRs
if: github.event.pull_request.draft == false
```

---

## Chapter 15: Real-World Workflows

### 15.1 Complete CI/CD Pipeline (Node.js)

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─── LINT AND TEST ─────────────────────────────
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
        run: npm run test:ci

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/lcov.info

  # ─── BUILD DOCKER IMAGE ────────────────────────
  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─── DEPLOY TO STAGING ─────────────────────────
  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com

    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to staging"
          # Your deployment command here

  # ─── DEPLOY TO PRODUCTION ──────────────────────
  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://www.myapp.com

    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production"
          # Your deployment command here
```

### 15.2 Auto-Release on Tag

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: |
            dist/*.js
            dist/*.css
```

### 15.3 Deploy to AWS S3 + CloudFront (Static Site)

```yaml
name: Deploy Static Site

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Sync to S3
        run: aws s3 sync dist/ s3://${{ secrets.S3_BUCKET }} --delete

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DIST_ID }} \
            --paths "/*"
```

### 15.4 Deploy to Docker Swarm via SSH

```yaml
name: Deploy to Swarm

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Docker Swarm
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SWARM_MANAGER_IP }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/app
            docker pull myregistry.com/myapp:latest
            docker stack deploy -c docker-stack.yml myapp
```

### 15.5 PR Checks with Comments

```yaml
name: PR Checks

on:
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build

      - name: Get bundle size
        id: size
        run: echo "size=$(du -sh dist/ | cut -f1)" >> $GITHUB_OUTPUT

      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `📦 Build size: **${{ steps.size.outputs.size }}**`
            })
```
