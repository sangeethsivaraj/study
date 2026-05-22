# GitHub Actions - Intermediate to Advanced

---

## Chapter 7: Environment Variables and Secrets

### Setting Environment Variables

```yaml
# Workflow-level (available to ALL jobs)
env:
  NODE_ENV: production
  APP_NAME: my-app

jobs:
  build:
    runs-on: ubuntu-latest

    # Job-level (available to all steps in this job)
    env:
      DATABASE_URL: postgres://localhost:5432/test

    steps:
      # Step-level (only this step)
      - name: Build
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: echo "Building $APP_NAME"
```

### GitHub Secrets

Secrets are encrypted values stored in GitHub Settings → Secrets.

```yaml
steps:
  - name: Deploy
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    run: aws s3 sync ./dist s3://my-bucket
```

**Secret types:**
- **Repository secrets** — available to one repo
- **Environment secrets** — available only in specific environments (production, staging)
- **Organization secrets** — shared across repos

### GITHUB_TOKEN (Built-in)

Every workflow gets an automatic token for GitHub API access:

```yaml
steps:
  - name: Comment on PR
    uses: actions/github-script@v7
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      script: |
        github.rest.issues.createComment({
          issue_number: context.issue.number,
          owner: context.repo.owner,
          repo: context.repo.repo,
          body: 'Build passed! ✅'
        })
```

---

## Chapter 8: Job Dependencies and Outputs

### Sequential Jobs (needs)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."

  test:
    needs: build              # Waits for build to finish
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."

  deploy:
    needs: [build, test]      # Waits for BOTH
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

```
┌───────┐     ┌──────┐     ┌────────┐
│ build │────►│ test │────►│ deploy │
└───────┘     └──────┘     └────────┘
```

### Parallel Jobs (no needs)

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    runs-on: ubuntu-latest
    steps: [...]

  security-scan:
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: [lint, test, security-scan]    # Waits for all 3
    runs-on: ubuntu-latest
    steps: [...]
```

```
┌──────┐
│ lint │─────┐
└──────┘     │
┌──────┐     ├────►┌────────┐
│ test │─────┤     │ deploy │
└──────┘     │     └────────┘
┌──────────┐ │
│ security │─┘
└──────────┘
```

### Passing Data Between Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      sha_short: ${{ steps.sha.outputs.sha_short }}
    steps:
      - id: version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT
      - id: sha
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
      - run: echo "Commit: ${{ needs.build.outputs.sha_short }}"
```

---

## Chapter 9: Matrix Strategy (Test Multiple Versions)

### Basic Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]
        os: [ubuntu-latest, windows-latest]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

This creates 6 jobs: Node 16/18/20 × Ubuntu/Windows.

### Matrix with Include/Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20]
    exclude:
      - os: macos-latest
        node: 18                # Skip macOS + Node 18
    include:
      - os: ubuntu-latest
        node: 20
        experimental: true      # Add extra variable to this combo
```

### Fail-Fast Control

```yaml
strategy:
  fail-fast: false    # Don't cancel other matrix jobs if one fails
  matrix:
    node: [16, 18, 20]
```

---

## Chapter 10: Caching (Speed Up Workflows)

### Cache Node Modules

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'          # Built-in caching for npm!
  - run: npm ci
  - run: npm test
```

### Manual Cache Control

```yaml
steps:
  - uses: actions/cache@v4
    id: cache-deps
    with:
      path: node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  - name: Install dependencies
    if: steps.cache-deps.outputs.cache-hit != 'true'
    run: npm ci
```

### Cache for Different Languages

```yaml
# Python (pip)
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

# Go
- uses: actions/cache@v4
  with:
    path: ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

# Maven (Java)
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
```

---

## Chapter 11: Artifacts (Share Files Between Jobs)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build

      # Upload build output
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 5

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # Download the artifact from build job
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      - run: ls dist/    # Files are here!
```

---

## Chapter 12: Environments and Deployment Protection

### Define Environments in GitHub

Settings → Environments → New Environment

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - run: echo "Deploying to staging"

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production               # Requires approval if configured!
      url: https://www.myapp.com
    steps:
      - run: echo "Deploying to production"
```

