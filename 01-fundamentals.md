# GitHub Actions - Complete Guide (Scratch to Advanced)

> This guide covers EVERYTHING — fundamentals, intermediate, and advanced concepts with real-world examples.

---

## Chapter 1: What is GitHub Actions?

### The Problem It Solves

Without CI/CD:
1. Developer pushes code
2. Someone manually runs tests
3. Someone manually builds the app
4. Someone manually deploys to server
5. Bugs slip through, deployments are inconsistent, process is slow

### What GitHub Actions Does

GitHub Actions is a **CI/CD (Continuous Integration / Continuous Deployment)** platform built into GitHub. It automatically runs tasks when events happen in your repository.

```
┌──────────────────────────────────────────────────────────────────┐
│                     GitHub Actions Flow                            │
│                                                                    │
│  ┌─────────┐     ┌───────────┐     ┌──────────┐     ┌────────┐ │
│  │  EVENT  │ ──► │  WORKFLOW  │ ──► │   JOBS   │ ──► │ STEPS  │ │
│  │         │     │           │     │          │     │        │ │
│  │ push    │     │ .yml file │     │ Build    │     │ Action │ │
│  │ PR      │     │ in repo   │     │ Test     │     │ Script │ │
│  │ schedule│     │           │     │ Deploy   │     │ Command│ │
│  │ manual  │     │           │     │          │     │        │ │
│  └─────────┘     └───────────┘     └──────────┘     └────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### Real-World Use Cases

| Use Case | What happens |
|----------|-------------|
| CI (Continuous Integration) | Run tests on every push/PR |
| CD (Continuous Deployment) | Auto-deploy when merged to main |
| Code quality | Lint, format check, type check |
| Security | Scan for vulnerabilities, secrets |
| Release | Auto-create releases, changelogs |
| Scheduled tasks | Nightly builds, data sync, cleanup |
| PR automation | Auto-label, assign reviewers, comment |

---

## Chapter 2: Core Concepts (The Building Blocks)

```
┌─────────────────────────────────────────────────────────────┐
│                     WORKFLOW (.yml file)                      │
│                                                              │
│  Trigger: on push to main                                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ JOB 1: "build"              (runs on ubuntu-latest)    │ │
│  │                                                         │ │
│  │   Step 1: Checkout code                                 │ │
│  │   Step 2: Install Node.js                               │ │
│  │   Step 3: Run "npm install"                             │ │
│  │   Step 4: Run "npm run build"                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                         │                                    │
│                         ▼ (depends on)                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ JOB 2: "test"              (runs on ubuntu-latest)     │ │
│  │                                                         │ │
│  │   Step 1: Checkout code                                 │ │
│  │   Step 2: Install Node.js                               │ │
│  │   Step 3: Run "npm test"                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                         │                                    │
│                         ▼ (depends on)                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ JOB 3: "deploy"            (runs on ubuntu-latest)     │ │
│  │                                                         │ │
│  │   Step 1: Deploy to production                          │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Hierarchy

| Concept | What it is | Contains |
|---------|-----------|----------|
| **Workflow** | A YAML file that defines automation | One or more Jobs |
| **Job** | A set of steps that run on one machine | One or more Steps |
| **Step** | A single task (run a command or action) | One action or command |
| **Action** | A reusable, pre-built step | Code from marketplace |
| **Runner** | The machine (VM) that executes the job | Ubuntu, Windows, macOS |

### Where Do Workflow Files Live?

```
your-repo/
├── .github/
│   └── workflows/
│       ├── ci.yml           ← Workflow 1
│       ├── deploy.yml       ← Workflow 2
│       └── nightly.yml      ← Workflow 3
├── src/
├── package.json
└── README.md
```

Every `.yml` file in `.github/workflows/` is a separate workflow.

---

## Chapter 3: Your First Workflow (Hello World)

### Create the file: `.github/workflows/hello.yml`

```yaml
name: Hello World                    # Name shown in GitHub UI

on: push                             # Trigger: run on every push

jobs:
  greet:                             # Job name (any name you want)
    runs-on: ubuntu-latest           # Machine to run on

    steps:
      - name: Say Hello              # Step name (displayed in logs)
        run: echo "Hello, GitHub Actions!"

      - name: Print date
        run: date

      - name: Multi-line command
        run: |
          echo "Current directory:"
          pwd
          echo "Files:"
          ls -la
```

### What Happens

