# AZ-400 — Implement CI with Azure Pipelines and GitHub Actions

> **Exam Weight:** "Design and implement build and release pipelines" = **50–55%** (largest domain)
> **MS Learn Path:** [AZ-400: Implement CI with Azure Pipelines and GitHub Actions](https://learn.microsoft.com/en-us/training/paths/az-400-implement-ci-azure-pipelines-github-actions/)

---

## Table of Contents
1. [Explore Azure Pipelines](#1-explore-azure-pipelines)
2. [Manage Agents and Pools](#2-manage-agents-and-pools)
3. [Pipelines and Concurrency](#3-pipelines-and-concurrency)
4. [Design and Implement a Pipeline Strategy](#4-design-and-implement-a-pipeline-strategy)
5. [Integrate with Azure Pipelines — YAML Deep Dive](#5-integrate-with-azure-pipelines--yaml-deep-dive)
6. [Introduction to GitHub Actions](#6-introduction-to-github-actions)
7. [Continuous Integration with GitHub Actions](#7-continuous-integration-with-github-actions)
8. [Container Build Strategy](#8-container-build-strategy)
9. [Additional Exam Topics (Beyond MS Learn)](#9-additional-exam-topics-beyond-ms-learn)
10. [Past Exam Scenario Bank](#10-past-exam-scenario-bank)

---

## 1. Explore Azure Pipelines

### What is Azure Pipelines?
- A cloud service to **automatically build, test, and deploy** code to any target.
- Works with any language, platform, and cloud.
- Part of Azure DevOps (also integrates with GitHub repositories).
- Implements **Continuous Integration (CI)** and **Continuous Delivery (CD)**.

### Key Terms (Exam Favorite)
| Term | Definition |
|---|---|
| **Pipeline** | The automated workflow definition; a series of stages/jobs/steps |
| **Stage** | A logical boundary in the pipeline (e.g., Build, Test, Deploy). Stages run sequentially by default |
| **Job** | A collection of steps that run on a single agent. A stage has one or more jobs |
| **Step** | The smallest unit of work — a task or script within a job |
| **Task** | A pre-built, reusable step (e.g., `DotNetCoreCLI@2`, `PublishTestResults@2`) |
| **Agent** | A machine (virtual or physical) that executes the pipeline job |
| **Pool** | A collection of agents |
| **Artifact** | Files produced by a build, passed between stages or used for deployment |
| **Trigger** | Event that starts the pipeline (push, PR, schedule, manual) |
| **Run** | A single execution of a pipeline |
| **Environment** | A logical deployment target (Dev, Test, Prod) with approval gates |

### Pipelines in DevOps
- **CI (Continuous Integration):** Every commit triggers an automated build + test.
- **CD (Continuous Delivery):** Code is always in a releasable state; deployment is triggered automatically or manually.
- **CD (Continuous Deployment):** Every passing build is automatically deployed to production.

> **Exam tip:** Know the difference between Continuous Delivery (human approval for prod) and Continuous Deployment (fully automated, no approval).

### Source Control Support in Azure Pipelines
Azure Pipelines can connect to:
- **Azure Repos Git** (native)
- **GitHub** (via OAuth, PAT, or GitHub App)
- **GitHub Enterprise Server**
- **Bitbucket Cloud**
- **TFVC** (Team Foundation Version Control)

---

## 2. Manage Agents and Pools

### Agent Types (Exam Favorite)
| Feature | Microsoft-Hosted | Self-Hosted |
|---|---|---|
| **Managed by** | Microsoft | Your team |
| **Cost** | Free tier (1 parallel job), then paid | Free (you pay for infrastructure) |
| **Pre-installed tools** | Yes (common tools/SDKs) | Only what you install |
| **Customization** | None — fresh VM each run | Full control |
| **Persistent state** | No — reset after each job | Yes — state persists between runs |
| **Network access** | Public internet only | Can access private networks |
| **Use case** | Standard builds, public repos | Custom tools, private networks, compliance |

> **Exam tip:** If a pipeline needs to access an **on-premises database, internal APIs, or custom software**, use a **self-hosted agent**. If the pipeline needs a **clean environment every run**, use Microsoft-hosted.

> **Exam tip (incremental builds):** "Use incremental builds without purging the environment between pipeline executions" → **self-hosted agent**. Self-hosted agents persist the workspace between runs (no automatic cleanup), enabling incremental builds where only changed files are recompiled. Microsoft-hosted agents provision a **fresh VM per job** — the environment is always purged, making incremental builds impossible. A File Transform task transforms config files and has no bearing on build caching.

### Microsoft-Hosted Agent Images
Common images available:
- `ubuntu-latest` / `ubuntu-22.04`
- `windows-latest` / `windows-2022`
- `macos-latest` / `macos-14`

Legacy images (exam-relevant):
- `vs2017-win2016` — Windows Server 2016 + Visual Studio 2017; full .NET Framework support
- `vs2015-win2012r2` — Windows Server 2012 R2 + Visual Studio 2015; full .NET Framework support
- `win1803` — Windows Server SAC 1803; Server Core only — does **NOT** support full .NET Framework

Each job on a Microsoft-hosted agent gets a **fresh virtual machine** — no carry-over between jobs.

> **Exam tip:** Full .NET Framework apps require **Windows** agents. Linux (ubuntu) and macOS agents cannot build full .NET Framework. Windows SAC releases (`win1803`) are Server Core and lack full .NET Framework. Only Visual Studio Windows images (`vs2017-win2016`, `vs2015-win2012r2`) include the full .NET Framework build toolchain.

### Hosted Pool — Platform Matching (Exam Favorite DRAG DROP)

| Application Type | Required Pool | Reason |
|---|---|---|
| **iOS app** | **Hosted macOS** | Xcode only runs on macOS — no other platform can build iOS apps |
| **IIS web app in Docker** | **Hosted Windows Container** | IIS is Windows-only; Docker on Windows requires Windows containers |
| **Linux containers / open-source** | Hosted Ubuntu | Linux-native toolchain |
| **General Windows/.NET** | Hosted (VS2017) | Standard Windows build VM |
| **On-premises / custom tools** | Default (self-hosted) | Private network access or custom software |

> **Exam tip:** iOS = **macOS** only (Xcode requirement). IIS in Docker = **Windows Container** (IIS cannot run in Linux containers — it is a Windows-only web server). Never use Ubuntu or macOS for IIS workloads.

### Self-Hosted Agent Setup
1. Download the agent package from Azure DevOps: **Project Settings → Agent pools → New agent**.
2. Configure with the organization URL and a **PAT (Personal Access Token)** — this is the required authentication mechanism for agent registration.
3. Run as a service (recommended for production).
4. Agent registers itself to a specified pool.

**Authentication for agent registration:**

| Method | Supported for agent registration? | Notes |
|---|---|---|
| **Personal Access Token (PAT)** | ✅ Yes — **required/recommended** | PAT needs scope: Agent Pools (Read & Manage). Used once at registration. |
| SSH key | ❌ No | SSH is for Git authentication, not agent registration |
| Alternate credentials | ❌ No | Deprecated and removed from Azure DevOps |
| Certificate | ❌ No | Not a supported registration mechanism |

> **Exam tip (tested):** "Register a self-hosted Linux agent — which authentication mechanism?" → **Personal Access Token (PAT)**. SSH keys are for Git, not agent registration. Alternate credentials are deprecated.

### Job Types
| Job Type | Description |
|---|---|
| **Agent job** | Runs on an agent (Microsoft-hosted or self-hosted). Most common type |
| **Agentless job (Server job)** | Runs on the Azure DevOps server itself. Used for approvals, delays, invoking REST APIs |
| **Container job** | Runs inside a Docker container on an agent |
| **Deployment job** | Special job type used with Environments; supports deployment strategies |

### Agent Pools
- **Default pool:** Automatically available in every Azure DevOps organization for self-hosted agents.
- **Azure Pipelines pool:** Contains all Microsoft-hosted agents.
- **Custom pools:** Created by admins; scoped to an organization or a project.
- Pool security: managed via roles — **Reader**, **User** (can use pool in pipelines), **Administrator**.

> **Exam tip:** Agent pool permissions — **User role** is needed to use a pool in a pipeline. **Administrator role** manages the pool itself.

### Agent Demands
- **Demands** ensure a job only runs on agents with specific capabilities.
- Two operators: `exists` (capability is present) and `equals` (capability has a specific value).
- Manually set in YAML:
```yaml
pool:
  name: MyPool
  demands:
    - agent.os -equals Windows_NT
    - npm
```
- Tasks automatically add demands (e.g., a .NET task demands the .NET SDK).

### Agent Communication
- Agents communicate with Azure DevOps via **HTTPS (port 443)** — outbound only.
- No inbound firewall rules needed on the agent machine.
- Agent polls Azure DevOps for jobs (pull model).
- For deployment to on-premises targets: agent on the same network makes the connection outbound.

### Agent Pool Security
- At organization level: **Organization Agent Pool** — shared across all projects.
- At project level: linked pools are project-scoped views of organization pools.
- Grant the **Build Service account** appropriate permissions on pipelines.
- Use **pipeline permissions** on an agent pool to restrict which pipelines can use it.

---

## 3. Pipelines and Concurrency

### Parallel Jobs
- A **parallel job** = one pipeline job running at a time on one agent.
- If you need 3 pipelines to run simultaneously, you need 3 parallel jobs.
- **Free tier:** 1 Microsoft-hosted parallel job (limited minutes/month), 1 self-hosted parallel job (unlimited minutes).
- Additional parallel jobs must be purchased.

### Estimating Parallel Jobs
Formula: `Number of parallel jobs needed = Peak concurrent jobs`
- Monitor queue wait times. Long waits = insufficient parallel jobs.
- For open-source (public) projects: **10 free Microsoft-hosted parallel jobs**.
- For private projects: 1 free + purchase additional.

> **Exam tip:** "Builds are queuing and not running concurrently" → solution is to **purchase more parallel jobs**.
>
> **Exam tip (tested — reduce build pipeline duration):** A pipeline builds 10 architectures sequentially and takes ~1 day. To reduce execution time:
> 1. **Create an agent pool** (with enough agents to run jobs concurrently)
> 2. **Increase the number of parallel jobs** (purchase additional parallel jobs so multiple architecture builds run simultaneously)
>
> Each architecture build is a separate job — with only 1 parallel job, they run one after another. With 10 parallel jobs and 10 agents, all 10 run simultaneously. Blue/green deployment (not about build speed), deployment groups (VM deployment targets, not build agents), and reducing repo size (minimal impact on a compute-bound build) do NOT reduce build duration.

### Identifying Agent Pool Exhaustion

When cycle times increase, the cause may be jobs waiting for an available agent (pool exhaustion). Two ways to diagnose this:

| Method | How it works |
|---|---|
| **Pool consumption report** (Organization level) | Built-in Azure DevOps report showing agent pool usage over time — how many agents were in use vs available. Available at: Organization Settings → Agent Pools → select pool → Analytics. Directly shows whether the pool was exhausted. |
| **`TaskAgentPoolSizeSnapshots` REST API endpoint** | Returns historical snapshots of agent pool size and demand. Lets you query whether pool capacity was exceeded during a time window of interest. |

> **What NOT to use for pool exhaustion diagnosis:**
> - **Pipeline duration report** — shows how long pipelines took, but not *why* (doesn't reveal pool wait time vs actual execution time).
> - **`PipelineRun/PipelineRuns` endpoint** — returns pipeline run data (status, duration), not agent pool capacity or wait queue depth.

> **Exam tip (tested):** "Identify whether agent pool exhaustion is causing increased cycle times" → use the **pool consumption report at org level** AND/OR query the **`TaskAgentPoolSizeSnapshots` endpoint**.

### Visual Designer (Classic) vs YAML Pipelines
| Aspect | Classic (Visual Designer) | YAML |
|---|---|---|
| Definition | GUI-based, stored in Azure DevOps | Code-based, stored in source control |
| Version control | No (separate from code) | Yes (pipeline as code) |
| Reusability | Task groups | Templates |
| Auditability | Limited | Full Git history |
| Recommended | Legacy / migration source | **Preferred for all new pipelines** |
| Branch-specific config | Manual | Native (per-branch YAML files) |

> **Exam tip:** YAML is the recommended approach. Classic pipelines are being deprecated. The exam tests migration from Classic → YAML.

### Enable Continuous Integration (CI Trigger)
```yaml
trigger:
  branches:
    include:
      - main
      - feature/*
  paths:
    exclude:
      - docs/*
```
- `trigger: none` — disables CI trigger (manual runs only).
- PR triggers use `pr:` keyword.

### "Batch changes while a build is in progress" Option

In the **Triggers tab** of a classic build pipeline, enabling the CI trigger reveals a checkbox: **"Batch changes while a build is in progress"**.

- This option is part of the **CI trigger** — it does NOT replace enabling CI; it is an option within CI.
- **What it does:** If multiple check-ins happen while a build is already running, they are batched together and a single new build is triggered once the current one finishes — instead of triggering one build per check-in.
- **Effect on goal "run a build automatically on check-in":** ✅ **Yes** — CI is enabled; builds still run automatically on check-in. Batching only changes how concurrent check-ins are handled.

> **Exam tip (tested — series question):** "Selecting Batch changes while a build is in progress — does this ensure a build runs automatically when code is checked in?" → **Yes**. The batch option lives inside the CI trigger settings; selecting it means CI is enabled. It meets the goal of automatic builds on check-in.

**Series — wrong solutions for "run a build automatically on check-in":**

| Proposed solution | Meets goal? | Why |
|---|---|---|
| Enable CI trigger on the **build pipeline** | ✅ Yes | Correct — build runs on every push/check-in |
| Enable "Batch changes" on build pipeline CI trigger | ✅ Yes | CI is still enabled; batching just groups concurrent check-ins |
| From **release pipeline** → Continuous deployment trigger → enable PR trigger | ❌ No | Release pipeline triggers deployments, not builds. PR trigger on release pipeline fires on PR creation, not code check-in |
| From release pipeline → enable Continuous deployment trigger | ❌ No | Triggers a release after a build completes — does not cause a build to run |
| From **release pipeline Pre-deployment conditions** → "Batch changes while a build is in progress" | ❌ No | Wrong location entirely — pre-deployment conditions control when a stage deploys; they have no effect on build triggers |

### Open-Source Projects (Public Projects)
- Public Azure DevOps projects allow anonymous access and get **10 free parallel jobs**.
- Good for open-source communities using Azure Pipelines.

---

## 4. Design and Implement a Pipeline Strategy

### Agent Demands (in Pipeline Design)
- Use demands to route jobs to specific agents (e.g., Windows-only build).
- Capability: a property of the agent (e.g., `java`, `node`, `visualstudio`).
- Demand: a requirement declared in the pipeline that must match agent capability.

### Multi-Configuration Builds (Matrix Strategy)
Run the **same job across multiple configurations** simultaneously:
```yaml
strategy:
  matrix:
    linux:
      imageName: 'ubuntu-latest'
    windows:
      imageName: 'windows-latest'
    mac:
      imageName: 'macos-latest'
  maxParallel: 2
```
- `maxParallel`: limits how many matrix legs run at once.
- Use case: cross-platform testing, multiple Node.js versions, multiple .NET targets.

### Multi-Agent Builds
- Distribute a single build across multiple agents for speed.
- Use `parallel` strategy:
```yaml
strategy:
  parallel: 4
```
- Typically used for large test suites split across agents.

### Multi-Job Builds
- A stage can contain multiple jobs.
- Jobs within a stage run **in parallel by default**.
- Use `dependsOn` to create sequential job dependencies:
```yaml
jobs:
  - job: Build
  - job: Test
    dependsOn: Build
  - job: Deploy
    dependsOn: Test
```

### GitHub Repository Integration with Azure Pipelines
- Connect GitHub repo in Azure Pipelines via:
  1. **GitHub App** (recommended — granular permissions)
  2. **OAuth** (user-level)
  3. **PAT** (Personal Access Token)
- Once connected, the YAML pipeline file can live in the GitHub repo.
- PR triggers can validate code before merge.

**GitHub Checks API — authentication requirement (Exam Favorite):**

| Auth Type | Supports GitHub Checks API? | Notes |
|-----------|---------------------------|-------|
| **GitHub App** | ✅ Yes — **required** | Only auth type with `checks:write` permission; designed for CI/CD integrations on public/open source repos |
| OAuth | No | User-level token; no Checks API access |
| PAT | No | User-scoped; no Checks API access |
| SAML | No | SSO/identity federation — not a pipeline auth mechanism |

> **Exam tip:** When a question mentions **open source project**, **public GitHub repo**, **Azure Pipelines continuous build**, and **GitHub Checks API** — the authentication type is always **GitHub App**. GitHub Apps are the only mechanism that can post check runs and annotations to GitHub PRs/commits.

### Testing Strategy in Pipelines
| Test Type | When to Run | Pipeline Stage |
|---|---|---|
| **Unit tests** | Every commit / CI | Build or Test stage |
| **Integration tests** | After build, against test environment | Test stage |
| **UI/End-to-end tests** | After deployment to test env | Post-deploy Test stage |
| **Load/Performance tests** | Pre-release | Performance stage |
| **Security scans** | Every build | Build/Security stage |

### Code Coverage
- Measure what % of code is exercised by tests.
- Publish to Azure DevOps using `PublishCodeCoverageResults@1` task.
- Supported formats: **Cobertura**, **JaCoCo**.
- View coverage reports in the pipeline run summary.
- Set **coverage thresholds** — fail the build if coverage drops below a minimum.

```yaml
- task: PublishCodeCoverageResults@1
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: '$(System.DefaultWorkingDirectory)/**/coverage.xml'
```

---

## 5. Integrate with Azure Pipelines — YAML Deep Dive

### YAML Pipeline Anatomy
Full structure of a YAML pipeline:
```yaml
name: $(BuildDefinitionName)_$(Date:yyyyMMdd)$(Rev:.r)

trigger:
  branches:
    include: [ main ]

pr:
  branches:
    include: [ main ]

variables:
  buildConfiguration: 'Release'

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo Hello World
          - task: DotNetCoreCLI@2
            inputs:
              command: 'build'
```

### Pipeline Structure Hierarchy
```
Pipeline
 └── Stage(s)
      └── Job(s)
           └── Step(s)  [Task or Script]
```

### Triggers
| Trigger Type | YAML Key | Description |
|---|---|---|
| CI trigger | `trigger:` | Runs on push to specified branches |
| PR trigger | `pr:` | Runs when a PR targets specified branches |
| Scheduled trigger | `schedules:` | Runs on a cron schedule |
| Pipeline trigger | `resources: pipelines:` | Runs when another pipeline completes |
| Manual | `trigger: none` | Disable CI; run manually only |

### Variables
```yaml
variables:
  myVar: 'value'                    # inline variable
  - group: MyVariableGroup          # link a variable group
  - name: myVar
    value: 'value'
```
- **Pipeline variables:** Defined in YAML or UI; available as `$(varName)`.
- **Variable groups:** Shared across pipelines; managed in Library.
- **Secret variables:** Masked in logs; set in UI or Variable Group; NOT available via `env:` automatically — must be mapped.
- **System variables:** Auto-set by Azure DevOps (e.g., `$(Build.BuildId)`, `$(System.DefaultWorkingDirectory)`).

**YAML Pipeline variable scopes — precedence (highest to lowest, exam DRAG DROP):**

| Precedence | Scope | Where defined |
|---|---|---|
| 1 (highest) | **job** | Under a specific job in YAML |
| 2 | **stage** | Under a specific stage in YAML |
| 3 | **pipeline root** | Top-level `variables:` block in YAML |
| 4 (lowest) | **pipeline settings UI** | Variables tab in the Azure DevOps pipeline editor UI |

> **Exam tip:** Four valid scopes — **job, stage, pipeline root, pipeline settings UI**. "Task" is NOT one of the four scopes (tasks consume variables but don't define them in the scoping hierarchy). A variable at a more specific scope (job) overrides the same variable name at a broader scope (pipeline settings UI).

### Key System Variables (Exam Favorite)
| Variable | Value |
|---|---|
| `$(Build.BuildId)` | Unique numeric ID of the build |
| `$(Build.BuildNumber)` | Display name of the build |
| `$(Build.SourceBranch)` | Full branch ref (e.g., `refs/heads/main`) |
| `$(Build.SourceBranchName)` | Short branch name (e.g., `main`) |
| `$(Build.Repository.Name)` | Repository name |
| `$(System.DefaultWorkingDirectory)` | Working directory on the agent |
| `$(Pipeline.Workspace)` | Root directory for all pipeline artifacts |
| `$(Agent.BuildDirectory)` | Directory where build outputs go |

### Enabling Detailed / Debug Logging

To enable detailed (debug-level) logging for a pipeline run, define a pipeline variable:

| Setting | Value |
|---|---|
| **Name** | `System.Debug` |
| **Value** | `true` |

This causes Azure Pipelines to emit verbose diagnostic output for every step — useful for troubleshooting failures. It can be set:
- As a pipeline variable in the YAML (`variables: System.Debug: true`)
- In the pipeline UI at queue time (set it as a runtime variable)
- It is **not** `Debug`, `Log`, or `System.Log` — only `System.Debug` is the recognized variable name.

> **Exam tip (tested — HOTSPOT):** "Enable detailed logging by defining a pipeline variable" → **Name: `System.Debug`, Value: `true`**. The value must be `true` (boolean string), not `1`, `detailed`, or any other value.

### Templates (Exam Favorite)
Templates enable **code reuse** across pipelines. Two types:

**1. Step template** — reuse a set of steps:
```yaml
# templates/build-steps.yml
parameters:
  - name: configuration
    type: string
    default: 'Release'
steps:
  - script: dotnet build --configuration ${{ parameters.configuration }}
```
```yaml
# azure-pipelines.yml
steps:
  - template: templates/build-steps.yml
    parameters:
      configuration: 'Debug'
```

**2. Job template** — reuse a whole job:
```yaml
# templates/test-job.yml
parameters:
  - name: testProject
    type: string
jobs:
  - job: Test
    steps:
      - script: dotnet test ${{ parameters.testProject }}
```

**3. Stage template** — reuse a whole stage.

**4. Pipeline template (extends)** — enforce an org-wide pipeline structure:
```yaml
extends:
  template: templates/org-pipeline.yml@templates
```

> **Exam tip:** Templates stored in a **separate repository** require declaring that repo as a `resource`. Use `extends` templates for enforcing security policies org-wide.

### Pipeline Caching — Cache Task (Exam Favorite)

The `Cache` task saves and restores pipeline dependencies between runs to speed up builds.

**Required inputs:**

| Input | Purpose |
|-------|---------|
| `key` | Unique cache fingerprint — combines identifiers so the cache is invalidated when dependencies change |
| `path` | The directory to cache and restore |

**Yarn caching example:**
```yaml
- task: Cache@2
  inputs:
    key: '"yarn" | "$(Agent.OS)" | yarn.lock'
    path: $(YARN_CACHE_FOLDER)
  displayName: Cache Yarn packages
- script: yarn --frozen-lockfile
```

**Key construction pattern:** `"tool-name" | "$(Agent.OS)" | lockfile`
- Tool name — differentiates caches for different package managers
- `$(Agent.OS)` — separate caches per OS (Windows/Linux/macOS)
- Lock file (`yarn.lock`, `package-lock.json`, `Pipfile.lock`) — cache invalidated when dependencies change

> **Exam tip:** The two required `Cache` task inputs are always **`key`** and **`path`**. `key` = the unique identifier (includes OS + lockfile for proper invalidation). `path` = the directory to cache (e.g. `$(YARN_CACHE_FOLDER)`, `$(npm_config_cache)`).

### YAML Resources
Declare external dependencies for the pipeline:
```yaml
resources:
  repositories:
    - repository: templates        # alias
      type: git
      name: MyProject/MyTemplates  # project/repo
      ref: refs/heads/main
  pipelines:
    - pipeline: upstream           # alias
      source: 'Upstream Pipeline'  # pipeline name
      trigger: true                # trigger this pipeline when upstream completes
  containers:
    - container: mycontainer
      image: ubuntu:20.04
```

### Multi-Repository Pipelines
- Reference files from multiple repos in one pipeline.
- Use `checkout` step to fetch additional repos:
```yaml
steps:
  - checkout: self
  - checkout: templates
```

### Classic Pipeline → YAML Migration
Steps to migrate:
1. Export the classic pipeline definition (view YAML in the classic editor).
2. Create a new YAML file in the repository.
3. Map tasks to their YAML equivalents.
4. Move variables from the classic UI to YAML or Variable Groups.
5. Recreate triggers in YAML syntax.
6. Test the YAML pipeline in a branch before switching over.
7. Disable the classic pipeline.

Key differences to handle:
- Classic uses **task groups** → YAML uses **templates**.
- Classic **release stages** → YAML **stages** with deployment jobs and environments.
- Classic **approvals** are on release stages → YAML approvals are on **Environments**.

---

## 6. Introduction to GitHub Actions

### What are GitHub Actions?
- A CI/CD platform **natively integrated into GitHub**.
- Automates workflows in response to GitHub events.
- Workflows are defined in **YAML files** stored in `.github/workflows/`.

### Core Components
| Component | Description |
|---|---|
| **Workflow** | An automated process defined in a YAML file. Triggered by events |
| **Event** | A specific activity that triggers a workflow (push, PR, schedule, manual) |
| **Job** | A set of steps that run on the same runner. Jobs run in parallel by default |
| **Step** | A single task in a job — either a shell command (`run:`) or an action (`uses:`) |
| **Action** | A reusable unit of code. Can be from GitHub Marketplace, same repo, or a Docker image |
| **Runner** | A server that executes jobs — GitHub-hosted or self-hosted |

### Workflow File Location
- Must be in `.github/workflows/` in the repository root.
- File name is arbitrary (e.g., `ci.yml`, `build.yml`).
- Multiple workflow files = multiple independent workflows.

### Workflow Syntax
```yaml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * 1'          # every Monday at 2am UTC
  workflow_dispatch:              # manual trigger

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: dotnet build
      - name: Test
        run: dotnet test
```

### Events (Exam Favorite)
| Event | Trigger |
|---|---|
| `push` | Code pushed to a branch |
| `pull_request` | PR opened, synchronized, or reopened |
| `schedule` | Cron-based time schedule |
| `workflow_dispatch` | Manual trigger from GitHub UI |
| `workflow_call` | Called by another workflow (reusable workflows) |
| `release` | Release published/created |
| `repository_dispatch` | External HTTP event trigger |
| `issues` | Issue opened, closed, labeled, etc. |

### Runners
| Type | Description |
|---|---|
| **GitHub-hosted runners** | Managed by GitHub; fresh VM per job. Ubuntu, Windows, macOS available |
| **Self-hosted runners** | Your own infrastructure; more control, can access private resources |

- GitHub-hosted runner labels: `ubuntu-latest`, `windows-latest`, `macos-latest`
- Self-hosted runner labels: custom labels you apply when registering the runner.

> **Exam tip:** To run container jobs on a self-hosted runner, **Docker must be installed** on the runner.

### Jobs
- Jobs run **in parallel by default**.
- Use `needs:` to create dependencies (sequential execution):
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]
  test:
    runs-on: ubuntu-latest
    needs: build       # test waits for build to complete
    steps: [...]
  deploy:
    needs: [build, test]
    steps: [...]
```

### Action Types
1. **JavaScript actions:** Run in Node.js on the runner. Fast startup.
2. **Docker container actions:** Run in a Docker container. Consistent environment; requires Linux runner.
3. **Composite actions:** Combine multiple steps into a single reusable action.

### Referencing Action Versions (Exam Favorite)

When using `uses:` to reference an action, you must specify a version after `@`. There are **three valid ways** to identify the version:

| Method | Syntax example | Notes |
|---|---|---|
| **SHA-based hash** | `uses: actions/checkout@abc1234def5678` | Most secure — immutable, cannot be tampered. Recommended for security-sensitive pipelines. |
| **Tag** | `uses: actions/checkout@v4` | Convenient; action author controls what `v4` points to. Tags can be moved/deleted by the owner. |
| **Branch** | `uses: actions/checkout@main` | Always uses the latest commit on that branch — least stable, not recommended for production. |

> **What is NOT a valid version identifier:**
> - **Runner** — the execution environment (`ubuntu-latest`, `windows-latest`); unrelated to action versioning.
> - **Serial number** — no such concept exists in GitHub Actions.

> **Exam tip (tested):** "Identify the specific version of an action" → valid options are **SHA hash**, **tag**, and **branch**. Runner is where the action *runs*, not which *version* it is.

### action.yml (Metadata file)
- **Every custom action must have an `action.yml` (or `action.yaml`) file**.
- Defines: `name`, `description`, `inputs`, `outputs`, `runs` (runtime type and entrypoint).

```yaml
name: 'My Action'
description: 'Does something useful'
inputs:
  my-input:
    description: 'An input parameter'
    required: true
    default: 'default-value'
outputs:
  my-output:
    description: 'The output value'
runs:
  using: 'node20'
  main: 'index.js'
```

---

## 7. Continuous Integration with GitHub Actions

### Environment Variables
- Set at workflow, job, or step level using `env:`.
- More specific scope overrides broader scope.
```yaml
env:
  GLOBAL_VAR: 'workflow-level'    # available to all jobs

jobs:
  build:
    env:
      JOB_VAR: 'job-level'        # available to all steps in this job
    steps:
      - name: Step
        env:
          STEP_VAR: 'step-level'  # available only in this step
        run: echo $GLOBAL_VAR $JOB_VAR $STEP_VAR
```

### Default Environment Variables (Exam Favorite)
| Variable | Value |
|---|---|
| `GITHUB_SHA` | Commit SHA that triggered the workflow |
| `GITHUB_REF` | Branch or tag ref (e.g., `refs/heads/main`) |
| `GITHUB_REPOSITORY` | `owner/repo` (e.g., `microsoft/azure-sdk`) |
| `GITHUB_WORKSPACE` | Working directory on the runner |
| `GITHUB_RUN_ID` | Unique ID for this workflow run |
| `GITHUB_ACTOR` | User who triggered the workflow |
| `GITHUB_TOKEN` | Auto-generated token for the workflow; has repo permissions |

### Sharing Artifacts Between Jobs
- Jobs run on **separate runners** — they don't share filesystem.
- Use **upload/download artifact actions** to pass files between jobs:
```yaml
jobs:
  build:
    steps:
      - run: dotnet publish -o ./publish
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: ./publish

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: ./publish
```

### Workflow Badges
- Add a status badge to your README to show build status.
- Format: `![CI](https://github.com/<owner>/<repo>/actions/workflows/<workflow-file>/badge.svg)`
- Can filter by branch: append `?branch=main`

### Encrypted Secrets
- Secrets are encrypted environment variables stored at the **repository**, **environment**, or **organization** level.
- Set in: Repository → Settings → Secrets and variables → Actions.
- Accessed in workflow with: `${{ secrets.MY_SECRET }}`
- **Never printed in logs** (masked automatically).
- **Secret size limit:** 48 KB. For larger secrets, store encrypted file in repo and save the decryption passphrase as a secret.

```yaml
steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.API_KEY }}
    run: ./deploy.sh
```

### Secret Scopes
| Scope | Availability |
|---|---|
| **Repository secret** | All workflows in that repository |
| **Environment secret** | Only workflows referencing that environment |
| **Organization secret** | All repos in the org (configurable policy) |

> **Exam tip:** Environment secrets are more restricted than repo secrets — workflows must explicitly reference the environment to access them. This is the recommended approach for production credentials.

### GITHUB_TOKEN
- Automatically created for every workflow run.
- Scoped to the **repository** the workflow runs in.
- Permissions: read by default, can be elevated with `permissions:` key.
- Used for: creating releases, commenting on PRs, pushing to the repo.
```yaml
permissions:
  contents: write
  pull-requests: read
```

### Marking Releases with Git Tags
- Create a release when a Git tag is pushed:
```yaml
on:
  push:
    tags:
      - 'v*'        # triggers on any tag starting with v
```
- Use `actions/create-release` or GitHub CLI to create a release.

### CI Best Practices for GitHub Actions
- Keep workflows **focused** — separate CI, CD, and security scanning.
- Use **pinned action versions** (SHA or tag) for security: `uses: actions/checkout@v4`
- Use **reusable workflows** (`workflow_call`) to avoid duplication.
- Store secrets at the **environment level** for production credentials.
- Set **timeout-minutes** on jobs to prevent runaway workflows.
- Use **concurrency groups** to cancel outdated runs:
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Reusable Workflows
- A workflow that can be called by other workflows.
- Defined with `on: workflow_call:`
- Called using `uses: ./.github/workflows/reusable.yml` or `uses: org/repo/.github/workflows/reusable.yml@main`
- Passes inputs and secrets to the called workflow.

---

## 8. Container Build Strategy

### Why Containers for CI/CD?
- **Consistency:** Same environment from dev to production.
- **Isolation:** Dependencies don't conflict between projects.
- **Portability:** Run anywhere Docker is installed.
- **Reproducibility:** Image tag = exact environment snapshot.

### Container Structure
- **Image:** Read-only template used to create containers.
- **Container:** Running instance of an image.
- **Registry:** Storage for images (Docker Hub, Azure Container Registry, GitHub Container Registry).
- **Layer:** Each Dockerfile instruction creates a layer. Layers are cached.

### Dockerfile Core Concepts
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build    # base image
WORKDIR /app                                        # set working dir
COPY . .                                            # copy files
RUN dotnet restore                                  # run a command
RUN dotnet build -c Release -o /app/build

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime  # new stage
WORKDIR /app
COPY --from=build /app/build .                      # copy from build stage
EXPOSE 80                                           # document port
ENTRYPOINT ["dotnet", "MyApp.dll"]                  # run command
```

### Key Dockerfile Instructions
| Instruction | Purpose |
|---|---|
| `FROM` | Specify base image. First instruction in every Dockerfile |
| `WORKDIR` | Set working directory for subsequent instructions |
| `COPY` | Copy files from host to image |
| `ADD` | Like COPY but also extracts tarballs and supports URLs |
| `RUN` | Execute commands during build (creates a new layer) |
| `CMD` | Default command when container starts (overridable at runtime) |
| `ENTRYPOINT` | Main process of the container (harder to override than CMD) |
| `EXPOSE` | Documents which port the container listens on |
| `ENV` | Set environment variables |
| `ARG` | Build-time variables (not persisted in image) |
| `LABEL` | Add metadata to the image |
| `VOLUME` | Create a mount point for persistent data |

### Multi-Stage Dockerfiles (Exam Favorite)
- Multiple `FROM` instructions in one Dockerfile — each is a **stage**.
- **Benefits:**
  - Smaller final image (build tools not included in runtime image).
  - Separate build and runtime concerns.
  - Test artifacts stay in build stage only.
- Pattern: Build SDK image → run tests → copy binaries to lean runtime image.

```dockerfile
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Test (optional)
FROM builder AS tester
RUN npm test

# Stage 3: Production
FROM nginx:alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

> **Exam tip:** Multi-stage builds = **smaller images** (no build tools in final image) and **cleaner CI** (test failures don't block image creation if stages are independent).

### Considerations for Multi-Stage Builds
- Each stage is independent; you can target a specific stage: `docker build --target builder .`
- Use **ARG** for build-time parameters that differ between environments.
- Use **build cache** wisely — put rarely-changing `COPY`+`RUN` early.
- Minimize layers: chain `RUN` commands with `&&`.

### Azure Container Services
| Service | Description | Use Case |
|---|---|---|
| **Azure Container Registry (ACR)** | Private Docker registry in Azure | Store and manage container images |
| **Azure Container Instances (ACI)** | Run containers without managing VMs | Serverless container execution, batch jobs |
| **Azure Kubernetes Service (AKS)** | Managed Kubernetes | Orchestrate large-scale containerized apps |
| **Azure App Service (Web App for Containers)** | PaaS for containerized web apps | Simple containerized web app hosting |
| **Azure Container Apps** | Serverless microservices platform | Event-driven, microservices, scale-to-zero |

### Deploying Docker Containers to Azure App Service
1. Build Docker image.
2. Push to ACR (or Docker Hub).
3. Create App Service with "Docker Container" publish option.
4. Configure the container source (ACR image + tag).
5. Enable **Continuous Deployment** in App Service → Docker Container settings (webhooks auto-deploy on new image push).

### Build and Push to ACR in Azure Pipelines
```yaml
- task: Docker@2
  inputs:
    containerRegistry: 'MyACRServiceConnection'
    repository: 'myapp'
    command: 'buildAndPush'
    Dockerfile: '**/Dockerfile'
    tags: |
      $(Build.BuildId)
      latest
```

### Build and Push to ACR in GitHub Actions
```yaml
- name: Log in to ACR
  uses: azure/docker-login@v1
  with:
    login-server: myregistry.azurecr.io
    username: ${{ secrets.ACR_USERNAME }}
    password: ${{ secrets.ACR_PASSWORD }}

- name: Build and push
  run: |
    docker build -t myregistry.azurecr.io/myapp:${{ github.sha }} .
    docker push myregistry.azurecr.io/myapp:${{ github.sha }}
```

---

## 9. Additional Exam Topics (Beyond MS Learn)

### YAML Pipeline Conditions
Control whether a step/job/stage runs:
```yaml
- script: echo "Only on main"
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')

- script: echo "On success"
  condition: succeeded()

- script: echo "Always runs"
  condition: always()

- script: echo "Even if failed"
  condition: failed()
```

Common condition functions:
- `succeeded()` — previous step/job succeeded (default)
- `failed()` — previous step/job failed
- `always()` — always runs
- `succeededOrFailed()` — runs unless cancelled
- `eq(a, b)`, `ne(a, b)`, `and(a, b)`, `or(a, b)`

### Pipeline Artifacts vs Build Artifacts
| Type | Task | Description |
|---|---|---|
| **Build Artifact** | `PublishBuildArtifacts@1` | Published to Azure DevOps server; accessible via UI |
| **Pipeline Artifact** | `PublishPipelineArtifact@1` | Faster, uses Azure Blob Storage; preferred |

> **Exam tip:** Pipeline artifacts (`PublishPipelineArtifact`) are **faster and cheaper** than build artifacts. New pipelines should use pipeline artifacts.

### Environments and Approvals
- **Environment:** A logical deployment target (Dev, Test, Prod).
- Configure **manual approvals** on an environment:
  - Approvals block the deployment stage until a reviewer approves.
  - Set in: Environments → Select environment → Approvals and Checks.
- **Checks available on environments:**
  - Manual approval
  - Branch control (deploy only from specific branches)
  - Business hours (deploy only during certain hours)
  - Invoke Azure Function / REST API

```yaml
- stage: DeployProd
  jobs:
    - deployment: Deploy
      environment: 'Production'    # triggers approval check
      strategy:
        runOnce:
          deploy:
            steps:
              - script: ./deploy.sh
```

### Deployment Strategies (in Azure Pipelines)
Configured under `deployment` job with `strategy:`:
| Strategy | Description |
|---|---|
| `runOnce` | Deploy once; standard deployment |
| `rolling` | Incrementally update instances in batches |
| `canary` | Deploy to a small subset first, then gradually roll out |

### Pipeline Caching
- Cache dependencies between runs to speed up pipelines.
- `Cache@2` task stores/restores files using a cache key.
```yaml
- task: Cache@2
  inputs:
    key: 'npm | $(Agent.OS) | package-lock.json'
    path: $(npm_config_cache)
    restoreKeys: |
      npm | $(Agent.OS)
```

### Task Groups (Classic) vs Templates (YAML)
| Aspect | Task Groups (Classic) | Templates (YAML) |
|---|---|---|
| Storage | Azure DevOps (not source control) | Source control (repo) |
| Reuse | Across classic pipelines in a project | Across YAML pipelines (any repo) |
| Version control | No | Yes (full Git history) |

### Variable Groups and Azure Key Vault Integration
- Variable groups in the **Library** store shared variables.
- Link a variable group to **Azure Key Vault** to automatically sync secrets:
  - Azure DevOps fetches secrets from Key Vault at pipeline runtime.
  - Secrets appear as pipeline variables (masked).
```yaml
variables:
  - group: MyKeyVaultGroup   # linked to Azure Key Vault
```

### Service Connections
- **Service connections** = credentials/configurations for connecting Azure Pipelines to external services.
- Types: Azure Resource Manager, Docker Registry, GitHub, Kubernetes, SSH, etc.
- Managed under: **Project Settings → Service connections**.
- Use **Workload Identity Federation (WIF)** instead of secrets where possible (modern, no credential expiry).

### GitHub Actions vs Azure Pipelines — Comparison (Exam Favorite)
| Feature | GitHub Actions | Azure Pipelines |
|---|---|---|
| Trigger source | GitHub events | Azure Repos, GitHub, Bitbucket, TFVC |
| Pipeline as code | YAML in `.github/workflows/` | YAML in repo root (or any path) |
| Reuse | Reusable workflows, composite actions | Templates, task groups |
| Secrets | Repository/org/environment secrets | Variable groups, Key Vault, pipeline variables |
| Agents | GitHub-hosted runners, self-hosted | Microsoft-hosted agents, self-hosted |
| Approvals | Environment protection rules | Environment approvals and checks |
| Marketplace | GitHub Marketplace (Actions) | Azure DevOps Marketplace (Extensions) |
| Free tier | 2,000 min/month (private), unlimited (public) | 1 parallel job, 1,800 min/month (private) |

### Runner/Agent Security Best Practices
- **Least privilege:** Give agents/runners only the permissions needed.
- **Ephemeral runners:** Use new VMs per job (Microsoft-hosted default behavior).
- **No secrets in logs:** Never `echo` secret variables.
- **Pin action versions by SHA:** Prevents supply chain attacks.
- **Review third-party actions** before using in pipelines.
- **Self-hosted runner hardening:** Run as a non-privileged user, isolate from production networks.

### GitHub Codespaces (Cloud-Hosted Development Environments)

**GitHub Codespaces** provides cloud-hosted, browser-accessible development environments directly from a GitHub repository.

| Feature | Detail |
|---|---|
| **GitHub integration** | Native — launch a Codespace directly from any GitHub repo |
| **Debugging tools** | Full VS Code experience including debugger, extensions, integrated terminal |
| **Remote / hot-desking** | Cloud-hosted — no local setup; consistent environment everywhere |
| **Browser / tablet / Chromebook** | Runs entirely in the browser — no local installation required |

**When to choose Codespaces over VS Code (desktop):**
- Developers use non-Windows/Mac devices (tablets, Chromebooks)
- Hot-desking environments where machines are shared
- Onboarding new developers quickly without local environment setup
- Remote workers who need consistent, reproducible environments

> **Exam tip:** The "supports browsers, tablets, Chromebooks" requirement is the key differentiator. Only a **cloud-hosted, browser-based** environment satisfies this — GitHub Codespaces. VS Code desktop requires local installation and doesn't run natively on tablets or Chromebooks.

### Azure DevOps User and License Management Automation

For large organizations with rapidly growing teams, Azure DevOps supports automation of most user management tasks via Azure Active Directory (AAD) integration and group rules:

| Task | Automated? | How |
|---|---|---|
| **Adding users** | ✅ Yes | Sync via Azure AD group membership — users added to an AAD group are auto-provisioned in Azure DevOps |
| **Modifying group memberships** | ✅ Yes | Azure AD dynamic group rules automatically add/remove members based on attributes |
| **Assigning entitlements (access levels)** | ✅ Yes | Azure DevOps group rules — assign Basic/Stakeholder access level to all members of an AAD group automatically |
| **Procuring licenses** | ❌ No — manual | Must be purchased through Azure portal or Microsoft commerce; no automation can approve a purchase |

> **Exam tip:** "Which task must be performed manually?" when automating user management = **procuring/buying licenses**. All other tasks (adding users, group memberships, entitlement assignment) can be automated through Azure AD integration and group rules. Only the act of purchasing licenses requires human approval through a commerce system.

---

## 10. Past Exam Scenario Bank

---

**Q1 — Builds queuing, not running concurrently**
> Your team has 5 pipelines that all need to run simultaneously. They are queuing instead of running in parallel. What should you do?

**A: Purchase additional parallel jobs.**
Each concurrent pipeline execution requires one parallel job. Free tier gives 1. Buy the additional 4 needed.

---

**Q2 — Self-hosted vs Microsoft-hosted agent choice**
> Your pipeline needs to access an internal on-premises SQL Server database during integration tests. Which agent type should you use?

**A: Self-hosted agent.**
Microsoft-hosted agents only have public internet access. A self-hosted agent on the same network can reach on-premises resources.

---

**Q3 — Agent demands: syntax**
> You need a pipeline job to run only on agents that have Visual Studio installed. How do you configure this?

**A: Use demands in the pool configuration:**
```yaml
pool:
  name: MyPool
  demands:
    - visualstudio
```
Demands use `exists` (capability present) or `equals` (specific value). Tasks auto-add demands.

---

**Q4 — Classic → YAML: what replaces task groups?**
> You are migrating a classic pipeline to YAML. The classic pipeline uses task groups shared across multiple pipelines. What is the YAML equivalent?

**A: YAML Templates.**
Task groups → YAML step/job/stage templates stored in source control.

---

**Q5 — Matrix strategy: test on multiple OS**
> You need to run tests on Ubuntu, Windows, and macOS simultaneously in the same pipeline. What strategy do you use?

**A: Matrix strategy.**
```yaml
strategy:
  matrix:
    linux:
      imageName: 'ubuntu-latest'
    windows:
      imageName: 'windows-latest'
    mac:
      imageName: 'macos-latest'
```

---

**Q6 — GitHub Actions: secret larger than 48 KB**
> You need to store a certificate file (200 KB) as a secret in GitHub Actions. What should you do?

**A:** Store the **encrypted file in the repository** and save the **decryption passphrase as a GitHub secret**. Decrypt the file in the workflow using the passphrase.

---

**Q7 — GitHub Actions: container job on self-hosted runner fails**
> A container job in GitHub Actions fails on a self-hosted runner with an error about containers not being available. What is the most likely cause?

**A: Docker is not installed on the self-hosted runner.**
Container jobs require Docker to be installed and running on the runner.

---

**Q8 — Multi-stage Dockerfile: smaller image**
> You have a Dockerfile that copies source code, installs build tools, compiles the app, and runs it. The resulting image is very large. How do you reduce the image size?

**A: Use a multi-stage Dockerfile.**
Stage 1: Build with full SDK image. Stage 2: Copy only compiled output to a lean runtime image. Build tools are not included in the final image.

---

**Q9 — YAML pipeline: run step only on main branch**
> You have a pipeline that runs on all branches, but a deployment step should only run when the branch is `main`. How do you configure this?

**A: Use a condition on the step:**
```yaml
- script: ./deploy.sh
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
```

---

**Q10 — Environment approvals in YAML**
> You need to require manual approval before deploying to production in a YAML pipeline. What is the correct approach?

**A:** Create an **Environment** named `Production` in Azure DevOps, configure **Approvals and Checks** on it, then reference that environment in a `deployment` job in the YAML pipeline. The pipeline will pause and wait for the approver.

---

**Q11 — GitHub Actions: artifacts between jobs**
> In a GitHub Actions workflow, a build job produces a binary. A deploy job (separate job) needs that binary. How do you pass it?

**A:** Use `actions/upload-artifact` in the build job and `actions/download-artifact` in the deploy job. Jobs run on separate runners so the filesystem is not shared.

---

**Q12 — GITHUB_TOKEN permissions**
> A GitHub Actions workflow needs to create a release and push a tag to the repository. The workflow keeps failing with a permissions error. What should you do?

**A:** Set the `permissions:` key in the workflow to grant `contents: write`:
```yaml
permissions:
  contents: write
```
`GITHUB_TOKEN` is read-only by default for `contents`.

---

**Q13 — Pipeline artifact vs build artifact**
> You are creating a new YAML pipeline and need to pass compiled files between stages. Should you use `PublishBuildArtifacts@1` or `PublishPipelineArtifact@1`?

**A: `PublishPipelineArtifact@1`** — it is faster and uses Azure Blob Storage. `PublishBuildArtifacts` is the older task; use pipeline artifacts for new pipelines.

---

**Q14 — Variable group with Key Vault**
> You want pipeline secrets to be automatically pulled from Azure Key Vault without hardcoding them. What do you configure?

**A:** Create a **Variable Group** in the Library and **link it to Azure Key Vault**. The pipeline references the variable group, and secrets are fetched from Key Vault at runtime.

---

**Q20 — "Batch changes" CI trigger: does it meet the auto-build goal?**
> You need to ensure that when code is checked in, a build runs automatically. Solution: from the Triggers tab, you selected "Batch changes while a build is in progress." Does this meet the goal?
>
> A. Yes  B. No

**A: A — Yes.**
"Batch changes while a build is in progress" is an option **inside** the CI trigger settings. Selecting it means CI is enabled — builds will run automatically on check-in. The batch option just groups multiple simultaneous check-ins into one build instead of triggering one build per check-in. The goal (automatic build on check-in) is met.

---

**Q19 — Self-hosted agent registration: authentication mechanism**
> You plan to provision a self-hosted Linux agent. Which authentication mechanism should you use to register the self-hosted agent?
>
> A. SSH key  B. Personal access token (PAT)  C. Alternate credentials  D. Certificate

**A: B — Personal access token (PAT).**
When registering a self-hosted agent, the configuration script requires a PAT with **Agent Pools: Read & Manage** scope. The PAT is used once at registration to authenticate the agent to the Azure DevOps organization. SSH keys are for Git authentication (not agent registration). Alternate credentials are deprecated. Certificates are not a supported registration method.

---

**Q18 — Reduce build pipeline duration for multi-architecture builds**
> A build pipeline uses different jobs to compile an application for 10 different architectures. It takes approximately one day to complete. You need to reduce the execution time. What two actions should you perform?
>
> A. Move to a blue/green deployment pattern.
> B. Create an agent pool.
> C. Create a deployment group.
> D. Reduce the size of the repository.
> E. Increase the number of parallel jobs.

**A: B and E — Create an agent pool + Increase the number of parallel jobs.**
Each architecture is a separate job. With only 1 parallel job, all 10 run sequentially. To run them concurrently:
- **Create an agent pool** with enough agents to handle simultaneous jobs (agents are the workers).
- **Increase parallel jobs** so Azure DevOps is licensed to run multiple jobs at the same time.

Both are required together — agents without parallel job licenses still queue; parallel job licenses without agents have nothing to run on.

A (blue/green) = deployment pattern, not build speed. C (deployment group) = VM targets for release, not CI builds. D (repo size) = minimal effect on compute-bound compilation.

---

**Q17 — GitHub Actions: identify a specific action version**
> You have a GitHub repository with workflows. Each action has one or more versions. You need to request a specific version of an action to execute. Which three attributes can you use to identify the version?
>
> A. SHA-based hash  B. Tag  C. Runner  D. Branch  E. Serial number

**A: A, B, D — SHA-based hash, Tag, Branch.**
In the `uses: owner/action@<ref>` syntax:
- **SHA hash** — immutable commit SHA (most secure, recommended for supply chain safety)
- **Tag** — semantic version tag like `v4` (convenient but mutable — author can retarget it)
- **Branch** — branch name like `main` (least stable — always latest commit on that branch)

Runner (C) is the execution environment, not a version reference. Serial number (E) is not a concept in GitHub Actions.

---

**Q16 — Enable detailed logging via pipeline variable**
> You have a CI/CD pipeline in Azure DevOps. You need to enable detailed logging by defining a pipeline variable. How should you configure the variable?
>
> Name options: Debug / Log / System.Debug / System.Log
> Value options: 1 / detailed / true

**A: Name = `System.Debug`, Value = `true`.**
`System.Debug` is the reserved Azure Pipelines variable that enables verbose/debug-level logging for all pipeline steps. The value must be the string `true`. No other name (`Debug`, `Log`, `System.Log`) or value (`1`, `detailed`) is recognized for this purpose.

---

**Q15 — Diagnosing agent pool exhaustion**
> You use Azure Pipelines and notice an increase in cycle times. You need to identify whether agent pool exhaustion is causing the issue. What are two ways to achieve this?
>
> A. View the Pipeline duration report.
> B. Query the PipelineRun/PipelineRuns endpoint.
> C. View the pool consumption report at the organization level.
> D. Query the TaskAgentPoolSizeSnapshots endpoint.

**A: C and D.**
- **C — Pool consumption report (org level):** Built-in Azure DevOps report showing historical agent pool usage vs capacity. Directly reveals if the pool was exhausted.
- **D — TaskAgentPoolSizeSnapshots endpoint:** REST API endpoint returning historical snapshots of pool size and demand — lets you query whether pool capacity was exceeded.

Pipeline duration report (A) shows total run times but cannot distinguish agent wait time from execution time. PipelineRun/PipelineRuns (B) returns run metadata and status, not pool capacity data.

---

**Q21 — Cloud-hosted development environment for browser/tablet/Chromebook**
> You plan to onboard 10 new developers. Requirements: integrates with GitHub, provides integrated debugging tools, supports remote workers and hot-desking, supports browsers/tablets/Chromebooks. What should you recommend?
>
> A. VS Code  B. Xamarin Studio  C. MonoDevelop  D. GitHub Codespaces

**A: D — GitHub Codespaces.**
GitHub Codespaces is a cloud-hosted, browser-accessible development environment that runs directly from a GitHub repository. It provides a full VS Code experience (including debugging) entirely in the browser — satisfying the tablet/Chromebook requirement without local installation. VS Code (A) requires local install. Xamarin Studio (B) is a mobile IDE. MonoDevelop (C) has no browser support.

---

**Q22 — Release pipeline PR trigger does NOT cause a build to run**
> You need to ensure that when code is checked in, a build runs automatically. Solution: from the Continuous deployment trigger settings of the release pipeline, you enable the Pull request trigger. Does this meet the goal?
>
> A. Yes  B. No

**A: B — No.**
The **release pipeline** controls deployments, not builds. Enabling the Pull request trigger on a release pipeline fires a deployment when a PR is created — it does NOT trigger a build to run on code check-in. To ensure a build runs automatically when code is checked in, enable the **CI trigger on the build pipeline**.

---

**Q23 — Incremental builds without purging the environment**
> You have an existing build pipeline in Azure Pipelines. You need to use incremental builds without purging the environment between pipeline executions. What should you use?
> A. A File Transform task  B. A self-hosted agent  C. Microsoft-hosted parallel jobs

**A: B — Self-hosted agent.**
Self-hosted agents run on your own infrastructure and **persist the workspace between pipeline runs** — the environment is not automatically wiped. This enables incremental builds where only changed files need to be recompiled. Microsoft-hosted agents provision a **fresh VM for every job** (environment always purged — incremental builds not possible). A File Transform task transforms config files (XML/JSON) and has no effect on build caching or environment persistence.

---

**Q26.** You are developing an open source solution using a GitHub repository. You create a new public project in Azure DevOps and plan to use Azure Pipelines for continuous build. The solution will use the GitHub Checks API. Which authentication type should you use?
> A. A personal access token  B. SAML  C. GitHub App  D. OAuth

**A: C — GitHub App.**
The GitHub Checks API requires a **GitHub App** — it is the only authentication type with the `checks:write` permission scope. OAuth apps and PATs cannot use the Checks API. SAML is an SSO/identity federation protocol, not a pipeline authentication mechanism. GitHub Apps are specifically designed for CI/CD integrations on public/open source repos.

---

**Q25. (FILL IN THE BLANK)** You have an Azure subscription that contains Azure DevOps build pipelines. You need to implement pipeline caching using the Cache task. Complete the YAML:
```yaml
inputs:
  [Box 1]:  '"yarn" | "$(Agent.OS)" | yarn.lock'
  [Box 2]:  $(YARN_CACHE_FOLDER)
displayName: Cache Yarn packages
- script: yarn --frozen-lockfile
```

**A:**
- **Box 1: `key`** — the unique cache fingerprint combining tool name, OS, and lockfile. Cache is invalidated when `yarn.lock` changes.
- **Box 2: `path`** — the directory to cache and restore (`$(YARN_CACHE_FOLDER)` is the yarn package cache directory).

---

**Q24. (DRAG DROP)** You have an Azure Pipeline. You need to store configuration values as variables. At which four scopes can the variables be defined, and what is the precedence from highest to lowest?

Scopes: task / job / stage / pipeline root / pipeline settings UI

**A (highest to lowest precedence):**
1. job
2. stage
3. pipeline root
4. pipeline settings UI

"task" is NOT one of the four valid variable definition scopes. A more specific scope (job) always overrides a broader scope (pipeline settings UI) for the same variable name.

---

**Q27 — DRAG DROP: Dockerfile for ASP.NET Core app with minimized image size**
> You are creating a container for an ASP.NET Core app. You need to create a Dockerfile to build the image. The solution must ensure that the size of the image is minimized.
> Structure: `FROM [Box1] As build-env` / `COPY . /app/` / `WORKDIR /app` / `RUN [Box2]` / `FROM [Box3]` / `COPY --from=build-env /app/out /app` / `WORKDIR /app` / `ENTRYPOINT ["dotnet", "MvcMovie.dll"]`
> Values: dotnet publish -c Release -o out / dotnet restore / microsoft/dotnet:2.2-aspnetcore-runtime / Microsoft/dotnet:2.2-sdk

**A:**
- **Box 1: `Microsoft/dotnet:2.2-sdk`** — full SDK image needed to compile and publish the app
- **Box 2: `dotnet publish -c Release -o out`** — compiles and publishes to `/app/out`; implicitly runs restore
- **Box 3: `microsoft/dotnet:2.2-aspnetcore-runtime`** — runtime-only image (~300 MB vs ~1.7 GB SDK) for the final stage

`dotnet restore` is NOT used — `dotnet publish` includes restore implicitly. Image size is minimized by using the runtime-only base image in the final stage (multi-stage build pattern).

---

**Q30. (MULTI-SELECT)** You plan to use Microsoft-hosted agents to build container images that host full .NET Framework apps in a YAML pipeline. Which two VM images can you use?

> A. vs2017-win2016  B. ubuntu-16.04  C. win1803  D. macOS-10.13  E. vs2015-win2012r2

**A: A and E.**
Full .NET Framework only runs on Windows — eliminates Ubuntu (B) and macOS (D). `win1803` is Windows Server Semi-Annual Channel (Server Core only) — does NOT include full .NET Framework toolchain. `vs2017-win2016` (Windows Server 2016 + VS 2017) and `vs2015-win2012r2` (Windows Server 2012 R2 + VS 2015) both include the full .NET Framework build tools.

---

**Q28. (HOT AREA)** You have the Azure DevOps pipeline shown in the exhibit (PartsUnlimitedE2E). The pipeline has one Cloud Agent job containing: NuGet restore, Compile Application, Copy Files, Publish Artifact.

> The pipeline has [0 / 1 / 4] job(s).
> The pipeline has [0 / 1 / 4] task(s).

**A:**
- **Jobs: 1** — "Cloud Agent" is a single agent job. The entire pipeline has one job.
- **Tasks: 4** — NuGet restore, Compile Application, Copy Files, Publish Artifact = 4 tasks within that job.

> **Exam tip:** In Azure Pipelines, a **job** is a unit of work that runs on an agent (or agentless). **Tasks** are the individual steps inside a job. Count the agent pool blocks = jobs; count the task entries inside = tasks.

---

**Q29.** You have a GitHub repository that contains multiple versions of an Azure Pipelines template. You plan to deploy multiple pipelines that will use a template stored in the repository. You need to ensure that you use a **fixed version** of the template. What should you use to reference which version of the template repository to use?

> A. the runner  B. the branch  C. the SHA-based hashes  D. the serial

**A: C — SHA-based hashes.** ⚠️ *Dump says B — INCORRECT*
A SHA hash is immutable and always resolves to the same commit. A branch reference moves as new commits are pushed — not a fixed version. Use `ref: <full-sha>` in the repository resource to pin to an exact commit.

> **Exam tip:** "Fixed version of template" = SHA hash (immutable). Branch = moves over time. Tags can also be used but are less reliable than SHAs since tags can be force-moved.

---

**Q31.** Your company has a hybrid cloud between Azure and Azure Stack. The company uses Azure DevOps for CI/CD pipelines. Some applications are built using Erlang and Hack. You need to ensure that Erlang and Hack are supported as part of the build strategy across the hybrid cloud. The solution must minimize management overhead. What should you use to execute the build pipeline?

A. Self-hosted agents on Azure DevTest Labs VMs  B. Self-hosted agents on Azure Stack VMs  C. Self-hosted agents on Hyper-V VMs  D. A Microsoft-hosted agent

**A: B — Self-hosted agents on Azure Stack VMs.**
Microsoft-hosted agents (D) do not include Erlang or Hack — eliminated immediately. Azure Stack is already part of the company's hybrid cloud, so running self-hosted agents on Azure Stack VMs leverages existing infrastructure with no additional platform to manage. DevTest Labs (A) adds management overhead for a purpose-built dev/test provisioning service. Hyper-V VMs (C) are separate on-premises infrastructure outside the Azure/Azure Stack hybrid model — more management overhead.

> **Exam tip:** Custom/niche languages (Erlang, Hack, etc.) always require **self-hosted agents**. Among self-hosted options, choose the platform already in the described infrastructure to minimize management overhead.

---

**Q32. (MULTI-SELECT)** Your company has an on-premises Bitbucket Server used for Git-based source control. The server is protected by a firewall that blocks inbound Internet traffic. You plan to use Azure DevOps to manage the build and release processes. Which two components are required to integrate Azure DevOps and Bitbucket?

> A. an External Git service connection  B. a Microsoft-hosted agent  C. service hooks  D. a self-hosted agent  E. a deployment group

**A: A and D.**
- **External Git service connection (A)** — provides Azure DevOps with the credentials and endpoint to authenticate against and pull source from the on-premises Bitbucket Server.
- **Self-hosted agent (D)** — the firewall blocks inbound traffic, so Microsoft-hosted agents cannot reach the on-premises server. A self-hosted agent inside the network communicates **outbound** to Azure DevOps on port 443 — no inbound rules needed.

Microsoft-hosted agents (B) cannot penetrate the inbound-blocking firewall. Service hooks (C) send notifications from Azure DevOps outward — not for source integration. Deployment groups (E) are for deploying to target servers, not source control integration.

> **Exam tip:** On-premises source control + inbound firewall = always **self-hosted agent** (outbound-only communication). External/third-party Git repos always need a **service connection** for credentials.

---

**Q35 — DRAG DROP: Match build agent pool to application type**
> You are configuring Azure DevOps build pipelines using hosted build agents. Match each application type to the correct pool:
> - An application that runs on iOS
> - An IIS web application that runs in Docker
>
> Pools: Hosted Windows Container / Hosted Ubuntu 1604 / Hosted macOS / Hosted / Default

**A:**
- **iOS app → Hosted macOS** — Xcode (required to build iOS apps) only runs on macOS; no other pool can build iOS.
- **IIS in Docker → Hosted Windows Container** — IIS is a Windows-only web server; Docker containers hosting IIS require Windows containers, which Hosted Windows Container provides.

Hosted Ubuntu and Hosted macOS cannot run IIS. "Hosted" (VS2017 Windows VM) does not support Windows containers. Default is self-hosted — not a Microsoft-hosted option.

---

**Q34 — Azure DevOps user management: which task must be performed manually?**
> You manage build pipelines and deployment pipelines using Azure DevOps. Your company has a team of 500 developers and new members are added continually. You need to automate the management of users and licenses whenever possible. Which task must you perform manually?
>
> A. modifying group memberships  B. procuring licenses  C. adding users  D. assigning entitlements

**A: B — procuring licenses.**
Adding users (C), modifying group memberships (A), and assigning entitlements (D) can all be automated via Azure AD group sync and group rules in Azure DevOps. Procuring licenses (B) requires a human purchase action through the Azure portal or Microsoft commerce system — no automation can approve a license purchase.

---

**Q33. (Yes/No)** You need to ensure that when code is checked in, a build runs automatically. Solution: From the Pre-deployment conditions settings of the release pipeline, you select "Batch changes while a build is in progress." Does this meet the goal?

**A: B — No.**
Pre-deployment conditions are settings on the **release pipeline** that control when a deployment stage triggers — they have no effect on build pipeline triggers. "Batch changes while a build is in progress" is a CI trigger option on the **build pipeline** (not the release pipeline). The correct solution is to enable a **CI trigger on the build pipeline**. See the series table above for all valid/invalid solutions.

---

## Quick Reference: Exam Checklist

### Azure Pipelines Fundamentals
- [ ] Know all key terms: pipeline, stage, job, step, task, agent, pool, artifact, trigger, run, environment
- [ ] Know the difference: CI vs CD vs Continuous Deployment
- [ ] Source control integrations supported by Azure Pipelines

### Agents & Pools
- [ ] Microsoft-hosted vs self-hosted — when to use each
- [ ] Self-hosted agent: accesses private networks, custom tools, persistent state
- [ ] Agent demands: `exists` and `equals` operators; tasks auto-add demands
- [ ] Job types: Agent job, Agentless/Server job, Container job, Deployment job
- [ ] Agent pool security: Reader, User (can use pool), Administrator roles
- [ ] Agent communicates outbound on HTTPS port 443 only

### Concurrency & Parallelism
- [ ] 1 parallel job = 1 concurrent pipeline run
- [ ] Free tier: 1 Microsoft-hosted, 1 self-hosted (unlimited minutes)
- [ ] Open-source public projects: 10 free Microsoft-hosted parallel jobs
- [ ] Long queue times = buy more parallel jobs

### YAML Pipeline
- [ ] Full hierarchy: Pipeline → Stage → Job → Step
- [ ] Trigger types: CI (`trigger:`), PR (`pr:`), schedule (`schedules:`), pipeline (`resources: pipelines:`), manual (`trigger: none`)
- [ ] Matrix strategy for cross-platform/multi-config builds
- [ ] `dependsOn:` for sequential jobs/stages
- [ ] Conditions: `succeeded()`, `failed()`, `always()`, `eq()`, etc.
- [ ] Templates: step, job, stage, extends — stored in source control
- [ ] Classic → YAML: task groups → templates; release stages → deployment jobs + environments

### GitHub Actions
- [ ] Workflow file in `.github/workflows/`
- [ ] Core components: workflow, event, job, step, action, runner
- [ ] Events: `push`, `pull_request`, `schedule`, `workflow_dispatch`, `workflow_call`
- [ ] Jobs parallel by default; `needs:` for sequential
- [ ] Secrets: repo / environment / org scope; 48 KB limit
- [ ] `GITHUB_TOKEN`: auto-created; `contents: write` needed for push/release
- [ ] Artifacts between jobs: `upload-artifact` / `download-artifact`
- [ ] Container jobs on self-hosted runners require Docker installed

### Container Build Strategy
- [ ] Multi-stage Dockerfile = smaller final image (no build tools in runtime)
- [ ] Key Dockerfile instructions: FROM, RUN, COPY, CMD, ENTRYPOINT, EXPOSE, ARG, ENV
- [ ] Azure container services: ACR (registry), ACI (serverless), AKS (K8s), App Service (web app), Container Apps (microservices)
- [ ] Docker task in Azure Pipelines: `Docker@2` for build/push to ACR
- [ ] Container jobs on self-hosted GitHub runners need Docker installed

---

*Sources: [MS Learn CI Path](https://learn.microsoft.com/en-us/training/paths/az-400-implement-ci-azure-pipelines-github-actions/) | [AZ-400 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-400)*