**Environment features:**
- Required reviewers (approval before deployment)
- Wait timer (delay before deploying)
- Branch restrictions (only main can deploy to production)
- Environment-specific secrets

---

## Chapter 13: Reusable Workflows (DRY)

### Define a Reusable Workflow

**.github/workflows/deploy-template.yml:**
```yaml
name: Deploy Template

on:
  workflow_call:                      # Makes this reusable!
    inputs:
      environment:
        required: true
        type: string
      app-version:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - name: Deploy
        run: |
          echo "Deploying v${{ inputs.app-version }} to ${{ inputs.environment }}"
```

### Call the Reusable Workflow

**.github/workflows/release.yml:**
```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: staging
      app-version: ${{ github.ref_name }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.STAGING_ROLE_ARN }}

  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: production
      app-version: ${{ github.ref_name }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.PRODUCTION_ROLE_ARN }}
```

---

## Chapter 14: Composite Actions (Custom Actions)

### Create Your Own Action

**.github/actions/setup-app/action.yml:**
```yaml
name: 'Setup Application'
description: 'Install dependencies and build the app'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'

outputs:
  build-version:
    description: 'The build version'
    value: ${{ steps.version.outputs.version }}

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - run: npm ci
      shell: bash

    - run: npm run build
      shell: bash

    - id: version
      run: echo "version=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT
      shell: bash
```

### Use It

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-app
        id: setup
        with:
          node-version: '20'
      - run: echo "Built version ${{ steps.setup.outputs.build-version }}"
```

---

## Chapter 15: Conditional Execution

### If Conditions

```yaml
steps:
  - name: Only on main
    if: github.ref == 'refs/heads/main'
    run: echo "On main branch"

  - name: Only on PRs
    if: github.event_name == 'pull_request'
    run: echo "This is a PR"

  - name: Only on tags
    if: startsWith(github.ref, 'refs/tags/')
    run: echo "Tagged release"

  - name: Only if previous step failed
    if: failure()
    run: echo "Something went wrong!"

  - name: Always run (even if job fails)
    if: always()
    run: echo "Cleanup..."

  - name: Only if specific files changed
    if: contains(github.event.head_commit.modified, 'src/')
    run: echo "Source code changed"
```

### Conditional Jobs

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

---

## Chapter 16: Services (Database/Cache for Tests)

Run containers alongside your job (like docker-compose in CI):

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
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
          DATABASE_URL: postgres://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
        run: npm test
```

---

## Chapter 17: Real-World Complete Workflows

### Full CI/CD Pipeline (Node.js)

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
  # ===== LINT & FORMAT CHECK =====
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check

  # ===== UNIT TESTS =====
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test_db
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  # ===== SECURITY SCAN =====
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high

  # ===== BUILD DOCKER IMAGE =====
  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ===== DEPLOY TO STAGING =====
  deploy-staging:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying to staging..."
          # kubectl set image deployment/app app=${{ needs.build.outputs.image-tag }}

  # ===== DEPLOY TO PRODUCTION =====
  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://www.myapp.com
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
```

### Auto-Release on Tag

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate changelog
        id: changelog
        run: |
          PREV_TAG=$(git describe --tags --abbrev=0 HEAD~1 2>/dev/null || echo "")
          if [ -n "$PREV_TAG" ]; then
            CHANGES=$(git log ${PREV_TAG}..HEAD --pretty=format:"- %s (%h)" --no-merges)
          else
            CHANGES=$(git log --pretty=format:"- %s (%h)" --no-merges)
          fi
          echo "changes<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          body: |
            ## Changes
            ${{ steps.changelog.outputs.changes }}
          generate_release_notes: true
```

### Scheduled Dependency Updates Check