1. You push this file to your repo
2. GitHub detects the workflow
3. GitHub spins up an Ubuntu VM
4. Runs each step in order
5. You see results in the "Actions" tab

---

## Chapter 4: Triggers (Events) — When Workflows Run

### 4.1 Push and Pull Request

```yaml
on:
  push:
    branches: [main, develop]        # Only these branches
  pull_request:
    branches: [main]                 # PRs targeting main
```

### 4.2 Push with Path Filters

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'                     # Only when src/ files change
      - 'package.json'
    paths-ignore:
      - '**.md'                      # Ignore documentation changes
      - 'docs/**'
```

### 4.3 Manual Trigger (workflow_dispatch)

```yaml
on:
  workflow_dispatch:                  # Button in GitHub UI
    inputs:
      environment:
        description: 'Deploy to which environment?'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Enable debug logging?'
        type: boolean
        default: false
```

### 4.4 Scheduled (Cron)

```yaml
on:
  schedule:
    - cron: '0 6 * * *'             # Every day at 6:00 AM UTC
    - cron: '0 */4 * * *'           # Every 4 hours
    # Format: minute hour day-of-month month day-of-week
    #         ┌───── minute (0-59)
    #         │ ┌───── hour (0-23)
    #         │ │ ┌───── day of month (1-31)
    #         │ │ │ ┌───── month (1-12)
    #         │ │ │ │ ┌───── day of week (0-6, Sun=0)
    #         * * * * *
```

### 4.5 On Release

```yaml
on:
  release:
    types: [published]               # When a release is published
```

### 4.6 On Issue/PR Events

```yaml
on:
  issues:
    types: [opened, labeled]
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  pull_request_review:
    types: [submitted]
```

### 4.7 Multiple Triggers

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:                  # Also allow manual runs
```

### All Available Triggers

| Trigger | When |
|---------|------|
| `push` | Code pushed to branch |
| `pull_request` | PR opened/updated |
| `workflow_dispatch` | Manual button click |
| `schedule` | Cron schedule |
| `release` | Release published |
| `issues` | Issue created/updated |
| `workflow_call` | Called by another workflow |
| `repository_dispatch` | External API call |
| `workflow_run` | After another workflow completes |

---

## Chapter 5: Runners (Where Jobs Execute)

### GitHub-Hosted Runners

```yaml
jobs:
  build:
    runs-on: ubuntu-latest       # Linux (most common, cheapest)
    # runs-on: ubuntu-22.04      # Specific version
    # runs-on: windows-latest    # Windows
    # runs-on: macos-latest      # macOS
```

| Runner | Use Case | Cost (free tier) |
|--------|----------|-----------------|
| `ubuntu-latest` | Most CI/CD, Docker, Node, Python, etc. | 2,000 min/month |
| `windows-latest` | .NET, Windows-specific builds | 2,000 min (2x rate) |
| `macos-latest` | iOS/macOS apps, Swift | 2,000 min (10x rate) |

### Self-Hosted Runners (Advanced)

For faster builds, custom hardware, or private network access:

```yaml
jobs:
  build:
    runs-on: self-hosted         # Your own machine
    # runs-on: [self-hosted, linux, x64]  # With labels
```

---

## Chapter 6: Steps — Actions vs Run Commands

### Using Pre-Built Actions (from Marketplace)

```yaml
steps:
  # Checkout your code (almost always needed!)
  - uses: actions/checkout@v4

  # Setup Node.js
  - uses: actions/setup-node@v4
    with:
      node-version: '20'

  # Setup Python
  - uses: actions/setup-python@v5
    with:
      python-version: '3.12'
```

**`uses` syntax:** `owner/repo@version`
- `actions/checkout@v4` → GitHub's official checkout action, version 4
- `@v4` can also be `@v4.1.0` (specific) or `@main` (latest, risky)

### Running Shell Commands

```yaml
steps:
  - name: Install dependencies
    run: npm install

  - name: Multiple commands
    run: |
      echo "Building..."
      npm run build
      echo "Done!"

  - name: Run with specific shell
    run: |
      Write-Host "Hello from PowerShell"
    shell: pwsh
```

### Action Inputs and Outputs

```yaml
steps:
  - uses: actions/setup-node@v4
    with:                          # Inputs to the action
      node-version: '20'
      cache: 'npm'                 # Cache npm dependencies

  - id: my-step                    # Give step an ID
    run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  - name: Use output
    run: echo "Version is ${{ steps.my-step.outputs.version }}"
```
