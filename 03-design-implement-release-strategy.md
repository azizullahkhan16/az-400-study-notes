# AZ-400 — Design and Implement a Release Strategy

> **Exam Weight:** "Design and implement build and release pipelines" = **50–55%** (shared with CI path)
> **MS Learn Path:** [AZ-400: Design and Implement a Release Strategy](https://learn.microsoft.com/en-us/training/paths/az-400-design-implement-release-strategy/)

---

## Table of Contents
1. [Create a Release Pipeline](#1-create-a-release-pipeline)
2. [Release Strategy Recommendations](#2-release-strategy-recommendations)
3. [Configure and Provision Environments](#3-configure-and-provision-environments)
4. [Manage and Modularize Tasks and Templates](#4-manage-and-modularize-tasks-and-templates)
5. [Automate Inspection of Health](#5-automate-inspection-of-health)
6. [Additional Exam Topics (Beyond MS Learn)](#6-additional-exam-topics-beyond-ms-learn)
7. [Past Exam Scenario Bank](#7-past-exam-scenario-bank)

---

## 1. Create a Release Pipeline

### Release Pipeline vs Build Pipeline
| Aspect | Build Pipeline | Release Pipeline |
|---|---|---|
| **Purpose** | Compile, test, produce artifact | Deploy artifact through environments |
| **Input** | Source code | Build artifact |
| **Output** | Build artifact | Deployed application |
| **Trigger** | Code push / PR | Artifact publication / manual / schedule |
| **YAML equiv** | CI stages | CD stages with deployment jobs |

### Key Release Pipeline Concepts
- **Release:** An instance of a release pipeline run — deploys a specific artifact version through stages.
- **Stage:** A logical deployment boundary (Dev → QA → Staging → Prod). Each stage has pre/post conditions.
- **Deployment:** The act of running the release in a stage.
- **Artifact:** The versioned output (binary, container image, package) being deployed.
- **Trigger:** What starts the release (artifact publication, schedule, manual).

### Artifact Sources (Exam Favorite)
A release pipeline can consume artifacts from multiple sources:

| Source Type | Description |
|---|---|
| **Azure Pipelines build** | Artifact from a build pipeline in the same or different project |
| **Azure Artifacts** | NuGet, npm, Maven, Python packages from a feed |
| **GitHub** | Source code tag/branch from GitHub (used for scripted deployments) |
| **TFVC** | Team Foundation Version Control repository |
| **Jenkins** | Artifact from a Jenkins build |
| **Docker Hub / ACR** | Container image (for container deployments) |
| **External build (TFS/other)** | Artifacts from external TFS or other CI systems |

> **Exam tip:** A release pipeline can link **multiple artifact sources**. Each artifact gets an alias used to reference it in tasks (e.g., `$(Build.ArtifactStagingDirectory)`).

### Choosing the Right Artifact Source
- Use **Azure Pipelines build artifact** for code built in Azure DevOps.
- Use **Azure Artifacts** when deploying packages (NuGet, npm).
- Use **GitHub releases/tags** when source is on GitHub and deployment is script-based.
- Use **Docker Hub/ACR** for containerized application deployments.

### Deployment Triggers (3 Types)
1. **Continuous Deployment trigger:** Automatically creates a release when a new artifact version is available (e.g., new build succeeds).
2. **Scheduled trigger:** Releases at a fixed time (e.g., deploy to QA every night at 11pm).
3. **Manual trigger:** A user must explicitly create a release.

> **Exam tip:** Know all 3 trigger types by name. "Deploy automatically after every successful build" → Continuous Deployment trigger.

### Considerations for Deployment to Stages
- Define **pre-deployment conditions**: approvals, gates, artifact filters, branch filters.
- Define **post-deployment conditions**: approvals, gates (run after stage completes).
- Set **deployment queue settings**: deploy only the latest release, deploy all queued releases.
- Use **artifact filters** to deploy to a stage only from specific branches (e.g., only deploy to prod from `main`).

### Build and Release Tasks
- **Tasks** are the individual units of work in a pipeline step.
- Available from: **Azure DevOps Marketplace**, built-in tasks, custom tasks.
- Categories: Build, Test, Deploy, Utility, Package, Tool.

**Common built-in tasks:**
| Task | Purpose |
|---|---|
| `DotNetCoreCLI@2` | Build/test/publish .NET apps |
| `VSBuild@1` | Build with MSBuild |
| `Maven@3` | Build Java with Maven |
| `Docker@2` | Build and push Docker images |
| `AzureWebApp@1` | Deploy to Azure App Service |
| `AzureRmWebAppDeployment@4` | Advanced App Service deployment |
| `SqlAzureDacpacDeployment@1` | Deploy database changes via DACPAC |
| `KubernetesManifest@0` | Deploy to Kubernetes |
| `AzureCLI@2` | Run Azure CLI scripts |
| `PowerShell@2` | Run PowerShell scripts |
| `CopyFiles@2` | Copy files between locations |
| `PublishBuildArtifacts@1` | Publish build artifacts |
| `DownloadBuildArtifacts@0` | Download build artifacts |

### Custom Build and Release Tasks
- Create custom tasks using the **Azure DevOps Extension SDK**.
- Packaged as **VSIX extensions** published to the Azure DevOps Marketplace.
- Written in TypeScript (Node.js) or PowerShell.
- Custom tasks can be shared within an organization or publicly.

### Release Jobs
- A stage in a classic release pipeline can have multiple **jobs**.
- Jobs run on agents (similar to build pipeline jobs).
- **Deployment group jobs** run on physical/virtual machines registered in a deployment group (for on-premises deployments).
- **Agentless jobs** (Server jobs) run logic on the Azure DevOps server (approvals, invoke REST APIs, delay).

### Deployment Groups — Deploying to On-Premises Servers

A **Deployment Group** is the correct mechanism for deploying artifacts to on-premises or VM targets from Azure Pipelines.

**How it works:**
1. Create a Deployment Group in Azure DevOps (Project Settings → Deployment Groups)
2. **Install the Azure Pipelines agent on each on-premises server** by running the registration script on that server
3. In the release pipeline, use a **Deployment Group job** (not a regular agent job) targeting the deployment group
4. Add tasks (e.g., **Download Build Artifacts**, copy files, run scripts) to deploy to those servers

**Why Docker alone does NOT solve on-premises deployment:**
- A Docker build produces a container image — the on-premises server must be running Docker and configured to pull/run that image
- Using Docker without proper agent/deployment group setup still requires the server to be reachable and configured
- Docker is a containerization approach, not a deployment mechanism for arbitrary on-premises servers

> **Exam tip (tested — series question):** "Deploy artifacts to on-premises servers — Solution: deploy a Docker build + Download Build Artifacts task" → **No**. Docker builds are for containerized workloads, not general on-premises artifact deployment. The correct solution is to **create a Deployment Group**, install agents on each on-premises server, and use a Deployment Group job in the release pipeline.

### Database Deployment
- Use **SQL Server Database project (DACPAC)** for schema changes.
- Task: `SqlAzureDacpacDeployment@1` — deploys DACPAC to Azure SQL.
- For on-premises SQL: use deployment groups + the SQL task.
- Database deployments should be **idempotent** — safe to run multiple times.
- Use **pre-deployment scripts** for data migrations, **post-deployment scripts** for seeding/verification.

**DACPAC vs BACPAC — exam distinction:**

| Artifact | Contains | Use Case | Pipeline deployments? |
|---|---|---|---|
| **DACPAC** (Data-tier Application Package) | Schema only (tables, views, stored procs, indexes) | **CI/CD pipeline deployments** — incremental schema updates | ✅ Yes — used by Azure SQL Database Deployment task |
| **BACPAC** (Backup Application Package) | Schema + data | One-time migration, backup, restore to new environment | ❌ No — not used with the deployment task |
| **MDF** | SQL Server primary database file | Raw server file | ❌ No |
| **LDF** | SQL Server transaction log file | Raw server file | ❌ No |

> **Exam tip:** "Azure SQL Database Deployment task in release pipeline" → **DACPAC**. BACPAC is for migration/backup scenarios, not automated pipeline deployments.

---

## 2. Release Strategy Recommendations

### Delivery Cadence
- **Continuous Deployment:** Deploy every passing build automatically.
- **Continuous Delivery:** Every build is release-ready; deploy on demand.
- **Scheduled delivery:** Deploy at a fixed cadence (e.g., weekly releases).
- **Event-driven:** Deploy in response to business events.

### Release Approvals (Exam Favorite)
Approvals are **human gates** — a person must approve before the release proceeds.

**Types:**
- **Pre-deployment approval:** Required before a stage starts. Approver is notified, must approve/reject.
- **Post-deployment approval:** Required after a stage completes (before the next stage begins).
- Both can be set on the same stage.

**Configuration:**
- Set per stage in classic release pipeline: Stage → Pre-deployment conditions → Approvals.
- Approver receives email notification with approve/reject link.
- Can set **approval timeout** — if not approved within X hours/days, release is rejected.
- Can **reassign** approval to another user.
- Can require **all approvers** or **any one approver** from a group.

> **Exam tip:** Approvals are set on **stages** (classic) or **Environments** (YAML). They are NOT set on the release pipeline itself.

**Pre-deployment vs Post-deployment approval timeout — critical distinction (Exam Favorite):**

| Approval type | When it runs | Timeout setting location |
|---|---|---|
| **Pre-deployment approval** | Before the stage deploys | Stage → **Pre-deployment conditions** → Approvals → Timeout |
| **Post-deployment approval** | After the stage deploys (before next stage) | Stage → **Post-deployment conditions** → Approvals → Timeout |

> **Exam tip (tested — series question):** "Releases must be approved before deployment. Deployments fail if approval takes > 2 hours, but policy allows 8 hours. Solution: modify Timeout in **Post-deployment conditions**" → **No**. The approval that gates deployment is a **pre-deployment** approval — modifying the post-deployment timeout has no effect on it. The correct fix is to change the timeout in **Pre-deployment conditions** to 8 hours.

### Release Gates (Exam Favorite)
Gates are **automated checks** — they poll external systems and proceed only when conditions are met. No human interaction needed.

**Gate types available in Azure DevOps:**
| Gate Type | What it checks |
|---|---|
| **Query Work Items** | Number of matching work items (e.g., open bugs) is below a threshold |
| **Invoke REST API** | Calls an HTTP endpoint; proceeds if response is success (2xx) |
| **Query Azure Monitor Alerts** | No active Azure Monitor alerts above a severity threshold |
| **Check Azure Policy Compliance** | Resources comply with Azure Policy rules |
| **Security and Compliance Assessment** | Runs the Azure DevOps Security and compliance assessment task — validates that planned deployments comply with Azure Policy before deployment |
| **Invoke Azure Function** | Executes an Azure Function and checks the return value |

**Gate evaluation behavior:**
- Gates are evaluated **periodically** (configurable interval, e.g., every 5 minutes).
- Gates must **all pass in the same evaluation cycle** before deployment proceeds.
- If gates don't pass within the **timeout window**, the deployment is rejected/paused.
- **Minimum duration before evaluation:** Gates won't be evaluated until this time passes after deployment starts (allows systems to stabilize).
- **Timeout:** Maximum time to wait for gates to pass.

**Gate placement:**
- **Pre-deployment gates:** Evaluated before the stage starts.
- **Post-deployment gates:** Evaluated after the stage completes (validates the deployment was successful).

> **Exam tip:** "Check that there are no active critical alerts before deploying to production" → **Query Azure Monitor Alerts** gate. "Ensure no high-priority bugs are open" → **Query Work Items** gate. "Call an external compliance service before deployment" → **Invoke REST API** gate.
>
> **Exam tip (tested — performance baseline):** "Releases negatively affected performance. Azure Monitor alerts are configured in staging. Prevent deployment to production if performance baseline is not met" → use a **gate** (Query Azure Monitor Alerts gate as a pre-deployment gate on the production stage). A trigger initiates a pipeline — it doesn't block it. An Azure Function requires custom code. An Azure Scheduler job is a scheduling tool unrelated to deployment gating.

> **Exam tip (HOTSPOT — two-stage QA→Prod pattern):** QA deploys to webapp1; Prod deploys to webapp2. Block Prod if Application Insights detects Failed requests after QA deployment:
> - **QA stage:** "Add a task to configure alert rules in Application Insights" — sets up the Application Insights alert rules on webapp1 during the QA deployment, so alerts can fire.
> - **Prod stage:** "Configure a gate in the pre-deployment conditions" — a Query Azure Monitor Alerts gate checks for active Failed request alerts before allowing Prod to deploy to webapp2. If alerts are firing, Prod is blocked.
> **Exam tip (tested — Bicep + Azure Policy):** "Pipeline deploys Bicep modules. Ensure all releases comply with Azure Policy before deploying to production" → Configure a **deployment gate** using the **Azure DevOps Security and compliance assessment task**. Do NOT use What-if (it previews changes but does NOT enforce Azure Policy compliance). Do NOT use a PR build check (it runs on PR, not as a gate before production). Do NOT use Azure Automation for What-if (custom code, more effort, still doesn't enforce policy).

### Approvals vs Gates
| Aspect | Approvals | Gates |
|---|---|---|
| Human interaction | Yes — human must approve | No — fully automated |
| Use case | Business sign-off, compliance | Automated quality/health checks |
| Trigger | Notification sent to approver | Periodic polling of external service |
| Location | Pre/post deployment conditions | Pre/post deployment conditions |

### GitOps Release Strategy
- **GitOps:** Using Git as the single source of truth for both application code **and** infrastructure/deployment configuration.
- Deployment is triggered by changes to the Git repository, not manually or by CI builds.
- A GitOps operator (e.g., Flux, ArgoCD) watches the Git repo and reconciles the live state with the desired state.
- Works well with Kubernetes deployments.
- Supports **pull-based deployments** (agent pulls config from Git) vs push-based (pipeline pushes to cluster).

**GitOps recommendations:**
- Store deployment manifests in a **separate repo** from application code.
- Use **branch-based promotion**: merge to `main` = deploy to prod; merge to `staging` = deploy to staging.
- All changes go through PRs with reviews — full audit trail.

---

## 3. Configure and Provision Environments

### Target Environment Types
| Environment | Examples | Provisioning approach |
|---|---|---|
| **Cloud PaaS** | Azure App Service, Azure Functions | ARM/Bicep templates, Azure CLI tasks |
| **Containers** | AKS, ACI, Container Apps | Kubernetes manifests, Helm charts |
| **Virtual Machines** | Azure VMs, on-premises | Deployment groups, ARM templates, Ansible |
| **Serverless** | Azure Functions, Logic Apps | ARM templates, Azure CLI |
| **Databases** | Azure SQL, Cosmos DB | DACPAC, migration scripts |

### Service Connections
- **Service connections** store credentials/configurations that allow Azure Pipelines to connect to external services.
- Configure in: **Project Settings → Service connections**.

**Common service connection types:**
| Type | Used for |
|---|---|
| **Azure Resource Manager** | Deploy to Azure resources (most common) |
| **Docker Registry** | Push/pull from Docker Hub or ACR |
| **Kubernetes** | Deploy to AKS or other Kubernetes clusters |
| **GitHub** | Connect to GitHub repos |
| **SSH** | Run commands on Linux/Unix servers |
| **Generic** | Custom HTTP endpoint |
| **Service Fabric** | Deploy to Service Fabric clusters |

**Azure Resource Manager connection options:**
- **Workload Identity Federation (recommended):** Federated credentials, no secret expiry.
- **Service Principal (automatic):** Azure DevOps creates a service principal with client secret.
- **Managed Service Identity (MSI) Authentication:** Uses the managed identity of the Azure-hosted agent or self-hosted agent's Azure VM — **no credentials or tokens stored in Azure DevOps**.

> **Exam tip:** **Workload Identity Federation** is the modern, most secure approach — no secrets to rotate. It is the recommended option for new service connections.

> **Exam tip (HOTSPOT — Key Vault + avoid persisting credentials):** When the scenario requires accessing Azure Key Vault secrets AND avoiding persisting credentials/tokens in Azure DevOps, AND the entire environment is in Azure:
> - **Service connection type: Team Foundation Server / Azure Pipelines service connection** (the Azure Pipelines agent uses its managed identity to access Azure services)
> - **Authentication method: Managed Service Identity Authentication** (no credentials stored — Azure AD manages the identity automatically)
>
> MSI provides the Azure Pipelines service with an automatically managed identity in Azure AD, which can authenticate to any Azure AD-supported service (including Key Vault) without any credentials in the code or pipeline. Azure Resource Manager service connection uses a service principal with a stored credential. OAuth 2.0 and Grant authorization require credentials/tokens to be persisted.

### Environments (YAML Pipelines)
- **Environments** in Azure DevOps YAML represent deployment targets with additional governance.
- Support **Deployment history** per resource.
- Can contain **resources**: Kubernetes namespaces, virtual machines (via deployment groups).
- Configure **approval and check policies** on environments (applies to all pipelines using that environment).

**Checks on environments:**
- Manual approval
- Branch control (e.g., only deploy from `main`)
- Business hours restriction
- Invoke Azure Function
- Invoke REST API
- Required template (enforce specific YAML template)

### Functional Test Automation
- **Functional tests** verify the application works correctly from a user perspective.
- Should be automated and run as part of the pipeline post-deployment.
- Tools: **Selenium** (UI/browser automation), **Playwright**, **Cypress**, **Postman/Newman** (API testing).
- Integrate results into Azure Pipelines using `PublishTestResults@2` task.
- Test results visible in the pipeline run summary.

### Shift-Left Testing
- Move testing **earlier** in the lifecycle — to the left on the timeline.
- **Developer workstation:** Unit tests, linting, SAST.
- **CI pipeline (build):** Unit tests, integration tests, code coverage.
- **Pre-deployment (test env):** Functional tests, integration tests.
- **Post-deployment (prod):** Smoke tests, synthetic monitoring.
- Earlier = cheaper to fix. A bug in production costs ~100× more than in development.

### Availability Tests
- **Availability tests** check that a deployed service is responding correctly from multiple geographic locations.
- Configure via **Azure Application Insights → Availability**:
  - **URL ping test:** Simple HTTP request to a URL; checks response code and optionally response content.
  - **Standard test:** More advanced HTTP checks (custom headers, redirects, certificates).
  - **Custom TrackAvailability test:** Fully custom code-based test.
- Alert when availability drops below threshold.
- Use as a **post-deployment gate** to verify the deployment succeeded.

### Azure Load Testing
- Fully managed load testing service based on Apache JMeter.
- Create tests from an existing JMeter script or from Azure portal.
- Integrates with Azure Pipelines via the **Azure Load Testing** task.
- Use as a **quality gate**: fail the pipeline if error rate > X% or p95 response time > Y ms.
- Identifies performance regressions before they reach production.

```yaml
- task: AzureLoadTesting@1
  inputs:
    azureSubscription: 'MyServiceConnection'
    loadTestConfigFile: 'loadtest.yaml'
    failCriteria:
      - 'avg(response_time_ms) > 500'
      - 'percentage(error) > 5'
```

### Functional Tests in Pipelines
**Flow:**
1. Deploy to test environment.
2. Run functional tests against the deployed URL.
3. Publish test results to Azure Pipelines.
4. Gate promotion to next stage on test pass rate.

**Selenium example in Azure Pipelines:**
```yaml
- task: VSTest@2
  inputs:
    testSelector: 'testAssemblies'
    testAssemblyVer2: '**\*UITest*.dll'
    searchFolder: '$(System.DefaultWorkingDirectory)'
```

---

## 4. Manage and Modularize Tasks and Templates

### Task Groups (Classic Pipelines)
- **Task groups** bundle multiple tasks into a single reusable unit in **classic pipelines**.
- Stored in Azure DevOps (not in source control).
- Used across multiple classic build/release pipelines in the **same project**.
- Support **parameters** to make them configurable.
- **Versioned:** Can have multiple versions (Draft, Preview, 1.x).

**Creating a task group:**
1. In a classic pipeline, select multiple tasks.
2. Right-click → Create task group.
3. Give it a name and define parameters.
4. Save — the task group is now available in the project.

> **Exam tip:** Task groups are classic-pipeline-only. YAML equivalent is **templates** stored in source control.

### Variables in Release Pipelines (Exam Favorite)

**Variable types and scopes:**
| Scope | Where defined | Accessible |
|---|---|---|
| **Pipeline variable** | Release pipeline settings | All stages |
| **Stage variable** | Per-stage configuration | That stage only |
| **Variable group** | Library (shared) | Any pipeline that links it |
| **System variables** | Auto-set by Azure DevOps | All tasks |
| **Queue-time variables** | Set when manually triggering | That run only |

**Variable scoping precedence (highest to lowest):**
1. Queue-time variable (set when triggering)
2. Stage variable (YAML: job-level)
3. Pipeline variable (YAML: stage-level)
4. Variable group
5. Pipeline variable (YAML: root-level)

> **Exam tip:** A variable defined at a more specific scope **overrides** the same variable at a broader scope. Stage variable wins over pipeline variable.

### Release Variables — Specific Behaviors
- Variables can be marked **settable at release time** — user can change them when creating a release.
- Variables can be marked **secret** — masked in logs, not passed between stages.
- **Output variables:** A task can set an output variable that other tasks/stages can consume.
  - Classic: via `##vso[task.setvariable variable=myVar;isOutput=true]value`
  - Reference in next stage: `$(stageName.jobName.taskName.myVar)`

### Variable Groups
- Centralize variables shared across multiple pipelines.
- Stored in **Library** (Project → Pipelines → Library).
- Can be **linked to Azure Key Vault** — secrets are sourced from Key Vault at runtime.
- Link to pipelines or to specific stages only.

**YAML usage:**
```yaml
variables:
  - group: MyVariableGroup
```

**Azure Key Vault integration:**
1. Create a variable group in Library.
2. Toggle "Link secrets from an Azure Key Vault."
3. Select Key Vault and the secrets to expose.
4. The pipeline reads secret values from Key Vault at runtime.

### Custom Build and Release Tasks
- Build using the **Azure DevOps Extension SDK** (`vss-web-extension-sdk`).
- Package with `tfx-cli` → generates a `.vsix` file.
- Publish to the **Visual Studio Marketplace** (public or private).
- Install on the Azure DevOps organization from the Marketplace.
- Tasks run on the pipeline agent; written in Node.js or PowerShell.

**task.json structure** (required for every custom task):
```json
{
  "id": "unique-guid",
  "name": "MyTask",
  "version": { "Major": 1, "Minor": 0, "Patch": 0 },
  "execution": {
    "Node16": { "target": "index.js" }
  },
  "inputs": [
    { "name": "myInput", "type": "string", "label": "My Input", "required": true }
  ]
}
```

---

## 5. Automate Inspection of Health

### Automated Health Inspection
- Use **automated gates and monitors** to detect issues before and after deployment.
- Replace manual "did the deploy work?" checks with systematic automated verification.
- Health checks run as part of the deployment pipeline — blocking promotion if they fail.

### Events and Notifications
- Azure DevOps can send notifications for pipeline events.
- **Event types:** Build completed, release created, release deployment started/completed/failed, approval pending, work item updated.
- Notifications sent via: **Email**, **Microsoft Teams**, **Slack** (via service hooks).

### Service Hooks
- **Service hooks** trigger external actions in response to Azure DevOps events.
- Configure in: **Project Settings → Service hooks → Create subscription**.

**Supported targets:**
| Target | Use case |
|---|---|
| **Microsoft Teams** | Post build/release notifications to a channel |
| **Slack** | Post notifications to a Slack channel |
| **Jenkins** | Trigger a Jenkins job when a build completes |
| **Azure Service Bus** | Send events to Service Bus for custom processing |
| **Azure Event Grid** | Broadcast events to multiple subscribers |
| **WebHooks (generic)** | POST JSON payload to any HTTP endpoint |
| **Trello** | Create Trello cards for events |
| **Bamboo** | Trigger Bamboo builds |

**Service hook events:**
- Build: Build completed, Build failed
- Release: Release created, Deployment started, Deployment completed, Approval pending
- Code: **Code pushed** (git push to branch), PR created, PR updated
- Work: Work item created, updated, etc.

> **Exam tip:** To notify Jenkins when a developer **commits/pushes code** to a branch in Azure Repos, use a service hook with the **"Code pushed"** event — NOT "Build completed." "Build completed" fires when an Azure Pipelines build finishes, which is a different event entirely. The correct event for a git push trigger is always "Code pushed."

### Azure DevOps Notifications Configuration
- **Personal notifications:** Managed by each user in their profile settings.
- **Team notifications:** Managed by team admins; notify the whole team.
- **Project notifications:** Managed by project admins.
- Default subscriptions exist for common events (can be disabled).
- Custom subscriptions filter by event type, pipeline, branch, status, etc.

### GitHub Notifications Configuration
- **Watch settings:** Notify for all activity, participating only, or ignore.
- **Subscriptions:** Per-repo, per-issue, per-PR notifications.
- **GitHub Apps:** Azure Boards app can post status updates back to GitHub.

### Measuring Quality of Release Process

**Key release quality metrics:**
| Metric | Definition |
|---|---|
| **Deployment frequency** | How often a new version is deployed to production |
| **Change failure rate** | % of deployments that cause an incident or require a hotfix |
| **Mean Time to Recovery (MTTR)** | Average time to restore service after a failure |
| **Lead time for changes** | Time from code commit to running in production |
| **Release success rate** | % of releases that complete without rollback |
| **Failed deployment rate** | % of deployments that fail |

> **Exam tip:** Deployment frequency and change failure rate are DORA metrics. Lower change failure rate + higher deployment frequency = high-performing DevOps team.

### Release Notes and Documentation
- **Release notes** communicate what changed in each release.
- Generate automatically from:
  - Work items associated with the build.
  - Commits since the last release.
  - GitHub PR descriptions.
- In Azure DevOps: **Release → Release notes** tab shows associated work items.
- Automate with `generate-changelogs` tasks or GitHub Actions `create-release`.

### Release Artifacts vs Release Process
| Aspect | Release Artifact | Release Process |
|---|---|---|
| Definition | The versioned deployable output (binary, package, image) | The pipeline/workflow that deploys the artifact |
| Versioned | Yes — tied to a build number/tag | Yes — but as pipeline configuration |
| Immutable | Yes — should not change after creation | Can be modified (pipeline config) |
| Traceability | Traces back to build → commit → work item | Traces execution history and approvals |

### Common Release Management Tools
| Tool | Type | Key features |
|---|---|---|
| **Azure Pipelines (Classic)** | Microsoft | Full ALM integration, approval gates, deployment groups |
| **Azure Pipelines (YAML)** | Microsoft | Pipeline as code, environments, templates |
| **GitHub Actions** | GitHub | GitHub-native, Marketplace actions, environment protection |
| **Jenkins** | Open source | Highly customizable, large plugin ecosystem |
| **Octopus Deploy** | Third-party | Advanced deployment patterns, runbooks |
| **Spinnaker** | Open source | Multi-cloud CD, sophisticated deployment strategies |
| **ArgoCD / Flux** | GitOps | Kubernetes-native GitOps operators |

---

## 6. Additional Exam Topics (Beyond MS Learn)

### Third-Party Agile Tool Integration with Azure DevOps

When a question asks which Agile/project management tool to integrate with Azure DevOps pipelines for **deployment tracking with minimum effort**, the answer is **Jira**.

| Tool | Azure DevOps Integration | Deployment Tracking |
|---|---|---|
| **Jira** | Native marketplace extension | ✅ Yes — shows deployment status per issue/commit |
| **Trello** | Service hook only (creates cards) | ❌ No deployment status tracking |
| **Basecamp** | No native integration | ❌ No integration |
| **Asana** | No native Azure DevOps integration | ❌ Requires third-party connectors |

**What the Jira + Azure DevOps integration provides:**
- Links commits, branches, and PRs to Jira issues
- Reports deployment status (which environment each issue reached) back to Jira automatically
- Pre-built extension from the Azure DevOps Marketplace — minimal setup, no custom code

> **Exam tip:** "Minimize integration effort + track commits deployed to production + report deployment status" → **Jira**. It is the only option with a purpose-built Azure DevOps Marketplace extension that natively connects release pipelines to work items.

---

### Azure Boards Work Item Link Types (Dependency Management)

Work items in Azure Boards can be linked to express relationships:

| Link Type | Meaning | When to Use |
|---|---|---|
| **Predecessor** | Linked item must finish before this item starts | Item A depends on item3 → add item3 as Predecessor of item A |
| **Successor** | This item must finish before the linked item starts | Opposite direction of Predecessor |
| **Related** | General association, no ordering implied | Items share context but no strict dependency |
| **Parent / Child** | Hierarchical (e.g., Epic → Feature → Story → Task) | Work breakdown structure |
| **Duplicate / Duplicate Of** | One item is a duplicate of another | Deduplication |

**How to define a dependency:**
1. Open the **dependent** work item (e.g., item A)
2. Go to the **Links** tab
3. Select **Add link**
4. Set **Link type = Predecessor**
5. Enter the ID of the item that must complete first (e.g., item3)

> **Exam tip:** Always define the dependency **from the dependent item** (item A). Set link type to **Predecessor** and point to the blocking item. "Related" does NOT express execution dependency — it's a general association only.

---

### Deployment Strategies Deep Dive (Exam Favorite)
These strategies are tested heavily — know when to use each:

| Strategy | How it works | Downtime | Risk | Use case |
|---|---|---|---|---|
| **Blue-Green** | Two identical environments (Blue=live, Green=new). Switch traffic after validation | Zero | Low (instant rollback by switching back) | Critical apps needing zero-downtime |
| **Canary** | Deploy to small % of users/instances first; gradually increase | Zero | Low (limited blast radius) | Gradual rollout; A/B-style testing |
| **Rolling** | Replace instances one at a time in batches | Near-zero | Medium (multiple versions briefly live) | Large fleets, gradual update |
| **Ring** | Deploy to "rings" (inner=early adopters, outer=all users) | Zero | Low | Enterprise-wide staged rollout |
| **Feature flags** | Code deployed to all, features toggled on/off per user | Zero | Low (toggle off if issues) | Decouple deploy from release |
| **A/B testing** | Route different users to different versions to test metrics | Zero | Low | Validate business hypothesis |
| **Recreate** | Stop old version, start new version | Yes | High | Dev/test environments only |

### Blue-Green Deployment in Azure
**Azure App Service Deployment Slots:**
- Deployment slots are separate environments (live slot + staging slot).
- Deploy to **staging slot** first.
- Run warm-up and smoke tests.
- **Swap slots** — staging becomes live, old live becomes staging. Zero downtime.
- Rollback = swap again.

```yaml
- task: AzureAppServiceManage@0
  inputs:
    azureSubscription: 'MyServiceConnection'
    action: 'Swap Slots'
    webAppName: 'myapp'
    sourceSlot: 'staging'
    swapWithProduction: true
```

### VIP Swap (Classic Azure Cloud Services)
- **VIP swap** swaps the Virtual IP address between staging and production environments.
- Zero-downtime deployment for Azure Cloud Services (classic).
- Similar concept to App Service slot swap but for VMs/Cloud Services.

### Hotfix Path
- A plan for deploying **critical fixes** to production quickly, bypassing normal release gates.
- Hotfix branch cut from production tag → minimal testing → fast-tracked through approval.
- Must still go through at minimum: code review + core smoke test.
- Post-hotfix: merge fix back into `develop`/`main` to avoid regression.

### Load Balancing and Rolling Deployments
- **Rolling deployment** with a load balancer:
  1. Remove instance from load balancer.
  2. Update instance.
  3. Validate instance is healthy.
  4. Add back to load balancer.
  5. Repeat for next instance.
- Ensures no downtime as at least some instances always serve traffic.

### Deployment Groups (Classic Pipelines)
- A **deployment group** is a pool of target machines for deploying to physical/virtual servers.
- Machines register themselves with Azure DevOps by running an agent.
- Used in classic release pipelines for on-premises or IaaS deployments.
- YAML equivalent: **Environment with VM resources**.

**Setting up a deployment group:**
1. Project Settings → Deployment groups → New.
2. Run the registration script on each target machine.
3. Reference in classic release pipeline job as "Deployment group job."

### Resiliency Strategy for Deployment
- **Health checks:** Verify application is healthy before routing traffic.
- **Automatic rollback triggers:** Roll back if error rate spikes post-deployment.
- **Graceful shutdown:** Drain connections before stopping old version.
- **Circuit breakers:** Stop cascading failures in microservices.
- **Retry logic:** Transient failure handling in application code.

### Pipeline Retention Policy
- Control how long pipeline runs, artifacts, and releases are retained.
- Set at: **Project Settings → Pipelines → Settings → Retention policy**.
- Defaults: keep last N builds per branch; releases retained for X days.
- Override: pin a specific run to prevent it from being deleted.
- Important for compliance — retain audit-relevant releases longer.

### Classic Release Pipeline Concepts (Still Tested)
- **Continuous deployment trigger:** On the artifact source; fires when new artifact is available.
- **Stage trigger (scheduled):** Runs a stage at a scheduled time regardless of new artifacts.
- **Manual-only trigger:** Stage requires explicit approval to run.
- **Artifact filter:** Deploy to a stage only from specific branches (e.g., only `main` → Prod).
- **Stage conditions:** Run stage only if previous stage succeeded, or always run (for cleanup).

**Release pipeline HOTSPOT — triggers and continuous delivery (Exam Favorite):**

| Question | Answer | Why |
|----------|--------|-----|
| How many stages have triggers set? | Count stages with "after stage/release" pre-deployment triggers | Development, QA, Pre-Prod, Load Test, Production = **5**; Stakeholder Review and Internal Review are manual |
| Which component to modify for continuous delivery? | **The blocking stage** (e.g. Internal Review) | If a stage in the pipeline has a manual or scheduled trigger, it blocks continuous delivery — change its trigger to "after stage" to enable CD |

> **Exam tip:** When asked "which component to modify to enable continuous delivery," look for the **stage whose trigger is blocking the flow** (manual/scheduled), not the artifact. The artifact CD trigger controls when a *new release* is created; a stage with a manual trigger blocks the release from flowing automatically through the pipeline.

### Visual Studio App Center — Mobile App Distribution

**App Center** is Microsoft's mobile CI/CD and distribution platform for iOS, Android, and other platforms.

**Distribution groups:** Segments of testers or users who receive app releases. Three types:

| Type | Scope | Access | Use case |
|---|---|---|---|
| **Private** | App-level | Invited members only | Restricted beta testing for a single app |
| **Public** | App-level | Anyone with the link (no sign-in) | Open beta distribution |
| **Shared** | **Organization-level** | Reusable across multiple apps in the org | Consistent access control across a suite of apps |

> **Exam tip:** "Managed at the organization level" + "suite of apps" = **Shared** distribution groups. Shared groups are defined once at the org level and applied to multiple apps — no need to recreate the group per app. Private and Public are app-scoped only.

**iOS App Distribution — Certificate requirement:**

| File Type | Contents | Used for |
|---|---|---|
| **`.p12`** | Certificate + private key (PKCS#12, Apple format) | **iOS code signing for distribution** — upload to App Center |
| **`.pfx`** | Certificate + private key (PKCS#12, Windows format) | Windows/Azure code signing (not iOS) |
| **`.cer`** | Public certificate only (no private key) | Certificate verification only — cannot sign |
| **`.pvk`** | Private key only (Windows format) | Paired with `.cer` in Windows; not used for iOS |

### App Center SDK Services

When initializing the App Center SDK, you must explicitly opt in to each service:

```swift
MSAppCenter.start("{Your App Secret}", withServices: [MSAnalytics.self, MSCrashes.self])
```

| Service | Purpose |
|---|---|
| **MSAnalytics** | Tracks device types, sessions, user events — "device types in use" |
| **MSCrashes** | Captures and reports application crashes |
| **MSDistribute** | Handles in-app update distribution to testers |
| **MSPush** | Sends push notifications to devices |

> **Exam tip:** "Centralize crash reporting and device types in use" → `[MSAnalytics.self, MSCrashes.self]`. MSDistribute = in-app updates (not reporting). MSPush = push notifications (not reporting). By default, no services are started — you must explicitly pass each service class to the `withServices` array.

> **Exam tip:** To distribute an iOS app through App Center (including to a private distribution group), you must upload the **`.p12`** certificate file. This is the Apple-native PKCS#12 format that bundles both the distribution certificate and private key, enabling App Center to sign the IPA before delivery. `.pfx` is the Windows equivalent but is NOT used for iOS distribution.

### AKS Deployment with Azure DevOps — Sequence

To deploy an application to an **Azure Kubernetes Service (AKS)** cluster using Azure DevOps pipelines, perform these three steps in order:

| Step | Action | Why |
|------|--------|-----|
| 1 | **Create a service principal in Azure Active Directory (Azure AD)** | Azure DevOps needs an Azure AD identity to authenticate against AKS/Azure |
| 2 | **Configure RBAC roles in the cluster** | Grant the service principal the necessary Kubernetes permissions (e.g., `cluster-admin` or a scoped role) |
| 3 | **Add a Helm package and deploy task to the deployment pipeline** | Helm is the standard Kubernetes package manager used to deploy app manifests to AKS via pipeline |

**Eliminated options (exam distractors):**

| Option | Why Wrong |
|--------|-----------|
| Create a service account in the cluster | Kubernetes service accounts are for workloads *inside* the cluster, not for Azure DevOps → cluster connectivity |
| Add Azure Function App for Container task | For Azure Functions deployments, not AKS |
| Add Docker Compose task | For Docker Compose environments, not Kubernetes |

> **Exam tip:** Service principal + RBAC + Helm is the correct AKS deployment sequence. "Service account in the cluster" is a deliberate distractor — it sounds similar to service principal but serves a completely different purpose (intra-cluster workload identity, not external pipeline auth).

---

**Q31 — Yes/No: Fix deployment failure when approvals exceed 2 hours (policy requires 8 hours)**
> Solution: From Pre-deployment conditions, modify the Timeout setting for pre-deployment approvals.

**A: Yes.** ⚠️ *Dump says No — INCORRECT*
The pre-deployment approval **Timeout** setting controls how long Azure DevOps waits for an approver before auto-rejecting the deployment. Current timeout = 2 hours → deployments fail after 2 hours. Changing it to 8 hours = deployments only fail after 8 hours. This is the exact and correct control for this scenario.

> **Exam tip:** Pre-deployment approval Timeout = how long the system waits for human approval before auto-failing the stage. Separate from Gates timeout (which controls polling duration for automated checks).

### AKS Cluster Provisioning with Azure AD RBAC — Sequence

To **provision** an AKS cluster and set permissions using RBAC roles that reference Azure AD identities:

| Step | Object | Why |
|---|---|---|
| 1 | **a cluster** | Create the AKS cluster first — the managed identity is generated during cluster creation |
| 2 | **a system-assigned managed identity** | The cluster is automatically assigned a system-assigned managed identity, which is registered as a service principal in the Azure AD tenant. Retrieve its Object ID from Azure AD to use in RBAC bindings |
| 3 | **an RBAC binding** | Create the Kubernetes RBAC binding using the Azure AD Object ID of the managed identity, granting the identity the required cluster permissions |

**Eliminated option:** "an application registration in contoso.com" — App registrations are for OAuth/auth flows; the AKS cluster's own system-assigned managed identity (a service principal in Azure AD) is what RBAC bindings reference, not a manually created app registration.

> **Exam tip:** AKS *provisioning* with Azure AD RBAC = cluster → system-assigned managed identity → RBAC binding. This differs from AKS *deployment via Azure DevOps* (service principal → RBAC → Helm). The key distinction: provisioning creates the cluster and its Azure AD identity; deployment uses an existing service principal to push app manifests.

---

## 7. Past Exam Scenario Bank

---

**Q20 — Azure Boards: defining work item dependency**
> You manage projects using Azure Boards. Work item A is dependent on work item item3 (item3 must complete before item A can start). You need to define the dependency for item A. What should you do?

**A:** From **item A**, open the **Links tab**, select **Add link**, set **Link type to Predecessor**, and add the ID of **item3**.

Item3 is the Predecessor (must finish first); item A is the Successor (depends on item3). You always define the relationship from the dependent item by linking to its Predecessor.

| Link Type | Meaning |
|---|---|
| **Predecessor** | The linked item must complete before this item starts |
| **Successor** | This item must complete before the linked item starts |
| **Related** | General association — no ordering implied |
| **Duplicate / Duplicate Of** | Marks one item as a duplicate of another |
| **Child / Parent** | Hierarchical relationship (e.g., Task under User Story) |

---

**Q1 — Release gate type: active bugs blocking deployment**
> You need to ensure that no more than 5 high-priority bugs are open before deploying to production. Which release gate type should you configure?

**A: Query Work Items gate.**
Configure a work item query that counts open high-priority bugs. Set the threshold to ≤ 5. The gate evaluates periodically and blocks deployment if the count exceeds 5.

---

**Q2 — Release gate type: external compliance service**
> Before deploying to production, you must get confirmation from an external compliance system that a change ticket is approved. Which gate type should you use?

**A: Invoke REST API gate.**
Calls the compliance system's HTTP endpoint. If it returns a success status code (2xx), the gate passes. Configure the endpoint URL and authentication.

---

**Q3 — Release gate type: no monitoring alerts**
> You want to verify that there are no active critical Azure Monitor alerts before deployment proceeds. Which gate type do you use?

**A: Query Azure Monitor Alerts gate.**
Checks configured Azure Monitor alert rules. Passes only when no active alerts exceed the specified severity.

---

**Q4 — Approvals vs Gates**
> You need a human sign-off from the security team before deploying to production. Should you use a release gate or a release approval?

**A: Release approval (pre-deployment).**
Approvals require human action. Gates are automated. For human sign-off, configure a pre-deployment approval and add the security team as required approvers.

---

**Q5 — Trigger type: deploy automatically after every build**
> Every time the build pipeline for your main branch succeeds, a release should be created and deployed to the Dev environment automatically. What type of trigger should you configure?

**A: Continuous Deployment trigger** on the artifact source (linked to the build pipeline). This automatically creates a new release when a new build artifact is available.

---

**Q6 — Variable scope: stage-specific value**
> You have a connection string variable that must have different values for Dev, QA, and Production stages. How should you configure this?

**A:** Define the variable at **stage scope** in each stage with the environment-specific value. Stage variables override pipeline-level variables of the same name.

---

**Q7 — Task group vs YAML template**
> You are using classic release pipelines. You have 10 pipelines that all need the same 5 deployment steps. What is the best way to reuse these steps?

**A: Create a task group.**
Task groups bundle tasks for reuse across classic pipelines in the same project. (YAML equivalent would be templates, but this scenario is classic.)

---

**Q8 — Variable group with Key Vault**
> Your pipeline needs to use a database password stored in Azure Key Vault without hardcoding it. What is the recommended approach?

**A:** Create a **Variable Group** in Library and link it to Azure Key Vault. Map the secret to a variable name. The pipeline accesses it as a regular variable — the value is fetched from Key Vault at runtime and masked in logs.

---

**Q9 — Blue-green deployment with App Service**
> You need to deploy a new version of an Azure App Service application with zero downtime and the ability to instantly roll back. What approach should you use?

**A: Azure App Service Deployment Slots (slot swap).**
Deploy to the staging slot, run validation, then swap staging to production. Rollback = swap again. Zero downtime.

---

**Q10 — Deployment strategy: limited blast radius**
> You are deploying a new feature to a large user base. You want to limit the potential impact by initially only exposing the new version to 5% of users. Which strategy should you use?

**A: Canary deployment.**
Deploy to a small subset of users/instances first. Monitor for issues. Gradually increase the percentage until 100% are on the new version.

---

**Q11 — Deployment strategy: test with specific user segment**
> You want to compare two versions of your app to determine which one leads to higher conversion rates. You will show version A to 50% of users and version B to the other 50%. Which strategy is this?

**A: A/B testing (experiment toggles / feature flags).**
Route different users to different versions to measure business outcomes.

---

**Q12 — Artifact filter on release stage**
> You have a release pipeline with Dev, QA, and Prod stages. You want the Prod stage to only be triggered when the artifact comes from the `main` branch. How do you configure this?

**A:** Set an **artifact filter** on the Prod stage's pre-deployment conditions. Filter by branch = `main`. Releases from any other branch will not trigger the Prod stage.

---

**Q13 — Service connection type for Azure deployment**
> A pipeline needs to deploy resources to an Azure subscription. What type of service connection should you create, and what is the modern recommended authentication method?

**A:** Create an **Azure Resource Manager** service connection using **Workload Identity Federation** (no secrets, no expiry). This is the recommended approach over service principal with client secret.

---

**Q14 — Hotfix process**
> A critical bug is found in production that must be fixed immediately. Normal releases take 2 weeks through QA. What should your hotfix process look like?

**A:**
1. Cut a hotfix branch from the production release tag.
2. Apply the fix with minimal scope.
3. Run core unit and smoke tests.
4. Fast-track approval (security/lead reviewer only).
5. Deploy directly to production.
6. Merge fix back to `main`/`develop` to prevent regression.

---

**Q19 — Approval timeout: pre vs post-deployment conditions**
> Releases must be approved by a team leader before deployment. Approvals must occur within 8 hours (policy). Deployments fail if approvals take longer than 2 hours. Solution: from Post-deployment conditions, you modify the Timeout setting for post-deployment approvals. Does this meet the goal?
>
> A. Yes  B. No

**A: B — No.**
The approval that controls whether a release is deployed is a **pre-deployment approval** (it must be granted *before* the stage deploys). Modifying the timeout in Post-deployment conditions changes the timeout for a different approval — one that runs after the stage completes. To fix the 2-hour timeout to 8 hours, you must change the timeout in **Pre-deployment conditions** → Approvals → Timeout.

---

**Q18 — Deploy artifacts to on-premises servers: Docker solution**
> Your build process creates several artifacts. You need to deploy the artifacts to on-premises servers. Solution: deploy a Docker build to an on-premises server and add a Download Build Artifacts task to the deployment pipeline. Does this meet the goal?
>
> A. Yes  B. No

**A: B — No.**
Docker builds produce container images — they are suitable for containerized workloads but are not the correct mechanism for deploying general build artifacts to on-premises servers. The correct solution is to create a **Deployment Group** in Azure DevOps, install the Azure Pipelines agent on each on-premises server, and use a **Deployment Group job** in the release pipeline with tasks to download and deploy the artifacts.

---

**Q17 — Prevent deployment if performance baseline not met**
> Past releases negatively affected system performance. Azure Monitor alerts are configured. You need to ensure new releases are only deployed to production if they meet defined performance baseline criteria in staging first. What should you use to prevent deployment of failing releases?
>
> A. An Azure Scheduler job  B. A trigger  C. A gate  D. An Azure function

**A: C — A gate.**
Configure a **pre-deployment gate** on the production stage using the **Query Azure Monitor Alerts** gate type. If the staging environment fires Azure Monitor alerts (performance threshold breached), the gate fails and the release is blocked from reaching production. Gates are automated checks — no human interaction or custom code required. A trigger starts a pipeline; it cannot block one. An Azure Scheduler job is for scheduled automation, not deployment gating. An Azure Function could work but requires custom code (more effort, not the minimal solution).

---

**Q16 — Azure Policy compliance gate with Bicep**
> You have a pipeline (Pipeline1) that deploys Azure resources using Bicep modules from a GitHub repository. You need to ensure all releases comply with Azure Policy before being deployed to production. What should you do?
>
> A. Configure a deployment gate including the Azure DevOps Security and compliance assessment task.
> B. Create a build that runs on PR creation and assesses code for compliance.
> C. Add a What-if deployment step before the deployment step.
> D. Configure a deployment gate that uses Azure Automation to run a What-if deployment.

**A: A — Configure a deployment gate with the Security and compliance assessment task.**
This task evaluates whether the planned deployment complies with Azure Policy assignments **before** the release reaches production, with no custom code. What-if (C, D) only previews resource changes — it does not check Azure Policy compliance. A PR build (B) runs at code review time, not as a production gate.

---

**Q15 — Gate evaluation: all gates must pass simultaneously**
> You have 3 gates configured on a pre-deployment condition: Query Work Items, Invoke REST API, and Query Azure Monitor Alerts. How does Azure DevOps evaluate these?

**A:** All 3 gates are evaluated **periodically** (e.g., every 5 minutes). They must **all pass in the same evaluation cycle** before deployment proceeds. If any gate fails, evaluation retries until all pass or the timeout is reached.

---

**Q21 — Matching release strategy to application goal**
> App1: Failure has major impact; a small group of opted-in users need to test new releases.
> App2: Minimize deployment time; must be able to roll back as quickly as possible.
> Choose from: Blue/Green, Canary, Rolling.

**A:**
- **App1 → Canary deployment** — routes new releases to a small, opted-in subset of users first; limits blast radius for a high-risk application.
- **App2 → Blue/Green deployment** — two identical environments always ready; cutover is instant (single traffic switch), rollback is equally instant (switch back). Minimizes both deploy time and rollback time.

Rolling is not suitable for App2 because rollback requires reversing each instance incrementally — not "as quickly as possible."

---

**Q22 — Third-party Agile tool integration: track commits deployed to production**
> You have a release pipeline in Azure DevOps. You need to integrate work item tracking and an Agile project management system to: track whether commits are deployed to production, report deployment status, and minimize integration effort. Which system should you use?
> A. Trello  B. Jira  C. Basecamp  D. Asana

**A: B — Jira.**
Jira has a native Azure DevOps Marketplace extension that links commits/deployments to Jira issues and reports deployment status (environment reached) back to Jira automatically — no custom code required. Trello only creates cards via service hooks (no deployment tracking). Basecamp and Asana have no native Azure DevOps integration.

---

**Q26 — HOTSPOT: Service endpoint for Key Vault — avoid persisting credentials**
> You manage build and release pipelines in Azure DevOps. Your entire managed environment resides in Azure. You need to configure a service endpoint for accessing Azure Key Vault secrets. Requirements: ensure secrets are retrieved by Azure DevOps; avoid persisting credentials and tokens in Azure DevOps. How should you configure the service endpoint?
>
> Service connection type: Azure Resource Manager / Generic service / Team Foundation Server / Azure Pipelines service connection
> Authentication method: Azure Active Directory OAuth 2.0 / Grant authorization / Managed Service Identity Authentication

**A:**
- **Service connection type: Team Foundation Server / Azure Pipelines service connection** — the Azure Pipelines service connection type enables the pipeline to use the agent's managed identity to access Azure resources including Key Vault. Azure Resource Manager connections use a service principal with a stored credential.
- **Authentication method: Managed Service Identity Authentication** — MSI provides the Azure Pipelines service with an automatically managed identity in Azure AD. It authenticates to Key Vault (and any Azure AD-supported service) without any credentials stored in Azure DevOps. OAuth 2.0 and Grant authorization both require persisting credentials/tokens.

---

**Q29 — HOTSPOT: Release pipeline — stages with triggers and continuous delivery**
> A release pipeline has 7 stages: Development, QA, Pre-Prod, Production, Stakeholder Review, Load Test, Internal Review. The artifact is "Web Application."
> - How many stages have triggers set? [0/1/2/3/4/5/6/7]
> - Which component should you modify to enable continuous delivery? [Development stage / Internal Review stage / Production stage / Web Application artifact]

**A:**
- **5 stages have triggers set** — Development, QA, Pre-Prod, Load Test, and Production each have pre-deployment triggers. Stakeholder Review and Internal Review do not.
- **The Internal Review stage** — this stage has a manual/scheduled trigger that blocks continuous delivery. Modifying its trigger to "after stage" enables the pipeline to flow automatically.

---

**Q28 — Service hook event: notify Jenkins on code commit**
> You integrate a cloud-hosted Jenkins server with Azure DevOps. You need Azure DevOps to send a notification to Jenkins when a developer commits changes to a branch in Azure Repos. Solution: Create a service hook subscription that uses the "build completed" event. Does this meet the goal?
> A. Yes  B. No

**A: B — No.**
The "build completed" event fires when an Azure Pipelines build finishes — not when code is committed. To notify Jenkins on a git push, the service hook must use the **"Code pushed"** event (Code category). "Build completed" is the wrong trigger category entirely.

---

**Q32 — Yes/No: Email subscription to notify Jenkins on code commit**
> You integrate a cloud-hosted Jenkins server with Azure DevOps. You need Azure DevOps to send a notification to Jenkins when a developer commits changes to a branch in Azure Repos. Solution: Create an email subscription to an Azure DevOps notification. Does this meet the goal?
> A. Yes  B. No

**A: B — No.**
Email subscriptions send notifications to **humans** (email addresses) — they cannot trigger or call an external system like Jenkins. To notify Jenkins on a code push, the correct solution is a **Service Hook** with the "Code pushed" event, which sends an HTTP POST to the Jenkins build trigger URL. This is a variant of the same scenario — any solution other than a service hook with the "Code pushed" event does NOT meet the goal.

---

**Q33 — MULTI-SELECT: Connect Jenkins to Azure Repos for source code retrieval**
> Your company uses cloud-hosted Jenkins for builds. You need to ensure that Jenkins can retrieve source code from Azure Repos. Which three actions should you perform?
>
> A. Add the TFS plug-in to Jenkins  B. Create a PAT in your Azure DevOps account  C. Create a webhook in Jenkins  D. Add a domain to your Jenkins account  E. Create a service hook in Azure DevOps

**A: A, B, and E.**
- **TFS plug-in (A)** — Jenkins-side plugin that enables connectivity to Azure Repos (Azure DevOps was formerly TFS/VSTS)
- **PAT (B)** — authentication credential for Jenkins to access Azure Repos
- **Service hook in Azure DevOps (E)** — notifies Jenkins when code is pushed, triggering source retrieval

Webhook in Jenkins (C) is the receiving side but the trigger originates from Azure DevOps via a service hook. Adding a domain (D) is irrelevant to source control integration.

---

**Q27 — DRAG DROP: Deploy application to AKS cluster using Azure DevOps — sequence**
> You have an AKS cluster. You need to deploy an application to the cluster by using Azure DevOps. Which three actions should you perform in sequence?
>
> Available actions: Create a service account in the cluster / Create a service principal in Azure Active Directory (Azure AD) / Add an Azure Function App for Container task to the deployment pipeline / Add a Helm package and deploy a task to the deployment pipeline / Add a Docker Compose task to the deployment pipeline / Configure RBAC roles in the cluster

**A (in order):**
1. **Create a service principal in Azure Active Directory (Azure AD)** — Azure DevOps needs service principals to authenticate against ACR (push/pull images and charts) and AKS (deploy the application).
2. **Add a Helm package and deploy a task to the deployment pipeline** — Helm packages and deploys the application to the AKS cluster.
3. **Add a Docker Compose task to the deployment pipeline** — Docker Compose (via Dockerfile) builds and packages the container image that Helm deploys.

"Create a service account in the cluster" is a Kubernetes-internal concept (for workloads running inside the cluster) — not for pipeline authentication. "Configure RBAC roles in the cluster" is part of service principal setup, not a separate pipeline step.

---

**Q25 — HOTSPOT: Block Prod deployment based on Application Insights alerts after QA**
> You have a release pipeline with QA (deploys to webapp1) and Prod (deploys to webapp2) stages. You need to block deployments to webapp2 if Application Insights generates Failed requests alerts after deploying new code to webapp1. What should you do for each stage?
>
> QA options: Add a task to configure alert rules in Application Insights / Configure a gate in the pre-deployment conditions / Configure an auto-redeploy trigger in the post-deployment conditions / Configure a post-deployment approval in the post-deployment conditions
>
> Prod options: Add a task to configure an alert rule in Application Insights / Configure a gate in the pre-deployment conditions / Configure a trigger in the pre-deployment conditions / Configure the Deployment queue settings in the pre-deployment conditions

**A:**
- **QA → Add a task to configure alert rules in Application Insights.** The QA stage sets up the Application Insights alert rules on webapp1 during deployment. Without alert rules configured, Application Insights cannot generate Failed request alerts to check against.
- **Prod → Configure a gate in the pre-deployment conditions.** A **Query Azure Monitor Alerts** pre-deployment gate on the Prod stage queries Application Insights for active Failed request alerts before allowing deployment to webapp2. If alerts are firing (indicating webapp1 is unhealthy), the gate blocks Prod from proceeding.

A pre-deployment gate on Prod **blocks** deployment; a trigger **starts** deployment — these are opposites. Deployment queue settings control parallelism, not conditional blocking. Auto-redeploy and post-deployment approval on QA do not affect whether Prod deploys.

---

**Q36 — HOTSPOT: App Center SDK initialization**
> Visual Studio App Center must be used to centralize the reporting of mobile application crashes and device types in use. How should you complete the code to initialize App Center?
>
> ```swift
> MSAppCenter.start
> ( "{Your App Secret}",
>   withServices: [Box1] [Box2]
> )
> ```
> Box 1: [MSAnalytics.self, / MSDistribute.self, / MSPush.self,]
> Box 2: [MSAnalytics.self] / MSCrashes.self] / MSDistribute.self]]

**A:**
- **Box 1: `[MSAnalytics.self,`**
- **Box 2: `MSCrashes.self]`**

Result: `MSAppCenter.start("{Your App Secret}", withServices: [MSAnalytics.self, MSCrashes.self])`

MSAnalytics = tracks device types and sessions ("device types in use"). MSCrashes = captures and reports crashes. MSDistribute = in-app update delivery (not reporting). MSPush = push notifications (not reporting). By default no services start — each must be explicitly passed.

---

**Q35 — HOTSPOT: App Center — control mobile build access at organization level**
> Your company is creating a suite of three mobile applications. You need to control access to the application builds. The solution must be managed at the organization level.
>
> Groups to control the build access: [Active Directory groups / Azure Active Directory groups / Microsoft Visual Studio App Center distribution groups]
> Group type: [Private / Public / Shared]

**A:**
- **Groups: Microsoft Visual Studio App Center distribution groups** — App Center's native distribution groups control who can access mobile builds. On-premises AD has no App Center integration. Azure AD groups manage identity/auth, not mobile build distribution.
- **Group type: Shared** — Shared groups are created at the **organization level** and can be applied across multiple apps. Private and Public are app-scoped only. "Suite of three apps" + "managed at org level" = Shared.

---

**Q24 — App Center iOS distribution certificate**
> Your company develops an iOS app. All users have devices that are members of a private distribution group in Visual Studio App Center. You plan to distribute a new release. Which certificate file type should you upload to App Center?
> A. .cer  B. .pvk  C. .pfx  D. .p12

**A: D — `.p12`.**
`.p12` is the Apple-native PKCS#12 format that bundles both the distribution certificate and private key. App Center requires this file to sign the iOS IPA before delivering it to the distribution group. `.cer` contains only the public certificate (no private key — cannot sign). `.pvk` is a Windows-only private key file. `.pfx` is also PKCS#12 but the Windows format — not used for iOS distribution.

---

**Q23 — Azure SQL Database Deployment task: correct artifact type**
> You have an Azure DevOps project and an Azure SQL database named DB1. You need to create a release pipeline that uses the Azure SQL Database Deployment task to update DB1. Which artifact should you deploy?
> A. BACPAC  B. DACPAC  C. LDF file  D. MDF file

**A: B — DACPAC.**
The Azure SQL Database Deployment task (`SqlAzureDacpacDeployment@1`) uses a **DACPAC** — a package containing only the database schema (tables, views, stored procedures). It applies incremental schema changes to the target database. BACPAC contains schema + data and is used for migration/backup, not pipeline deployments. MDF and LDF are raw SQL Server data/log files — not deployable via pipeline tasks.

---

**Q28 — DRAG DROP: AKS cluster provisioning with Azure AD RBAC — sequence**
> Your company has an Azure subscription associated with Azure AD tenant contoso.com. You need to provision an AKS cluster and set permissions using RBAC roles that reference identities in contoso.com. Which three objects should you create in sequence?
>
> Objects: a system-assigned managed identity / a cluster / an application registration in contoso.com / an RBAC binding

**A (in order):**
1. **a cluster** — create the AKS cluster first; the system-assigned managed identity is generated during creation
2. **a system-assigned managed identity** — the cluster's managed identity is registered as a service principal in contoso.com Azure AD; retrieve its Object ID from Azure AD portal
3. **an RBAC binding** — create the RBAC binding referencing the managed identity's Object ID to grant cluster permissions

"an application registration in contoso.com" is the distractor — app registrations are for OAuth flows; the cluster's system-assigned managed identity is already a service principal in the tenant and is what RBAC bindings reference.

---

**Q30 — HOTSPOT: Kubernetes pod probe configuration**
> You have an AKS pod. You need to configure a probe to: confirm the pod is responding to service requests, check status four times a minute, and initiate a shutdown if the pod is unresponsive.

```yaml
[Box1]:          # livenessProbe / readinessProbe / ShutdownProbe / startupProbe
  httpGet:
    path: /checknow
    port: 8123
  [Box2]: 15     # initialDelaySeconds / periodSeconds / timeoutSeconds
```

**A:**
- **Box 1: `livenessProbe`** — checks if the pod is alive and responding; **kills and restarts** the container on failure (= "initiate a shutdown"). readinessProbe only removes pod from load balancer endpoints without restarting it.
- **Box 2: `periodSeconds: 15`** — 4 times per minute = 60 ÷ 4 = **15 seconds** between checks.

> **Exam tip — Kubernetes probe types:**
> | Probe | Purpose | On failure |
> |---|---|---|
> | `livenessProbe` | Is the container alive? | Kill + restart container |
> | `readinessProbe` | Is the container ready for traffic? | Remove from Service endpoints (no restart) |
> | `startupProbe` | Has the app finished starting? | Kill + restart (used for slow-start apps) |
>
> "4 times a minute" = `periodSeconds: 15`. `initialDelaySeconds` = delay before first check; `timeoutSeconds` = per-request timeout.

---

**Q34 — DRAG DROP: Deploy Helm charts to RBAC-enabled AKS — sequence**
> You need to deploy charts using Helm and Tiller to AKS in an RBAC-enabled cluster. Which three commands should you run in sequence?
>
> Commands: helm install / kubectl create / helm completion / helm init / helm serve

**A (in order):**
1. **`kubectl create`** — create a Kubernetes ServiceAccount and ClusterRoleBinding for Tiller (required in RBAC-enabled clusters so Tiller has permission to manage resources)
2. **`helm init`** — initialize Helm and install Tiller into the cluster, using the service account created above (`--service-account tiller`)
3. **`helm install`** — deploy the Helm chart to the cluster

Eliminated: `helm completion` generates shell tab-completion scripts (unrelated to deployment); `helm serve` starts a local chart repository server (unrelated to cluster setup).

> **Exam tip:** This is Helm v2 (Tiller-based). RBAC-enabled clusters require the kubectl create step first so Tiller has cluster-admin permissions. Without it, Helm init/install fails with RBAC permission errors. Helm v3 removed Tiller entirely — this question specifically tests the v2 RBAC setup sequence.

---

## Quick Reference: Exam Checklist

### Release Pipeline Fundamentals
- [ ] Release vs Build pipeline — purpose, inputs, outputs
- [ ] Artifact source types: Azure Pipelines build, Azure Artifacts, GitHub, TFVC, Jenkins, Docker
- [ ] 3 deployment trigger types: Continuous Deployment, Scheduled, Manual
- [ ] Deployment to stages: pre/post conditions, artifact filters, branch filters
- [ ] Built-in tasks: AzureWebApp, SqlAzureDacpacDeployment, Docker, AzureCLI

### Approvals and Gates
- [ ] Approvals = human gates; configured on stages (classic) or Environments (YAML)
- [ ] Pre-deployment vs post-deployment conditions
- [ ] Gate types: Query Work Items, Invoke REST API, Query Azure Monitor Alerts, Check Azure Policy Compliance, Invoke Azure Function
- [ ] Gate evaluation: periodic polling; all gates must pass in same cycle; timeout behavior

### Environments & Service Connections
- [ ] Service connection types: Azure Resource Manager, Docker Registry, Kubernetes, GitHub, SSH
- [ ] Workload Identity Federation = recommended, no secrets, no expiry
- [ ] YAML Environments: support approvals, checks, branch control, deployment history
- [ ] Deployment groups: for on-premises/VM targets; YAML equivalent = Environment with VM resources

### Deployment Strategies
- [ ] Blue-Green: swap slots, zero downtime, instant rollback
- [ ] Canary: small % first, gradually increase
- [ ] Rolling: replace instances one by one
- [ ] Ring: staged rollout by user/geography tiers
- [ ] Feature flags: decouple deploy from release; Azure App Configuration Feature Manager
- [ ] A/B testing: compare versions for business metrics
- [ ] VIP swap: Azure Cloud Services zero-downtime swap

### Tasks, Templates & Variables
- [ ] Task groups: classic only; bundle tasks; versioned
- [ ] Variable scoping: job > stage > pipeline > variable group; queue-time overrides all
- [ ] Variable groups: shared across pipelines; linkable to Azure Key Vault
- [ ] Stage variables override pipeline variables of the same name
- [ ] Secret variables: masked in logs; must map explicitly to env vars

### Health & Quality
- [ ] Availability tests: URL ping, Standard test, Custom (via App Insights)
- [ ] Azure Load Testing: JMeter-based; pipeline integration; fail on error rate/latency
- [ ] DORA metrics: deployment frequency, change failure rate, MTTR, lead time
- [ ] Service hooks: Teams, Slack, Jenkins, Service Bus, Event Grid, Webhooks
- [ ] GitOps: Git as source of truth; pull-based deployments; separate manifests repo

---

*Sources: [MS Learn Release Strategy Path](https://learn.microsoft.com/en-us/training/paths/az-400-design-implement-release-strategy/) | [AZ-400 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-400)*