```yaml
name: Dependency Check

on:
  schedule:
    - cron: '0 9 * * 1'    # Every Monday at 9 AM

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci

      - name: Check outdated
        id: outdated
        run: |
          OUTPUT=$(npm outdated --json || true)
          echo "result=$OUTPUT" >> $GITHUB_OUTPUT

      - name: Create issue if outdated
        if: steps.outdated.outputs.result != '{}'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: '🔄 Outdated dependencies detected',
              body: 'Run `npm outdated` to see details.',
              labels: ['dependencies']
            })
```

### PR Auto-Labeler

```yaml
name: Label PRs

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  label:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/github-script@v7
        with:
          script: |
            const { data: files } = await github.rest.pulls.listFiles({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
            });

            const labels = new Set();
            for (const file of files) {
              if (file.filename.startsWith('src/')) labels.add('app');
              if (file.filename.startsWith('docs/')) labels.add('documentation');
              if (file.filename.startsWith('test/')) labels.add('tests');
              if (file.filename.includes('Dockerfile')) labels.add('docker');
              if (file.filename.endsWith('.tf')) labels.add('infrastructure');
            }

            if (labels.size > 0) {
              await github.rest.issues.addLabels({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                labels: [...labels],
              });
            }
```

---

## Chapter 18: Security Best Practices

1. **Pin action versions** — `uses: actions/checkout@v4.1.0` (not `@main`)
2. **Use minimum permissions** — `permissions: { contents: read }`
3. **Never echo secrets** — `run: echo ${{ secrets.KEY }}` exposes it in logs
4. **Use environments** for production deployments (approval gates)
5. **Use OIDC** instead of long-lived credentials for cloud access
6. **Audit third-party actions** before using them
7. **Use Dependabot** to keep actions updated

### OIDC Authentication (No Stored Credentials)

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/github-actions
      aws-region: us-east-1
      # No access keys needed! Uses OIDC token exchange
```

---

## Chapter 19: Debugging and Troubleshooting

```yaml
steps:
  # Enable debug logging
  - name: Debug info
    run: |
      echo "Event: ${{ github.event_name }}"
      echo "Ref: ${{ github.ref }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Workspace: ${{ github.workspace }}"

  # Dump full context
  - name: Dump GitHub context
    env:
      GITHUB_CONTEXT: ${{ toJson(github) }}
    run: echo "$GITHUB_CONTEXT"
```

**Enable debug mode:** Set repository secret `ACTIONS_RUNNER_DEBUG` = `true`

---

## Chapter 20: GitHub Actions Cheat Sheet

### Context Variables
| Variable | Value |
|----------|-------|
| `github.ref` | Branch/tag ref (refs/heads/main) |
| `github.sha` | Full commit SHA |
| `github.actor` | User who triggered |
| `github.event_name` | Event type (push, pull_request) |
| `github.repository` | owner/repo |
| `github.workspace` | Working directory path |
| `github.run_number` | Incrementing run number |
| `runner.os` | Linux, Windows, macOS |

### Expression Functions
| Function | Example |
|----------|---------|
| `contains(str, 'sub')` | String/array contains |
| `startsWith(str, 'v')` | String starts with |
| `endsWith(str, '.md')` | String ends with |
| `format('{0}/{1}', a, b)` | String formatting |
| `toJson(value)` | Convert to JSON |
| `fromJson(str)` | Parse JSON |
| `hashFiles('**/lock*')` | Hash file contents |
| `success()` | Previous steps succeeded |
| `failure()` | Any previous step failed |
| `always()` | Run regardless of status |
| `cancelled()` | Workflow was cancelled |

### Most-Used Actions
| Action | Purpose |
|--------|---------|
| `actions/checkout@v4` | Checkout code |
| `actions/setup-node@v4` | Install Node.js |
| `actions/setup-python@v5` | Install Python |
| `actions/cache@v4` | Cache dependencies |
| `actions/upload-artifact@v4` | Upload files |
| `actions/download-artifact@v4` | Download files |
| `actions/github-script@v7` | Run GitHub API scripts |
| `docker/build-push-action@v5` | Build & push Docker |
| `docker/login-action@v3` | Login to registry |
| `aws-actions/configure-aws-credentials@v4` | AWS auth |
| `softprops/action-gh-release@v2` | Create GitHub release |
