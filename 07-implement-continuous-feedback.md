# AZ-400 Study Notes: Implement Continuous Feedback

**Learning Path:** Implement Continuous Feedback
**Exam Weight:** "Develop an instrumentation strategy" = **5–10%** of exam
**MS Learn Path:** [AZ-400: Implement Continuous Feedback](https://learn.microsoft.com/en-us/training/paths/az-400-implement-continuous-feedback/)
**Modules Covered:** 5

---

## Table of Contents

1. [Implement Tools to Track Usage and Flow](#1-implement-tools-to-track-usage-and-flow)
2. [Develop Monitor and Status Dashboards](#2-develop-monitor-and-status-dashboards)
3. [Share Knowledge Within Teams](#3-share-knowledge-within-teams)
4. [Design Processes to Automate Application Analytics](#4-design-processes-to-automate-application-analytics)
5. [Manage Alerts, Blameless Retrospectives, and a Just Culture](#5-manage-alerts-blameless-retrospectives-and-a-just-culture)
6. [Additional Exam Topics](#6-additional-exam-topics)
7. [Past Exam Scenario Bank](#7-past-exam-scenario-bank)
8. [Quick Reference Checklist](#8-quick-reference-checklist)

---

## 1. Implement Tools to Track Usage and Flow

### Inner Loop vs Outer Loop

| Loop | Scope | Feedback Speed | Examples |
|---|---|---|---|
| **Inner loop** | Local dev cycle | Fast (seconds–minutes) | Unit tests, local build, lint, local debugging |
| **Outer loop** | Deployed system | Slower (minutes–hours) | CI/CD pipeline, integration tests, production monitoring |

**Goal:** Shift feedback left — catch issues in the inner loop before they reach production.

### Continuous Monitoring

**Continuous monitoring** means instrumenting every layer of the application and infrastructure so metrics, logs, and traces are collected constantly — not just during incidents.

**What to monitor:**
- Application health (response time, error rate, dependencies)
- Infrastructure health (CPU, memory, disk, network)
- Pipeline health (build success rate, test pass rate, deployment frequency)
- User behavior (page views, user flows, conversion funnels)

### Azure Monitor Overview

**Azure Monitor** is the unified monitoring platform for Azure. It collects, analyzes, and acts on telemetry from cloud and on-premises environments.

**Key components:**

| Component | Purpose |
|---|---|
| **Metrics** | Numerical time-series data (CPU %, response time ms). Near real-time. |
| **Logs** | Structured/unstructured data stored in Log Analytics workspace. Queryable with KQL. |
| **Alerts** | Rules that trigger notifications or actions when conditions are met. |
| **Action Groups** | Define who gets notified and how (email, SMS, webhook, ITSM, Azure Function) |
| **Workbooks** | Interactive reports combining queries, text, and visualizations |
| **Insights** | Pre-built monitoring experiences (VM Insights, Container Insights, Application Insights) |

### Azure Monitor RBAC Roles

Following the **principle of least privilege**, assign the minimum role needed:

| Role | Permissions | Use when user needs to… |
|---|---|---|
| **Monitoring Reader** | Read-only access to all monitoring data — metrics, logs, alerts, autoscale settings, action groups | View alert activities, view autoscale settings |
| **Monitoring Contributor** | Read + write on all monitoring resources — create/edit dashboards, alert rules, action groups; can query Log Analytics | Create dashboards, search workspace data |
| **Monitoring Metrics Publisher** | Write metrics to Azure resources only | Push custom metrics from an app/service |
| **Log Analytics Reader** | Read Log Analytics workspaces and their data (query logs, view settings) | Query logs without broader monitoring write access |
| **Log Analytics Contributor** | Read + write Log Analytics workspaces (create/edit workspace, run queries, configure data collection) | Manage workspace configuration |

> **Exam tip (tested — least privilege):**
> - User needs to **create dashboards + search workspace data** → **Monitoring Contributor**
> - User needs to **view autoscale settings + view alert activities** (read-only) → **Monitoring Reader**
> - Do NOT assign Monitoring Contributor for read-only tasks; that violates least privilege.

### Log Analytics Workspace

A **Log Analytics workspace** is the storage and query engine for Azure Monitor Logs.

- All logs from Azure resources, VMs, and Application Insights are collected here
- Query with **Kusto Query Language (KQL)**
- Multiple resources can send data to the same workspace (centralized monitoring)
- **Workspace-based Application Insights** (current model) stores all AI telemetry in Log Analytics

> **Exam tip:** Application Insights resources should be **workspace-based** (not classic). Classic AI resources are deprecated. Workspace-based AI stores all telemetry in a Log Analytics workspace, enabling unified queries across logs and traces.

### Connecting a VM to a Log Analytics Workspace (MMA / Enterprise Cloud Monitoring Extension)

The **Microsoft Enterprise Cloud Monitoring** extension (also called the **Log Analytics agent / MMA — Microsoft Monitoring Agent**) connects a VM to a Log Analytics workspace to collect logs and telemetry.

**Two values required to configure the extension:**

| Value | Where to find it | Purpose |
|---|---|---|
| **Workspace ID** | Log Analytics workspace → Settings → Agents | Identifies which workspace to send data to |
| **Workspace Key (Primary/Secondary)** | Log Analytics workspace → Settings → Agents → Primary key | Authenticates the agent to the workspace |

> **Exam tip (tested — CONFLICTING ANSWERS in exam dumps):**
> This question appears with two different answers across exam dumps:
> - **Answer AD (Workspace Key + Workspace ID):** Technically correct for extension *configuration settings* — these are the two values placed in `settings` and `protectedSettings` of the extension definition.
> - **Answer DE (Workspace ID + VM Resource ID):** Reflects what you need to *deploy* the extension operation — the target VM (resource ID) and the workspace to connect to. Some exam sources favor this interpretation.
>
> **Microsoft documentation confirms**: the MMA extension requires `workspaceId` (D) and `workspaceKey` (A) in its configuration. The VM resource ID (E) is the *target* of the deployment command, not a configuration value for the extension itself. Favour **A and D** if you must choose one answer set.

### Log Analytics Workspace CMK Encryption — Setup Sequence (Exam Tested)

To encrypt a Log Analytics workspace using a **customer-managed key (CMK)** from Azure Key Vault, a **dedicated cluster** is required as an intermediary. The sequence is:

| Step | Action | Why |
|---|---|---|
| 1 | **Register the Azure subscription to allow cluster creation** | CMK requires a dedicated cluster; subscription must be registered for this feature |
| 2 | **Create a Log Analytics cluster** | The dedicated cluster gets a system-assigned managed identity automatically |
| 3 | **Grant permissions to the key vault** | The cluster's managed identity needs `Get`, `Wrap Key`, `Unwrap Key` on the KV encryption key |
| 4 | **Link the workspace** | Associates the Log Analytics workspace to the cluster so all data is encrypted with the CMK |

> **Exam tip (DRAG DROP tested):** "Enable soft delete for the key vault" is a Key Vault prerequisite — it is NOT one of the 4 steps in the CMK workspace setup sequence. You cannot directly link a workspace to Key Vault — the dedicated cluster is the required intermediary.

### Service Map Solution — Dependency Agent

**Service Map** is an Azure Monitor solution (deployed from Azure Marketplace) that automatically discovers application components and maps communication between services and processes on a VM.

**Agents required for Service Map:**

| Agent | Purpose | Required? |
|---|---|---|
| **Log Analytics agent (MMA)** | Transmits collected data to Log Analytics workspace | ✅ Required (must already be present) |
| **Dependency agent** | Captures network connections, active ports, processes, and inter-service dependencies | ✅ Must be deployed additionally |

The Dependency agent **cannot function without the Log Analytics agent** — it uses MMA to send its data to the workspace.

**Other agents (not suitable for Service Map):**

| Agent | Purpose | Why not for Service Map |
|---|---|---|
| **Telegraf agent** | Custom metrics collection for Azure Monitor / InfluxDB | No dependency/network mapping |
| **Azure Monitor agent (AMA)** | Modern replacement for MMA (logs/metrics) | Does not provide Service Map dependency data |
| **WAD extension** | Diagnostics data from Azure VMs | No dependency mapping capability |

> **Exam tip:** If a VM already has the Log Analytics agent installed and you need to deploy Service Map → add the **Dependency agent**. Service Map requires both agents; MMA is already present, so only the Dependency agent is missing.

### Application Insights

**Azure Application Insights** is an APM (Application Performance Monitoring) service built on top of Azure Monitor. It monitors live applications.

**What it monitors:**
- Request rates, response times, failure rates
- Dependency calls (database, external APIs)
- Exceptions and stack traces
- Page views and user flows
- Custom events and metrics
- Availability (URL ping tests)
- Performance counters (CPU, memory on server)

### Application Insights Instrumentation

**Two ways to instrument:**

| Method | Description | When to Use |
|---|---|---|
| **SDK-based (code)** | Add Application Insights SDK NuGet/npm package to app code | Full control, custom events, highest fidelity |
| **Auto-instrumentation (agent)** | Attach agent without code changes | Legacy apps, no source access, quick setup |

**Connection string** (modern, preferred) vs Instrumentation Key (legacy):

```csharp
// Modern: connection string in appsettings.json
"ApplicationInsights": {
  "ConnectionString": "InstrumentationKey=xxx;IngestionEndpoint=https://..."
}

// Legacy: instrumentation key only
"ApplicationInsights": {
  "InstrumentationKey": "your-guid-here"
}
```

> **Exam tip:** Use **connection string** (not just instrumentation key) for new Application Insights implementations. Connection strings include the ingestion endpoint for regional deployments and are more resilient.

### Application Insights Telemetry Data Types

| Telemetry Type | Description | Table in Log Analytics |
|---|---|---|
| **Requests** | Incoming HTTP requests to the app | `requests` |
| **Dependencies** | Outgoing calls to databases, APIs, queues | `dependencies` |
| **Exceptions** | Handled and unhandled exceptions | `exceptions` |
| **Traces** | Diagnostic log messages | `traces` |
| **Page Views** | Client-side page load events | `pageViews` |
| **Custom Events** | Business events tracked with `TrackEvent()` | `customEvents` |
| **Custom Metrics** | Business metrics tracked with `TrackMetric()` | `customMetrics` |
| **Availability** | Results from availability (ping) tests | `availabilityResults` |
| **Performance Counters** | CPU, memory, etc. from server | `performanceCounters` |

> **Exam tip:** Know all telemetry types and their corresponding Log Analytics table names. Exam questions often ask "which telemetry type captures X" or "which table do you query for Y".

### Sampling in Application Insights

**Sampling** reduces the volume of telemetry sent to Application Insights to control costs and data volume.

| Type | Description |
|---|---|
| **Adaptive sampling** | Automatically adjusts sampling rate based on traffic volume (default for .NET Core SDK) | SDK/code — requires redeployment to change |
| **Fixed-rate sampling** | You set a fixed percentage (e.g., 10% of requests) | ApplicationInsights.config / code — requires redeployment |
| **Ingestion sampling** | Applied at the AI service after data arrives — discards excess at ingestion endpoint | **Portal only (Usage and estimated costs)** — ✅ no redeployment needed |

> **Exam tip:** Adaptive sampling is the default for .NET Core. Ingestion sampling reduces stored data but does NOT reduce SDK/network traffic (app still sends everything). Fixed-rate sampling reduces both.
>
> **Exam tip (HOTSPOT tested):** "Modify sampling rate without redeploying the app" → use **Ingestion sampling**:
> 1. From code repo: **Disable adaptive sampling** (so all telemetry flows to AI — one-time code change)
> 2. From AppInsights1: **Modify the Usage and estimated costs settings** (where ingestion sampling % is set — portal-only, no redeployment ever needed)

### Kusto Query Language (KQL)

KQL is the query language for Azure Monitor Logs and Application Insights.

> **Exam tip:** "Write ad-hoc queries against Azure SQL Database Intelligent Insights and Azure Application Insights monitoring data" → **Azure Log Analytics (KQL)**. Both services send diagnostic data to a Log Analytics workspace; KQL is the query language used there. T-SQL is for querying SQL databases directly. PL/pgSQL and PL/SQL are PostgreSQL and Oracle procedural languages — unrelated to Azure monitoring.

**Basic syntax:**
```kql
// Table | filter | transform | output
requests
| where timestamp > ago(1h)
| where success == false
| summarize count() by bin(timestamp, 5m)
| order by timestamp asc
```

**Key operators:**

| Operator | Purpose | Example |
|---|---|---|
| `where` | Filter rows | `where resultCode == 500` |
| `summarize` | Aggregate | `summarize avg(duration) by name` |
| `project` | Select columns | `project name, duration, success` |
| `extend` | Add computed column | `extend durationSec = duration / 1000` |
| `order by` | Sort results | `order by duration desc` |
| `take` / `limit` | Limit rows | `take 100` |
| `join` | Join two tables | `requests \| join exceptions on operation_Id` |
| `render` | Visualize output | `render timechart` |
| `bin()` | Time bucket grouping | `bin(timestamp, 1h)` |
| `ago()` | Relative time | `ago(24h)` |

**Common exam KQL patterns:**

```kql
// Find failed requests in the last 24 hours
requests
| where timestamp > ago(24h) and success == false
| summarize failCount = count() by name
| order by failCount desc

// P95 response time per operation
requests
| summarize percentile(duration, 95) by name

// Exception count by type
exceptions
| summarize count() by type
| order by count_ desc

// Availability by location
availabilityResults
| summarize successRate = countif(success == true) * 100.0 / count() by location
```

### Azure Load Testing Integration with Application Insights

**Azure Load Testing** runs high-scale load tests against applications. Integrating with Application Insights:
- Correlate load test results with Application Insights telemetry
- Identify performance bottlenecks under load
- View server-side metrics during load test run
- Set pass/fail criteria based on response time thresholds

---

## 2. Develop Monitor and Status Dashboards

### Dashboard Tools Comparison

| Tool | Best For | Key Capabilities |
|---|---|---|
| **Azure Dashboards** | Operational monitoring, quick Azure resource overview | Pin charts from any Azure service; shared with team |
| **Azure Monitor Workbooks** | Interactive reports with parameters, queries, visualizations | Parameterized, mix text + KQL + charts; shareable |
| **Power BI** | Executive reporting, advanced analytics, cross-data-source | Connect to Log Analytics, rich visuals, publish to org |
| **Custom Application** | Specialized needs, embedded dashboards | Azure DevOps REST API + Application Insights SDK |

### Azure Dashboards

- Created in Azure portal — pin any chart, metric, log query result, or resource to a dashboard
- **Shared dashboards** can be shared with other users in the Azure subscription
- Support for RBAC — control who can view vs. edit
- Tiles: metrics charts, log query results, resource health, markdown text

### Azure Monitor Workbooks

- **Interactive** — parameterized with dropdowns, time pickers, text inputs
- Combine: KQL queries + text/markdown + charts + grids + metrics
- **Templates** available for common scenarios (performance, failures, usage)
- **Linked workbooks** — one workbook can link to another for drill-down
- Stored in Azure Monitor → Workbooks

> **Exam tip:** Workbooks are better than dashboards for **interactive analysis with parameters**. Dashboards are for static real-time overviews.

### Power BI for DevOps

- Connect Power BI to **Azure Monitor / Log Analytics** using the Log Analytics connector
- Connect to **Azure DevOps Analytics** views for work item and pipeline reporting
- **Azure DevOps connector** in Power BI: query boards, backlogs, pipeline metrics
- Best for: stakeholder reporting, release dashboards, historical trend analysis

### Monitoring in GitHub

| Feature | Description |
|---|---|
| **Status badges** | Embed workflow status badge in README: `![CI](https://github.com/{owner}/{repo}/actions/workflows/{workflow}/badge.svg)` |
| **GitHub Insights** | Code frequency, pulse, contributor activity, traffic |
| **Actions analytics** | Workflow run history, duration trends, failure patterns |
| **Dependabot alerts** | Security vulnerability notifications |

### Azure Boards Dashboard Widgets — Flow Metrics (Exam Favorite)

Azure Boards provides built-in widgets for tracking work item flow. The exam frequently tests the distinction between **Cycle Time** and **Lead Time**:

| Widget | Measures | Clock starts | Clock ends |
|--------|----------|-------------|-----------|
| **Cycle Time** | Time work is actively being done | Work item moves to **active/in-progress** state | Work item **closed** |
| **Lead Time** | Total elapsed time from request to delivery | Work item **created** | Work item **closed** |
| **Velocity** | Amount of work completed per sprint | N/A | N/A — counts story points/items per iteration |
| **Cumulative Flow** | Volume of work across workflow states over time | N/A | N/A — shows flow diagram |

> **Exam tip:** "Time from when **work starts**" = **Cycle time** (excludes queue/wait time before work begins). "Time from when **work is created**" = **Lead time** (includes all wait time). These are frequently confused in exam questions — cycle time is the active-work measurement.

### DevOps KPIs — Full Reference (Exam Favorite)

| KPI | Definition | Exam Phrase |
|-----|-----------|-------------|
| **Cycle time** | Time from when work **starts** → closed | "How long it takes to complete a work item" |
| **Lead time** | Time from work item **created** → closed | "Total time from request to delivery" |
| **Defect escape rate** | `(production bugs / total bugs) × 100%` — % of defects that escape testing and reach production | "**Percentage** of defects found in production" |
| **Bug report rates** | Volume/frequency of bug reports filed over time | "How many bugs are being reported" (count, not %) |
| **Application failure rates** | Rate of application failures/crashes in production | "How often the app fails" |
| **Deployment speed** | Time to deploy a release to an environment | "How fast deployments complete" |
| **Mean time to recover (MTTR)** | Time to restore service after an incident | "Time to recover from failure" |
| **Burndown trend** | Rate at which work is completed toward sprint/release goal | "Sprint progress against plan" |

> **Exam tip:** **Defect escape rate** = *percentage* of defects found in production. **Bug report rates** = *count/frequency* of reports. When the question says "percentage of defects found in production," the answer is always Defect escape rate.

---

### Pipeline Health Metrics (Exam Favorite)

Key pipeline metrics to monitor and optimize:

| Metric | What It Measures | Why It Matters |
|---|---|---|
| **Build success rate** | % of builds that pass | Low rate = unstable codebase or broken tests |
| **Build duration** | Time per build | Long builds slow feedback; target < 10 min |
| **Test pass rate** | % of tests passing | Failing tests = blocked releases |
| **Flaky tests** | Tests that pass/fail non-deterministically | Create false confidence; must be identified and fixed |
| **Deployment frequency** | How often you deploy to production | DORA metric; higher = better |
| **Mean time to recovery (MTTR)** | Time to restore after failure | DORA metric; lower = better |
| **Change failure rate** | % of deployments causing incidents | DORA metric; lower = better |

### Flaky Tests

A **flaky test** is a test that yields different results (pass/fail) without any code change — non-deterministic.

**Causes:** Race conditions, network timeouts, test order dependencies, hardcoded dates/times.

**Impact:** Teams start ignoring test failures → missed real bugs → reduced pipeline trust.

**Solutions:**
- Retry logic (quarantine flaky tests)
- Azure Test Plans: marks tests as flaky automatically
- Fix the root cause (eliminate non-determinism)

> **Exam tip:** Azure Pipelines can detect flaky tests automatically using **flaky test detection** in the test result report. Tests marked flaky don't fail the build but are flagged for investigation.

**Flaky tests vs Test Impact Analysis (TIA) — exam distinction:**

| Feature | Purpose | Solves flaky/intermittent failures? |
|---|---|---|
| **Flaky test detection / management** | Identifies and quarantines non-deterministic tests | ✅ Yes — directly addresses tests that fail intermittently without code changes |
| **Test Impact Analysis (TIA)** | Selects only tests *affected by a code change* to run | ❌ No — optimizes test selection based on code changes; does nothing for random/flaky failures |

> **Exam tip:** If test failures are "unrelated to source code changes and the execution environment" → this is the definition of a **flaky test**. The solution is **flaky test detection/management**, NOT Test Impact Analysis. TIA is for reducing test run time by only running relevant tests.

### Pipeline Optimization Strategies

| Strategy | Technique |
|---|---|
| **Reduce build time** | Parallel jobs, cache dependencies, incremental builds |
| **Reduce cost** | Use self-hosted agents for long-running jobs, scale down idle agents |
| **Improve reliability** | Retry failing steps, set appropriate timeouts |
| **Concurrency optimization** | Balance parallel jobs vs. agent costs; use demands to route jobs |

---

## 3. Share Knowledge Within Teams

### Knowledge Sharing Strategy

**Why it matters:**
- Prevents "tribal knowledge" — critical info known only to specific individuals
- Reduces onboarding time for new team members
- Prevents repeating past mistakes
- Enables asynchronous collaboration across time zones

**Tools:**
- Azure DevOps Wikis
- GitHub documentation (README, GitHub Pages, GitHub Wiki)
- Microsoft Teams channels + integrations
- Confluence, Notion, SharePoint (third-party)

### Azure DevOps Wikis

Two types:

| Type | Description | Storage |
|---|---|---|
| **Project Wiki** | Auto-provisioned wiki for a project. Each project gets one. | Stored as a hidden Git repo (`{project}.wiki`) |
| **Code Wiki** | Publish a folder from an existing Git repo as a wiki | Stored in your code repo |

> **Exam tip:** **Project Wiki** is automatically created per project. **Code Wiki** lets you publish documentation that lives alongside your code — for example, `/docs` folder in your repo.

**Project Wiki key facts:**
- Supports Markdown formatting
- Pages organized in a tree hierarchy
- Version-controlled (every edit is a commit)
- Searchable within Azure DevOps
- Supports **Mermaid diagrams** (flowcharts, sequence diagrams, ERDs) inline in Markdown

**Wiki page ordering:**

| Wiki Type | How to reorder pages in the portal |
|---|---|
| **Project Wiki** (provisioned) | **Drag and drop** pages in the navigation pane |
| **Published Wiki** (Code Wiki) | **Drag and drop** pages in the navigation pane — portal commits a `.order` file to the Git repo automatically |

> **Exam tip:** For a **published wiki**, reordering pages in the Azure DevOps portal is done by **drag and drop**. The portal handles the underlying `.order` file update in the Git repository. Creating a file named `order` (option A) is a distractor — the actual file is `.order` and you don't need to create it manually via the portal.

**Mermaid diagram in wiki:**
```markdown
:::mermaid
graph LR
  A[Feature Branch] --> B[Pull Request]
  B --> C{Review}
  C -->|Approved| D[Merge to Main]
  C -->|Changes Required| A
:::
```

### Markdown Essentials for Wikis

```markdown
# Heading 1
## Heading 2

**Bold text**
*Italic text*
`inline code`

```code block```

| Column1 | Column2 |
|---|---|
| Value | Value |

- Unordered list
1. Ordered list

[Link text](URL)
![Image alt](image-url)

> Blockquote / callout
```

### GitHub Documentation

- **README.md** — primary entry point for any GitHub repo
- **GitHub Pages** — host static sites from repo content (documentation, API docs)
- **GitHub Wiki** — per-repo wiki, simpler than Azure DevOps wiki
- **`.github/` folder** — special directory for PR templates, issue templates, CODEOWNERS, workflows

**GitHub status badge in README:**
```markdown
![Build Status](https://github.com/{owner}/{repo}/actions/workflows/{workflow-file}/badge.svg)
```

### Microsoft Teams Integration

**GitHub + Microsoft Teams:**
- Install **GitHub app for Teams** (or use incoming webhooks)
- Subscribe to notifications: push, PR, issue, release events
- Use `/github subscribe {owner}/{repo}` in Teams channel
- Discuss PRs directly in Teams without leaving the app

**Azure DevOps + Microsoft Teams — Two distinct apps:**

| App | Purpose |
|---|---|
| **Azure Pipelines app for Teams** | CI/CD notifications — build completions, release deployments, pipeline failures |
| **Azure Boards app for Teams** | Work item tracking — PR status, work item updates, board queries |

- Install the correct app from the Teams App Store
- Configure a **subscription** in a Teams channel to receive notifications
- Use `@AzureDevOps` to create work items directly from Teams

**Integration commands (Teams):**
```
# Subscribe to build notifications (Azure Pipelines app)
@AzureDevOps subscribe https://dev.azure.com/{org}/{project}

# Filter: only failed builds
@AzureDevOps subscribe https://dev.azure.com/{org}/{project}/_build
@AzureDevOps subscriptions [filter: status=failed]
```

> **Exam tip (tested):** To notify a Teams channel on build/release **failures** from Azure Pipelines with **minimum development effort** → install the **Azure Pipelines app for Teams** and configure a subscription in a channel. Do NOT confuse with the Azure Boards app (work items) or custom solutions like Azure Functions/Automation (too much dev effort).

---

## 4. Design Processes to Automate Application Analytics

### Automated Analytics Design

**Goal:** Move from reactive (investigate after incident) to proactive (detect issues before users notice).

**Design principles:**
1. **Instrument everything** — telemetry from every layer (client, server, infrastructure)
2. **Automate analysis** — use smart detection and anomaly detection, not manual review
3. **Close feedback loops** — connect monitoring alerts back to the work tracking system
4. **Augmented search** — use AI/ML to surface relevant telemetry quickly during incidents

### Augmented Search / Rapid Response

Traditional log search requires knowing what to look for. **Augmented search** (AI-assisted) surfaces relevant signals automatically:

- Application Insights **Smart Detection** — ML-based anomaly detection on telemetry
- Azure Monitor **AI-powered alerts** — detect unusual patterns without manual thresholds
- **Log Analytics anomaly detection** functions in KQL: `series_decompose_anomalies()`

**Rapid response workflow:**
1. Alert fires automatically (smart detection or threshold alert)
2. Alert links to relevant telemetry (Application Map, Exception details, Profiler)
3. Engineer investigates with pre-built workbooks or live metrics
4. Work item created in Azure Boards via ITSM connector or automation

### Telemetry Integration

**Telemetry pipeline:**
```
Application → Application Insights SDK → Log Analytics Workspace → Alerts / Workbooks / Power BI
Infrastructure → Azure Monitor Agent → Log Analytics Workspace
```

**Custom telemetry:**
```csharp
// Track custom event
_telemetryClient.TrackEvent("UserPurchased", new Dictionary<string, string> {
    { "ProductId", productId },
    { "Amount", amount.ToString() }
});

// Track custom metric
_telemetryClient.TrackMetric("CartSize", cartItems.Count);

// Track dependency manually
var operation = _telemetryClient.StartOperation<DependencyTelemetry>("CustomCall");
// ... call external service ...
_telemetryClient.StopOperation(operation);
```

### Application Insights Usage Analytics Tools

Application Insights includes usage analytics tools to understand how users interact with your application:

| Tool | What it analyzes | Exam clue |
|---|---|---|
| **Users** | Counts distinct users, sessions, and events | "Number of people who used actions/features" |
| **User Flows** | Visualizes navigation paths users take through pages/features; shows drop-offs and loops | "Feature usage" / "how users navigate" |
| **Impact** | Correlates performance (load times, response times) with conversion rates and feature usage | "Effect of performance on usage of a page/feature" |
| **Funnels** | Tracks how users progress through a defined sequence of steps (conversion funnel) | "Conversion rate through checkout steps" |
| **Retention** | Measures how many users return after their first visit | "User retention after first use" |
| **Cohorts** | Groups users/sessions sharing a common characteristic for comparative analysis | "Compare behavior of users who did X vs didn't" |

> **Exam tip (DRAG DROP):** Feature usage → **User Flows** | Number of people who used actions → **Users** | Effect of performance on usage → **Impact**

### Application Insights Application Map

The **Application Map** visualizes the topology of your application — all components and their dependencies:
- Each node = a component (web app, microservice, database, external API)
- Edges = dependency calls between components
- Color-coded: green (healthy), yellow (warnings), red (failures)
- Click a node to drill into metrics/failures for that component

> **Exam tip:** Application Map is the go-to tool for identifying **which component** in a distributed application is causing failures or performance degradation.

### Monitoring Tools Selection Guide

| Scenario | Recommended Tool |
|---|---|
| Monitor a .NET web app end-to-end | Application Insights (SDK) |
| Monitor Azure VM / on-premises server | Azure Monitor Agent + Log Analytics |
| Monitor Kubernetes clusters | Container Insights (Azure Monitor for Containers) |
| Monitor Azure SQL Database | Azure SQL Insights |
| Query logs with custom analysis | Log Analytics + KQL |
| Executive-level reports | Power BI with Log Analytics connector |
| Real-time server metrics during load test | Application Insights Live Metrics |
| Multi-cloud / third-party monitoring | Azure Monitor integration (Prometheus, Grafana) |

### IT Service Management (ITSM) Connector

The **ITSM Connector** (also called ITSMC) integrates Azure Monitor with external IT service management tools.

**Supported ITSM tools:** ServiceNow, System Center Service Manager, Provance, Cherwell

**What it does:**
- When an Azure Monitor alert fires → automatically creates an **incident or event** in the ITSM tool
- Bidirectional sync: alert status changes in Azure → update in ITSM

**Configuration:**
1. Install the Microsoft Log Analytics integration app from ServiceNow store
2. Create an ITSM Connection in Azure Monitor (Connections → Add)
3. In an Azure alert rule, add an **ITSM action** in the action group

> **Exam tip:** ITSM actions are configured in **Action Groups**. When an alert fires, the action group calls the ITSM connector, which creates a work item (incident) in ServiceNow or other tools.

> **Important note:** Microsoft deprecated ITSM Connector for new deployments starting 2023. The current recommended approach is **Azure Monitor secure webhook** to ServiceNow. The exam may still reference ITSMC — know its purpose and setup.

---

## 5. Manage Alerts, Blameless Retrospectives, and a Just Culture

### Azure Monitor Alert Types

| Alert Type | What It Monitors | Source |
|---|---|---|
| **Metric alerts** | Threshold on a numeric metric (CPU > 80%) | Azure Monitor Metrics |
| **Log search alerts** | KQL query result meets condition | Log Analytics workspace |
| **Activity log alerts** | Azure resource operations (VM started, policy change) | Azure Activity Log |
| **Smart detection alerts** | AI-detected anomalies in Application Insights | Application Insights |
| **Availability alerts** | Application Insights availability test failures | Application Insights |

### Alert Components

An alert rule has three parts:

| Part | Purpose |
|---|---|
| **Condition** | What triggers the alert (metric threshold, log query) |
| **Action Group** | What happens when the alert fires (who gets notified, how) |
| **Alert details** | Name, severity (0-4), description |

**Alert severity levels:**
| Severity | Label |
|---|---|
| 0 | Critical |
| 1 | Error |
| 2 | Warning |
| 3 | Informational |
| 4 | Verbose |

### Action Groups

An **Action Group** defines the notification and automation actions taken when an alert fires.

**Action types:**

| Action Type | Description |
|---|---|
| **Email/SMS/Push/Voice** | Notify individuals or on-call groups |
| **Azure Function** | Trigger a serverless function for auto-remediation |
| **Logic App** | Run a workflow (e.g., create ITSM ticket, send Teams message) |
| **Webhook** | HTTP POST to any URL |
| **ITSM** | Create incident in ServiceNow, SCSM, etc. |
| **Automation Runbook** | Trigger Azure Automation runbook |
| **Event Hubs** | Stream alert data to event hub for processing |

> **Exam tip:** Action Groups are **reusable** — the same action group can be referenced by multiple alert rules. Create action groups once, reuse across all relevant alerts.

### Application Insights Smart Detection

**Smart Detection** uses machine learning to automatically detect anomalies in your application's telemetry — no manual threshold configuration required.

**What it detects:**
- Abnormal rise in **failed request rate**
- Abnormal rise in **dependency failure rate**
- **Response time degradation** (server response time suddenly increases)
- Memory leak patterns
- Abnormal rise in exceptions

**How it works:**
- Baseline: 24 hours to learn normal behavior after first data received
- Continuously monitors deviations from the learned baseline
- Sends alerts when significant anomalies detected
- Alerts include: summary, affected operations, diagnostic charts

> **Exam tip:** Smart Detection takes **~24 hours** to establish the baseline before it can fire alerts. It monitors failure rates and response times automatically without configuring thresholds.

### Availability Tests

Application Insights **Availability Tests** monitor whether your application is reachable from different geographic locations.

| Test Type | Description |
|---|---|
| **URL ping test (classic)** | Simple HTTP GET to a URL; check response code and optional content |
| **Standard test** | HTTP GET/POST/HEAD with custom headers, SSL validation, redirect handling |
| **Custom TrackAvailability** | Use Azure Function or code to define complex availability checks |
| ~~Multi-step web test~~ | **Deprecated** (Visual Studio recorded tests) |

**Test configuration:**
- **Frequency:** 5 minutes (default) to 15 minutes
- **Test locations:** Up to 16 geographic locations (e.g., East US, West Europe, Southeast Asia)
- **Success criteria:** HTTP status code, response content, response time
- **Alert:** Auto-creates an alert rule when test is created

> **Exam tip:** The old **multi-step web test** (requires Visual Studio Enterprise to record) is deprecated. Use **Custom TrackAvailability** via Azure Functions for multi-step tests.

### Server Response Time Degradation

Application Insights smart detection flags **server response time degradation** when:
- Average response time for an operation increases significantly compared to the baseline
- The degradation is statistically significant (not random noise)
- It affects a meaningful percentage of requests

**Diagnosing response time degradation:**
1. Application Insights → Performance blade → drill into slow operations
2. **Profiler traces** — sample execution traces showing exactly which code is slow
3. **End-to-end transaction details** — see all dependency calls for a slow request
4. **Application Map** — identify which component is the bottleneck

### Reducing Alert Fatigue (Meaningless Alerts)

**Alert fatigue** = teams ignore alerts because too many fire, most being noise.

**Causes of meaningless alerts:**
- Thresholds set too low (triggers on normal fluctuations)
- Alerts that don't require action (informational only)
- Duplicate alerts from multiple monitoring systems
- No alert routing (everyone gets all alerts)

**Solutions:**

| Solution | Description |
|---|---|
| **Use smart detection** | ML-based; reduces false positives vs. manual thresholds |
| **Set appropriate thresholds** | Base thresholds on observed normal behavior, not arbitrary values |
| **Alert suppression / maintenance windows** | Suppress alerts during planned maintenance |
| **Dynamic thresholds** | Use Azure Monitor's dynamic (ML-based) thresholds instead of static |
| **Action group routing** | Route different severity alerts to different teams |
| **Consolidate duplicate alerts** | Single source of truth for monitoring |

> **Exam tip:** **Dynamic thresholds** in Azure Monitor automatically adapt to seasonal patterns and trends — they reduce false positives better than static thresholds.

**Automated response pattern — minimize administrative overhead:**

When an alert fires, the action group can trigger an **Azure Automation Runbook** to automatically remediate without human intervention. This is the exam-tested pattern for "minimize administrative overhead":

| Requirement | Resource to use |
|---|---|
| Detect "exceeds normal usage patterns" | **Azure Monitor alert with dynamic threshold** (ML-learned baseline — no manual config) |
| Automatically change app configuration (e.g., logging level) | **Azure Automation Runbook** (triggered by alert action group) |

- **Email action** (action group with email) just notifies — a human must still act → does NOT minimize overhead.
- **Static threshold** requires manual baseline definition → higher admin overhead than dynamic.
- **Auto scale settings** adjust instance count, not app configuration like logging level.

### Blameless Retrospectives

A **blameless retrospective** (also called a **post-mortem** or **incident review**) focuses on **what went wrong with the system**, not who made a mistake.

**Key principles:**
- Engineers acted with best intentions given the information they had
- Focus on **systemic factors** (missing safeguards, poor tooling, unclear runbooks)
- Document: timeline, impact, root cause, contributing factors, action items
- Action items must be specific, assigned, and tracked

**Blameless retrospective structure:**
1. **Summary** — what happened, impact, duration
2. **Timeline** — sequence of events with timestamps
3. **Root cause analysis** — 5 Whys or fault tree analysis
4. **Contributing factors** — what made the incident worse or harder to detect
5. **Action items** — specific improvements with owners and due dates

> **Exam tip:** Blameless retrospectives are a cultural practice — the goal is **learning**, not assigning blame. Punishing mistakes causes engineers to hide issues, making systems less safe.

### Just Culture

**Just culture** is an organizational culture where:
- People feel safe to report mistakes, near-misses, and problems
- Accountability is about **learning and improvement**, not punishment
- **System design** is prioritized over individual fault
- Engineers trust that admitting errors leads to improvement, not termination

**Just culture vs blame culture:**

| Just Culture | Blame Culture |
|---|---|
| Errors reported openly | Errors hidden to avoid punishment |
| Systems improved after incidents | Individual punished; system unchanged |
| Engineers feel psychologically safe | Engineers feel fear and anxiety |
| Root cause analysis finds systemic issues | Root cause analysis stops at "human error" |
| Builds organizational resilience | Repeats incidents |

> **Exam tip:** The AZ-400 exam tests the *concept* of blameless retrospectives and just culture as essential DevOps practices. Questions may ask "what is the primary goal of a blameless retrospective?" → **To improve systems and processes, not to assign blame.**

---

## 6. Additional Exam Topics

### Application Insights Live Metrics

**Live Metrics Stream** provides near-real-time telemetry with < 1 second latency:
- Incoming request rate, failure rate, response time
- Outgoing dependency calls
- CPU, memory on the server
- **No data stored** — live stream only (not in Log Analytics)

> **Exam tip:** Use Live Metrics during **active incidents or load tests** to see real-time impact. It does NOT persist data — it's a live view only.

### Application Insights Snapshot Debugger

**Snapshot Debugger** automatically captures a diagnostic snapshot when an exception occurs:
- Captures local variable values at the time of the exception
- No code change needed (agent-based)
- Snapshots viewable in Visual Studio or Azure portal
- Useful for diagnosing production exceptions without reproducing them locally

### Application Insights Profiler

**Application Insights Profiler** captures detailed execution traces (call stacks) for slow requests:
- Identifies exactly which line of code is slow
- Minimal performance overhead (sampling-based)
- Works for ASP.NET, .NET Core, Java apps
- Results viewable in Application Insights → Performance blade

### DORA Metrics and Continuous Feedback

The **DORA (DevOps Research and Assessment) metrics** measure DevOps performance — all require continuous feedback/monitoring:

| Metric | Measurement | Monitoring Tool |
|---|---|---|
| **Deployment Frequency** | How often you deploy to production | Azure DevOps Analytics, pipeline metrics |
| **Lead Time for Changes** | Commit to production time | Azure DevOps Analytics, Application Insights |
| **Mean Time to Recovery (MTTR)** | Time to restore after incident | Alert timestamps, incident tracking |
| **Change Failure Rate** | % of deployments causing incidents | Pipeline + Application Insights alerts |

### Azure Monitor Alerts — Alert Processing Rules

**Alert processing rules** (formerly called alert suppression rules) can:
- Suppress notifications during maintenance windows
- Override action groups for certain alerts
- Apply to multiple alert rules at once

```
Scenario: Suppress all alerts between 2 AM – 4 AM on Sundays (maintenance window)
Solution: Create an Alert Processing Rule with a schedule to suppress during that window
```

### Grafana Integration with Azure Monitor

**Azure Managed Grafana** integrates Azure Monitor and Application Insights data with Grafana dashboards:
- Azure Monitor data source built-in
- Prometheus metrics support
- Useful for multi-cloud or team-preference scenarios

### Microsoft Defender for DevOps

**Microsoft Defender for DevOps** (part of Defender for Cloud) provides:
- Security posture management for DevOps environments
- Scans IaC templates (ARM, Bicep, Terraform) for misconfigurations
- Integrates with Azure DevOps and GitHub
- Exposes security findings in Azure portal and in PRs

> **Exam tip:** Defender for DevOps surfaces **IaC security issues** in the pipeline and as PR annotations — it provides shift-left security for infrastructure code.

### Microsoft Test & Feedback Extension — Access Levels

The **Microsoft Test & Feedback** extension (browser-based, installed in Chrome/Edge) enables stakeholders and testers to provide structured feedback on features.

**Access levels required (principle of least privilege):**

| Role | Required Access Level | Reason |
|---|---|---|
| **Developers** | **Basic** | Need to *request* feedback, view responses, file bugs/tasks from feedback sessions |
| **Pilot users / Testers** | **Stakeholder** | Only need to *provide* feedback when requested — free Stakeholder access is sufficient |

> **Exam tip:** Stakeholder access is **free** and sufficient for anyone who only *responds* to feedback requests. Only those who *initiate and manage* feedback workflows (developers, test leads) need Basic access. Giving pilot users Basic would violate least privilege.

---

### SRE (Site Reliability Engineering) Concepts

| Concept | Definition |
|---|---|
| **SLI (Service Level Indicator)** | A specific measurable metric (e.g., request success rate %) |
| **SLO (Service Level Objective)** | Target value for an SLI (e.g., 99.9% success rate) |
| **SLA (Service Level Agreement)** | Contractual commitment to customers based on SLOs |
| **Error Budget** | Allowable downtime/errors before SLO is violated. `Error Budget = 1 - SLO` |
| **Toil** | Manual, repetitive operational work with no lasting value — should be automated |

> **Exam tip:** Error budget = how much downtime/error you can afford before breaching your SLO. If error budget is exhausted, freeze feature releases until reliability is restored.

---

## 7. Past Exam Scenario Bank

**Q1.** Your team has integrated Application Insights into an ASP.NET Core web application. Users report intermittent slowness but you cannot reproduce it locally. Which Application Insights feature captures detailed execution traces showing exactly which code path is slow?

**A:** **Application Insights Profiler**. It samples production requests and captures call-stack traces showing exactly which methods are slow. Snapshot Debugger captures variable state at exceptions — different tool, different purpose.

---

**Q2.** You configure smart detection for an Application Insights resource. An hour after enabling it, no smart detection alerts fire even though the application is having issues. What is the most likely reason?

**A:** Smart Detection requires approximately **24 hours** to learn the normal baseline behavior of the application. It cannot detect anomalies until a baseline is established. After 24 hours, it will begin monitoring for deviations.

---

**Q3.** You need to query Application Insights to find all failed HTTP requests in the last 6 hours, grouped by the operation name. Write the KQL query.

**A:**
```kql
requests
| where timestamp > ago(6h)
| where success == false
| summarize failedCount = count() by name
| order by failedCount desc
```

---

**Q4.** A developer wants to track when users complete a purchase in the application and include the order value as a metric. Which Application Insights API calls should they use?

**A:**
```csharp
// Track the event with properties
_telemetryClient.TrackEvent("PurchaseCompleted",
    properties: new Dictionary<string, string> { { "ProductId", id } },
    metrics: new Dictionary<string, double> { { "OrderValue", amount } });
```
Or separately: `TrackEvent()` for the event + `TrackMetric()` for the order value metric.

---

**Q5.** Your web application is deployed across three Azure regions. You need to verify the application is accessible from multiple geographic locations and get alerted if it becomes unreachable. What should you configure?

**A:** Configure **Availability Tests** in Application Insights. Use a **Standard test** (or URL ping test) pointing to the application URL, configure multiple test locations (e.g., East US, West Europe, Southeast Asia), set the failure threshold, and enable alerts. This monitors availability from external geographic locations.

---

**Q6.** After a major production incident, your team holds a meeting to assign responsibility and discipline the engineer who made the deployment. What DevOps practice does this violate?

**A:** This violates **blameless retrospective** and **just culture** principles. Assigning blame and punishing individuals discourages engineers from reporting issues, creating a culture of fear. The correct approach is to focus on systemic improvements — what process/tool/safeguard could have prevented the incident regardless of individual actions.

---

**Q7.** Your team wants to embed a build status badge in the GitHub repository README to show the current status of the main branch CI workflow. What is the correct badge URL format?

**A:**
```markdown
![CI Status](https://github.com/{owner}/{repo}/actions/workflows/{workflow-filename}.yml/badge.svg?branch=main)
```

---

**Q8.** You need to send an alert to both a Microsoft Teams channel and create an incident in ServiceNow when a critical Azure Monitor alert fires. What Azure Monitor construct enables this?

**A:** An **Action Group** with two actions: a **webhook** (or Logic App) targeting the Teams channel, and an **ITSM action** targeting the ServiceNow ITSM connector. The action group is attached to the alert rule and executes all actions when the alert fires.

---

**Q9.** Your Log Analytics workspace receives logs from 10 different Azure services. You need to create a report that stakeholders can view in Power BI showing deployment frequency and lead time trends over the past 3 months. How do you connect Power BI to the data?

**A:** Use the **Azure Monitor (Log Analytics) connector** in Power BI. Authenticate to the Log Analytics workspace, write KQL queries in Power BI Query Editor, and build visuals from the returned data. Publish the report to Power BI service for stakeholder access.

---

**Q10.** A pipeline frequently fires a "build failed" alert, but investigation shows the failures are transient (test environment timeouts). Team members have stopped responding to the alerts. What is this problem called and how do you fix it?

**A:** This is **alert fatigue** caused by **meaningless alerts**. Fix by:
1. Identifying the transient failure root cause (fix test environment timeouts)
2. Adding retry logic to the pipeline for transient failures
3. Adjusting the alert threshold or using **dynamic thresholds**
4. Using an alert processing rule to suppress non-actionable patterns
The goal: every alert that fires should require a human action.

---

**Q11.** Your organization has a shared Azure DevOps project. You want to create documentation that lives alongside the code in the repository and is published as a wiki. Which wiki type should you use?

**A:** **Code Wiki**. A Code Wiki publishes a folder (e.g., `/docs`) from an existing code repository as a wiki. Changes to wiki pages are commits to the code repo, keeping documentation version-controlled alongside the code. Project Wiki is separate from code repos.

---

**Q12.** You need to query Application Insights to calculate the 95th percentile response time for each operation name in the last hour.

**A:**
```kql
requests
| where timestamp > ago(1h)
| summarize p95_duration = percentile(duration, 95) by name
| order by p95_duration desc
```

---

**Q13.** Application Insights Smart Detection sends a notification about an abnormal rise in failed requests. The notification shows affected operations and a chart of failure rate vs baseline. What was the root cause detection method used?

**A:** **Machine learning / AI-based anomaly detection**. Smart Detection establishes a baseline of normal failure rates using ML and automatically detects when the failure rate rises significantly above that baseline — no manual threshold configuration is needed.

---

**Q14.** After deploying a new release, your Application Insights shows the server response time for `/api/checkout` increased from 200ms (baseline) to 2000ms. Which Application Insights feature helps you identify which line of code or dependency call is causing the slowness?

**A:** **Application Insights Profiler**. It captures detailed execution traces (call stacks) for slow requests, showing which specific method or dependency call is responsible for the latency. The **Application Map** helps identify which component is involved; the Profiler shows the exact code path.

---

**Q15.** Your team's SLO is 99.9% availability per month. In a 30-day month, how many minutes of downtime does the error budget allow?

**A:**
- Total minutes in 30 days = 30 × 24 × 60 = 43,200 minutes
- Error budget = (1 - 0.999) × 43,200 = 0.001 × 43,200 = **43.2 minutes**

If the error budget is exhausted before month end, feature releases should be paused until reliability is restored.

---

**Q16 (new).** You have a Log Analytics workspace named WS1 and a VM named VM1. You need to install the Microsoft Enterprise Cloud Monitoring extension on VM1. Which two values are required to configure the extension?

A. The secret key of WS1
B. The ID of the subscription
C. The system-assigned managed identity of VM1
D. The ID of WS1
E. The resource ID of VM1

**A: A and D — Workspace Key (secret key) + Workspace ID.**
The Log Analytics agent (MMA) requires the **Workspace ID** to identify the target workspace and the **Workspace Key (primary/secondary key)** to authenticate. Subscription ID, VM managed identity, and VM resource ID are not used by the extension configuration.

---

**Q16.** You have a project in Azure DevOps with User1 and User2. You plan to use Azure Monitor. The solution must follow least privilege.

- User1 needs to: create private monitoring dashboards, search usage data for an Azure Monitor workspace.
- User2 needs to: view autoscale settings, view alert activities and settings.

Which role should you assign to each user?

**A:**
- **User1 → Monitoring Contributor** — needs write access to create dashboards and query workspace data.
- **User2 → Monitoring Reader** — only needs read access to view autoscale settings and alert activities; least-privilege read-only role.

Do not assign Monitoring Contributor to User2 — that grants unnecessary write access, violating least privilege.

---

**Q17.** Your team uses Azure Pipelines to deploy applications. You need to ensure that when a failure occurs during the build or release process, all team members are notified via Microsoft Teams. The solution must minimize development effort. What should you do?

A. Install the Azure Boards app for Teams and configure a subscription.
B. Use Azure Automation to connect to the Azure DevOps REST API and notify the team.
C. Use an Azure function to connect to the Azure DevOps REST API and notify the team.
D. Install the Azure Pipelines app for Teams and configure a subscription to receive notifications in a channel.

**A: D — Install the Azure Pipelines app for Teams and configure a subscription.**

The Azure Pipelines app for Teams is a pre-built integration that natively surfaces build and release events (including failures) directly into a Teams channel via subscriptions — no code required. Options B and C require writing custom code (high dev effort). Option A (Azure Boards app) is for work item tracking, not pipeline events.

---

**Q18.** Your company is building a new web application. You plan to collect feedback from pilot users using the Microsoft Test & Feedback extension installed in Chrome. Using the principle of least privilege, which access levels should you assign in Azure DevOps?

- Developers: [Basic / Stakeholder]
- Pilot users: [Basic / Stakeholder]

**A:**
- **Developers → Basic** (need to request feedback, view responses, file bugs from feedback)
- **Pilot users → Stakeholder** (only need to provide/respond to feedback — Stakeholder is free and sufficient)

---

**Q19.** You manage an Azure web app supporting an e-commerce website. You need to increase the logging level when the web app exceeds normal usage patterns. The solution must minimize administrative overhead. Which two resources should you include?

A. An Azure Automation Runbook
B. An Azure Monitor alert with a dynamic threshold
C. An Azure Monitor alert with a static threshold
D. Azure Monitor auto scale settings
E. An Azure Monitor alert using an action group with an email action

**A: A and B.**
- **B (dynamic threshold alert):** Detects when usage exceeds the ML-learned "normal" baseline automatically — no manual threshold definition needed.
- **A (Automation Runbook):** Triggered by the alert's action group to automatically increase the logging level — no human action required.
- C (static threshold) requires manually defining "normal" — higher admin overhead.
- D (auto scale) adjusts instance count, not logging level.
- E (email action) only notifies — a human must still change the logging level manually, which does not minimize overhead.

---

**Q20.** You have an Azure virtual machine monitored by Azure Monitor. The VM has the Azure Log Analytics agent installed. You plan to deploy the Service Map solution from Azure Marketplace. What should you deploy to the virtual machine to support the Service Map solution?

A. The Telegraf agent
B. The Azure Monitor agent
C. The Dependency agent
D. The Windows Azure diagnostics extension (WAD)

**A: C — The Dependency agent.**
Service Map requires two agents: the Log Analytics agent (MMA) — already installed — and the **Dependency agent**, which captures network connections, active ports, processes, and inter-service dependencies. The Dependency agent sends its data through MMA to the Log Analytics workspace. Telegraf (A) collects custom metrics only. AMA (B) is a modern log/metrics agent with no Service Map dependency mapping. WAD (D) collects diagnostics data with no dependency mapping.

---

**Q21.** You have an Azure pipeline used to deploy a web app. The pipeline includes a test suite (TestSuite1) that validates the web app. TestSuite1 fails intermittently. The failures are unrelated to changes in source code and the execution environment. You need to minimize troubleshooting effort. Solution: You enable Test Impact Analysis (TIA). Does this meet the goal?

A. Yes
B. No

**A: B — No.**
The failures are intermittent and unrelated to code changes — this is the definition of a **flaky test**. The correct solution is **flaky test detection/management** in Azure Pipelines, which identifies and quarantines non-deterministic tests so they don't block the pipeline. **TIA** (Test Impact Analysis) selects which tests to run based on code changes — it has no effect on tests that fail randomly regardless of code changes.

---

**Q22. (HOTSPOT)** You have an Azure web app (webapp1) using .NET Core with an Application Insights resource (AppInsights1). You need to modify the sampling rate of telemetry without redeploying webapp1 after each modification.

- From the code repository of webapp1: [Modify ApplicationInsights.config / **Disable adaptive sampling** / Enable fixed-rate sampling]
- From AppInsights1: [Configure Continuous export / Configure Smart Detection / **Modify the Usage and estimated costs settings**]

**A:**
- **Disable adaptive sampling** (code repo) — adaptive sampling is the .NET Core default; disabling it ensures all telemetry reaches AI so ingestion sampling has full control.
- **Modify the Usage and estimated costs settings** (AppInsights1) — this is where ingestion sampling percentage is set. It is a portal-only change — no redeployment ever needed.

---

**Q23. (DRAG DROP)** You have an Azure Key Vault with an encryption key named key1. You plan to create a Log Analytics workspace encrypted using key1. Which four actions should you perform in sequence?

Actions: Register the Azure subscription to allow cluster creation / Enable soft delete for the key vault / Create a Log Analytics cluster / Grant permissions to the key vault / Link the workspace

**A (in order):**
1. Register the Azure subscription to allow cluster creation
2. Create a Log Analytics cluster
3. Grant permissions to the key vault
4. Link the workspace

"Enable soft delete for the key vault" is excluded — it is a Key Vault prerequisite, not one of the 4 configuration steps. The dedicated cluster is required as intermediary; its managed identity must be granted Key Vault permissions before the workspace can be linked.

---

**Q24.** You have a project in Azure DevOps named Project1. Project1 contains a published wiki. You need to change the order of pages in the navigation pane of the published wiki in the Azure DevOps portal. What should you do?

A. At the root of the wiki, create a file named order that defines the page hierarchy.
B. At the root of the wiki, create a file named wiki.md that defines the page hierarchy.
C. Rename the pages in the navigation pane.
D. Drag and drop the pages in the navigation pane.

**A:** D — **Drag and drop the pages in the navigation pane.** For both provisioned and published wikis, Azure DevOps portal supports drag and drop to reorder pages. For a published (Code) wiki, the portal automatically commits the `.order` file to the Git repository. Option A is a distractor — the underlying file is `.order` (with a leading dot) and you don't create it manually.

---

**Q26.** DRAG DROP — You are implementing a new project in Azure DevOps. You need to assess the performance of the project. The solution must identify: (1) How long it takes to complete a work item, (2) The percentage of defects found in production. Which DevOps KPI should you use for each metric?
> KPIs: Application failure rates / Bug report rates / Burndown trend / Cycle time / Defect escape rate / Deployment speed / Lead time / Mean time to recover

**A:**
- **How long it takes to complete a work item → Cycle time.** Cycle time measures the duration from when work starts to when the item is closed.
- **The percentage of defects found in production → Defect escape rate.** Defect escape rate = (production bugs / total bugs) × 100% — it is a *percentage* metric measuring defects that escaped testing and reached production. Bug report rates (dump answer) is a volume/frequency measure, not a percentage.

> Note: Dump listed "Bug report rates" for the second metric — this is incorrect. "Percentage of defects found in production" = Defect escape rate by definition.

---

**Q25.** You are creating a dashboard in Azure Boards. You need to visualize the time from when work starts on a work item until the work item is closed. Which type of widget should you use?
> A. Cycle time  B. Velocity  C. Cumulative flow  D. Lead time

**A: A — Cycle time.**
Cycle time measures from when **work starts** (item enters active/in-progress state) → closed. Lead time (D) measures from when the work item was **created** → closed, which includes pre-work queue time. Velocity measures work completed per sprint. Cumulative flow shows volume across states over time. The key phrase "when **work starts**" distinguishes cycle time from lead time.

> Note: The dump lists D (Lead time) as the answer — this is incorrect. "Work starts" = active state = cycle time by definition.

---

**Q27.** You need to write ad-hoc queries against Azure SQL Database Intelligent Insights and Azure Application Insights monitoring data. Which query language should you use?

A. PL/pgSQL  B. Transact-SQL  C. Azure Log Analytics  D. PL/SQL

**A: C — Azure Log Analytics (KQL).**
Both Azure SQL Database Intelligent Insights and Application Insights send diagnostic data to a **Log Analytics workspace**, where it is queried using **KQL (Kusto Query Language)**. T-SQL is for querying SQL databases directly — not monitoring telemetry stored in Log Analytics. PL/pgSQL (PostgreSQL) and PL/SQL (Oracle) are unrelated to Azure monitoring.

---

**Q28.** DRAG DROP — Your company wants to use Azure Application Insights to understand how user behaviors affect an application. Which Application Insights tool should you use to analyze each behavior?

- Feature usage: ?
- Number of people who used the actions and its features: ?
- The effect that the performance of the application has on the usage of a page or a feature: ?

Tools: Impact / User Flows / Users

**A:**
- **Feature usage → User Flows** — visualizes how users navigate through pages and features, showing the paths they take
- **Number of people who used the actions and its features → Users** — counts distinct users who performed specific actions or used features
- **The effect that the performance has on the usage → Impact** — correlates load times and performance with conversion rates and feature usage

---

**Q29.** You have an Azure subscription that contains multiple Azure pipelines. You need to deploy a monitoring solution. The solution must: parse logs from multiple sources, and identify the root cause of issues. What advanced feature of a monitoring tool should you include?

A. directed monitoring  B. synthetic monitoring  C. analytics  D. Alert Management

**A: C — Analytics.** ⚠️ *Dump says B — INCORRECT*
**Analytics** (e.g., Azure Monitor Log Analytics / KQL) ingests and parses logs from multiple sources and enables queries to correlate events and **identify root causes**. This directly satisfies both requirements. **Synthetic monitoring** simulates user transactions proactively to test availability — it does not parse logs or perform root cause analysis. **Alert Management** fires notifications when thresholds are breached — does not parse logs or find root causes. **Directed monitoring** is not a standard monitoring category.

> **Exam tip — Monitoring feature types:**
> | Feature | Purpose |
> |---|---|
> | **Analytics** | Parse multi-source logs, run queries (KQL), identify root causes |
> | **Synthetic monitoring** | Simulate user transactions to test availability proactively |
> | **Alert Management** | Notify on threshold breaches |
> | **Smart Detection** | Automatic anomaly detection (AI-driven) |

---

**Q30.** Your company creates a web application. You need to recommend a solution that automatically sends to Microsoft Teams a daily summary of the exceptions that occur in the application. Which two Azure services should you recommend?

A. Microsoft Visual Studio App Center  B. Azure DevOps Project  C. Azure Logic Apps  D. Azure Pipelines  E. Azure Application Insights

**A: C and E — Azure Logic Apps + Azure Application Insights.**
- **Azure Application Insights (E)** — APM service that automatically captures, tracks, and stores exceptions from web applications. This is the data source.
- **Azure Logic Apps (C)** — workflow automation with a built-in scheduler. Triggered daily → queries Application Insights data → sends summary to Microsoft Teams via connector. This is the automation/delivery piece.

App Center = mobile app crash reporting (not web apps). Azure DevOps Project = project management. Azure Pipelines = CI/CD pipelines, not monitoring or Teams notification workflows.

---

**Q31.** You use Azure Pipelines to manage project builds and deployments. You plan to use Azure Pipelines for Microsoft Teams to notify the legal team when a new build is ready for release. You need to configure the Organization Settings in Azure DevOps to support Azure Pipelines for Microsoft Teams. What should you turn on?

A. Azure Active Directory Conditional Access Policy Validation  B. Alternate authentication credentials  C. Third-party application access via OAuth  D. SSH authentication

**A: C — Third-party application access via OAuth.**
The Azure Pipelines app for Microsoft Teams authenticates to Azure DevOps using **OAuth**. "Third-party application access via OAuth" must be enabled in Azure DevOps Organization Settings to allow the Teams app to obtain an OAuth token and connect. **Conditional Access Policy Validation** enforces AAD CA policies on DevOps access — unrelated to Teams integration. **Alternate authentication credentials** = legacy basic auth (deprecated). **SSH authentication** = git operations only.

> **Exam tip:** Any third-party app integrating with Azure DevOps (Teams, Slack connectors, etc.) requires **"Third-party application access via OAuth"** to be enabled at the Organization level.

---

**Q32.** You have a multi-tier application with an Azure Web Apps front end and an Azure SQL Database back end. You need to recommend a solution to capture and store telemetry data. The solution must: support ad-hoc queries to identify baselines, trigger alerts when metrics exceed baselines, and store application and database metrics in a central location. What should you include in the recommendation?

A. Azure Application Insights  B. Azure SQL Database Intelligent Insights  C. Azure Event Hubs  D. Azure Log Analytics

**A: D — Azure Log Analytics.**
Log Analytics is the **central workspace** that receives telemetry from both Application Insights (web app metrics) and SQL Intelligent Insights (database metrics). It supports **KQL for ad-hoc queries** to identify baselines, and integrates with **Azure Monitor alerts** to trigger on query results when thresholds are exceeded. Application Insights (A) captures app telemetry only — not a central store for DB metrics. SQL Intelligent Insights (B) captures DB performance diagnostics only. Event Hubs (C) is a streaming ingestion pipeline — no built-in query engine or alerting.

> **Exam tip:** "Central location + ad-hoc queries + alerts" for multi-source telemetry (app + database) = **Azure Log Analytics**. It is the hub where all monitoring data converges for KQL querying and alert rule evaluation.

---

## 8. Quick Reference Checklist

### Azure Monitor & Application Insights
- [ ] Azure Monitor = platform; Application Insights = APM service built on Azure Monitor
- [ ] Workspace-based AI (current) stores data in Log Analytics workspace
- [ ] Connection string preferred over instrumentation key (includes endpoint)
- [ ] SDK-based = code changes; auto-instrumentation = agent, no code changes
- [ ] Telemetry types: requests, dependencies, exceptions, traces, pageViews, customEvents, customMetrics, availabilityResults
- [ ] Adaptive sampling = default (auto-adjusts rate); ingestion sampling = at service (doesn't reduce bandwidth)
- [ ] Smart Detection: 24h to learn baseline; detects failure rate + response time anomalies automatically
- [ ] Live Metrics: real-time < 1s latency, NOT stored
- [ ] Profiler: slow request traces (code-level); Snapshot Debugger: exception variable capture

### KQL Essentials
- [ ] `where` → filter; `summarize` → aggregate; `project` → select columns
- [ ] `bin(timestamp, 5m)` → time buckets; `ago(1h)` → relative time
- [ ] `percentile(duration, 95)` → P95 response time
- [ ] `render timechart` → visualize as chart
- [ ] Table names: `requests`, `dependencies`, `exceptions`, `traces`, `pageViews`, `customEvents`

### Availability Tests
- [ ] URL ping test (classic) → simple HTTP GET
- [ ] Standard test → custom headers, POST, SSL check
- [ ] Custom TrackAvailability → complex multi-step (via Azure Function)
- [ ] Multi-step web test = DEPRECATED
- [ ] Test frequency: 5 min default; up to 16 locations

### Alerts
- [ ] 5 alert types: Metric, Log search, Activity log, Smart detection, Availability
- [ ] Severity: 0=Critical, 1=Error, 2=Warning, 3=Informational, 4=Verbose
- [ ] Action Group = reusable; contains: Email/SMS, Webhook, Azure Function, Logic App, ITSM, Runbook
- [ ] Dynamic thresholds = ML-based; fewer false positives than static thresholds
- [ ] Alert processing rules = suppress/override during maintenance windows

### Dashboards
- [ ] Azure Dashboards = static operational overview (pin from any Azure service)
- [ ] Azure Monitor Workbooks = interactive, parameterized analysis
- [ ] Power BI = executive reports, cross-data-source analytics
- [ ] GitHub status badge: `https://github.com/{owner}/{repo}/actions/workflows/{file}/badge.svg`

### Wikis
- [ ] Project Wiki = auto-provisioned per project, stored in hidden Git repo
- [ ] Code Wiki = publish a folder from your code repo as a wiki
- [ ] Both support Markdown + Mermaid diagrams
- [ ] Azure DevOps + Teams: get build/release/PR/work item notifications in Teams channel

### Culture & Process
- [ ] Blameless retrospective = focus on systems, not individuals
- [ ] Just culture = safe to report mistakes; fear = hidden problems
- [ ] SLI = metric; SLO = target; SLA = contract; Error budget = 1 - SLO
- [ ] DORA metrics: Deployment Frequency, Lead Time, MTTR, Change Failure Rate
- [ ] Alert fatigue = too many meaningless alerts → engineers ignore all alerts
- [ ] Flaky test = non-deterministic result; flag but don't fail build; fix root cause

---

*Sources: [MS Learn AZ-400 Path](https://learn.microsoft.com/en-us/training/paths/az-400-implement-continuous-feedback/) | [AZ-400 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-400) | [Application Insights telemetry data model](https://learn.microsoft.com/en-us/azure/azure-monitor/app/data-model-complete) | [Types of Azure Monitor alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-types)*
