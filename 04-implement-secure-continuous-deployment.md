# AZ-400 — Implement a Secure Continuous Deployment

> **Exam Weight:** "Design and implement build and release pipelines" = **50–55%** | "Develop a security and compliance plan" = **10–15%**
> **MS Learn Path:** [AZ-400: Implement a Secure Continuous Deployment using Azure Pipelines](https://learn.microsoft.com/en-us/training/paths/az-400-implement-secure-continuous-deployment/)

---

## Table of Contents
1. [Introduction to Deployment Patterns](#1-introduction-to-deployment-patterns)
2. [Blue-Green Deployment and Feature Toggles](#2-blue-green-deployment-and-feature-toggles)
3. [Canary Releases and Dark Launching](#3-canary-releases-and-dark-launching)
4. [A/B Testing and Progressive Exposure Deployment](#4-ab-testing-and-progressive-exposure-deployment)
5. [Integrate with Identity Management Systems](#5-integrate-with-identity-management-systems)
6. [Manage Application Configuration Data](#6-manage-application-configuration-data)
7. [Additional Exam Topics (Beyond MS Learn)](#7-additional-exam-topics-beyond-ms-learn)
8. [Past Exam Scenario Bank](#8-past-exam-scenario-bank)

---

## 1. Introduction to Deployment Patterns

### Classical vs Modern Deployment Patterns

**Classical (Big Bang) Deployment:**
- Deploy everything at once to all users simultaneously.
- High risk — if something breaks, everyone is affected.
- Requires downtime window.
- Difficult to roll back.
- Still used in legacy systems with scheduled maintenance windows.

**Modern Deployment Patterns:**
- Gradual, progressive rollout to limit blast radius.
- Zero or near-zero downtime.
- Data-driven decisions (metrics, telemetry drive rollout speed).
- Fast, automated rollback.
- Decouple **deployment** (code goes to production servers) from **release** (feature is visible to users).

### Microservices Architecture Principles
- Applications decomposed into **small, independent services** — each owns its data and logic.
- Each service can be **deployed, scaled, and updated independently**.
- Services communicate via **APIs (REST, gRPC, events)**.
- Enables **independent deployment** — updating one service doesn't require redeploying others.

**DevOps implications:**
- Multiple pipelines — one per service.
- Need **service versioning** and **backward-compatible APIs**.
- Deployment complexity increases — need orchestration (Kubernetes).
- Requires **observability** (distributed tracing, centralized logging).

> **Exam tip:** Microservices enable independent deployments but increase operational complexity. They are the architectural foundation for modern progressive delivery patterns.

### VMSS Health Monitoring for Upgrade Operations

When deploying to an **Azure Virtual Machine Scale Set (VMSS)** with rolling upgrades, each instance must report its health status so Azure knows whether it is **eligible for an upgrade**.

**Two mechanisms exist — key differences:**

| Mechanism | How it works | Controls upgrade eligibility? | Supports mutual TLS (client cert)? |
|---|---|---|---|
| **Application Health Extension** | Installed on each VMSS instance; probes the app endpoint **locally from within the instance** | ✅ Yes — VMSS uses this signal to determine per-instance upgrade eligibility | ✅ Yes — probes from inside the VM, bypasses mutual auth requirement |
| **Azure Load Balancer health probe** | LB probes the instance from outside the network | ❌ No — only controls traffic routing (whether LB sends traffic to instance), not upgrade eligibility | ❌ No — LB probes cannot present client certificates for mutual TLS |
| **Custom Script Extension** | Runs arbitrary scripts on VMs | ❌ No — no native health signal to VMSS | Possible but very high admin effort |
| **Azure Monitor autoscale** | Scales instance count based on metrics | ❌ No — scaling, not health/upgrade control | N/A |

> **Exam tip (tested):** "VMSS + identify instances eligible for upgrade + minimize admin effort" → **Application Health Extension**. The LB health probe is a distractor — it manages traffic routing, NOT upgrade eligibility. When the app requires mutual TLS (client certificate), the LB probe cannot satisfy this; the Application Health Extension probes locally from inside the instance, so mutual TLS is not an issue.

### Deployment Pattern Comparison
| Pattern | Downtime | Rollback | Risk | Infrastructure cost |
|---|---|---|---|---|
| **Big Bang (Recreate)** | Yes | Hard | High | Low |
| **Rolling** | No | Moderate | Medium | Same |
| **Blue-Green** | No | Instant (swap back) | Low | Double (briefly) |
| **Canary** | No | Fast (reroute) | Low | Slight increase |
| **Ring** | No | Per ring | Low | Moderate |
| **Feature flag** | No | Instant (toggle off) | Very low | Same |

---

## 2. Blue-Green Deployment and Feature Toggles

### Blue-Green Deployment
- Maintain **two identical production environments**: Blue (current live) and Green (new version).
- Deploy new version to **Green** (idle environment).
- Run smoke tests and validation against Green.
- **Switch traffic** from Blue to Green (instant cutover).
- Blue becomes the standby — instant rollback by switching traffic back.

```
Users → [Load Balancer / Traffic Manager]
           ├── Blue  (v1 — current live)
           └── Green (v2 — new version, being tested)

After cutover:
Users → Green (v2 live)
Blue (v1 kept as rollback target)
```

**Benefits:**
- Zero downtime.
- Instant rollback (switch traffic back).
- Green can be tested with production-like load before cutover.

**Drawbacks:**
- Requires double infrastructure (briefly).
- Database schema changes must be backward-compatible (both versions share the DB during cutover).

### Azure App Service Deployment Slots (Exam Favorite)
- **Deployment slots** are the Azure-native implementation of blue-green for App Service.
- Each slot is a separate instance of the app with its own URL.
- Default: **production slot** + create additional slots (staging, QA, etc.).

**Slot swap process:**
1. Deploy new version to the **staging slot**.
2. Warm up the staging slot (app starts, health checks pass).
3. Run smoke tests against the staging slot URL.
4. **Swap** staging and production slots — staging becomes live, old production becomes staging.
5. If issues arise: **swap back** immediately.

```yaml
# Azure Pipelines task to swap slots
- task: AzureAppServiceManage@0
  inputs:
    azureSubscription: 'MyServiceConnection'
    action: 'Swap Slots'
    webAppName: 'myapp'
    resourceGroupName: 'myRG'
    sourceSlot: 'staging'
    swapWithProduction: true
```

**Slot-specific settings:**
- Settings marked **"Slot setting"** stick to the slot (don't swap) — e.g., connection strings pointing to different databases per environment.
- Settings NOT marked as slot-specific travel with the swap.

> **Exam tip:** App Service slots = blue-green for PaaS. Know the swap process, slot-specific settings, and the `AzureAppServiceManage@0` task with `action: 'Swap Slots'`.

### Feature Toggles (Feature Flags)
- **Feature toggle** = a code-level on/off switch for a feature.
- Decouple **deployment** (code is in production) from **release** (feature is visible to users).
- Enables **trunk-based development** — incomplete features are deployed but hidden.

### Feature Toggle Types (Exam Favorite)
| Type | Lifecycle | Use case |
|---|---|---|
| **Release toggle** | Short-lived (days–weeks) | Hide incomplete features until ready for release |
| **Experiment toggle** | Short-lived | A/B testing — show different versions to different users |
| **Ops toggle** | Medium-lived | Kill switch — disable a feature in production if it causes issues |
| **Permission toggle** | Long-lived | Enable features for specific users/roles/tiers (premium users, beta testers) |
| **Infrastructure toggle** | Long-lived | Switch between infrastructure implementations (e.g., old vs new DB) |

### Feature Toggle Maintenance
- Feature toggles are **technical debt** — they must be cleaned up after their purpose is served.
- **Toggle debt:** Too many old, unused toggles make code complex and hard to test.
- Best practices:
  - Add a **removal date** or ticket when creating a toggle.
  - Track toggles in a backlog.
  - Remove toggles soon after the rollout is complete.
  - Never let toggles become permanent unless they are permission toggles by design.

> **Exam tip:** The biggest risk of feature toggles is **accumulation of toggle debt** — old toggles never removed. Know the toggle types and their intended lifespans.

### Azure App Configuration Feature Management
- **Azure App Configuration** has native **Feature Manager** support.
- Feature flags stored as key-value pairs with the prefix `.appconfig.featureflag/`.
- SDK: `Microsoft.FeatureManagement` (.NET), `@azure/app-configuration` (JavaScript).
- Filters: **TimeWindowFilter** (enable during a time window), **TargetingFilter** (enable for % of users or specific groups), **CustomFilter** (custom logic).

```csharp
// .NET Feature Management
if (await featureManager.IsEnabledAsync("NewCheckout"))
{
    // show new checkout UI
}
```

---

## 3. Canary Releases and Dark Launching

### Canary Releases
- Named after the "canary in a coal mine" — an early warning system.
- Deploy new version to a **small subset** of users/instances (e.g., 5–10%).
- **Monitor** errors, latency, business metrics.
- If healthy: **gradually increase** the percentage (5% → 25% → 50% → 100%).
- If issues detected: **reroute** all traffic back to the old version.

**Implementation approaches in Azure:**
- **Azure Traffic Manager** — weighted routing between two App Service instances.
- **Azure Front Door** — weighted routing with health probes.
- **App Service deployment slot** — traffic routing setting (% to staging slot).
- **Kubernetes** — multiple deployments with different replica counts + load balancer.
- **Feature flags** — deploy to all but enable feature for X% of users.

### Azure Traffic Manager for Canary
- **Azure Traffic Manager** is a DNS-based global load balancer.
- **Traffic routing methods:**

| Method | Description | Use case |
|---|---|---|
| **Priority** | Route to primary; failover to secondary | Active-passive failover |
| **Weighted** | Distribute traffic by weight | Canary releases (e.g., 10% new, 90% old) |
| **Performance** | Route to lowest-latency endpoint | Geo-distributed apps |
| **Geographic** | Route by user location | Data residency, regional compliance |
| **Multivalue** | Return multiple endpoints | DNS-based client-side selection |
| **Subnet** | Route by client IP range | Test specific IP ranges |

> **Exam tip:** For canary with Traffic Manager → **Weighted** routing. For failover blue-green → **Priority** routing.

### App Service Traffic Routing for Canary
```
App Service → Production Slot: 90%
           → Staging Slot: 10%   ← canary (new version)
```
- Set via Azure Portal: App Service → Deployment slots → Traffic % slider.
- Users sticky to a slot via a **cookie** (`x-ms-routing-name`).
- Allows testing the new version with real users without full rollout.

### Dark Launching
- Deploy a new version to production but **keep it invisible to users**.
- The new code is executed in the background (shadow mode) to test performance and correctness.
- **No user impact** — dark launched features don't change the UI/experience.
- Also called **shadow deployment** or **dark traffic**.

**Use cases:**
- Test new backend logic with real production traffic without affecting responses.
- Validate performance characteristics under real load.
- Test database queries/APIs before exposing to users.

**vs Canary:** Canary shows the new version to real users (small %). Dark launching runs the new code but doesn't expose it.

> **Exam tip:** Dark launching = code runs in production but users don't see it. Canary = users see the new version (small %). Feature flag = users see the new version only if flag is on for them.

---

## 4. A/B Testing and Progressive Exposure Deployment

### A/B Testing
- Show **version A** to one group of users and **version B** to another group simultaneously.
- Measure which version achieves better **business outcomes** (conversion rate, click-through, engagement).
- Data-driven decision to promote the winning version.
- **Not about finding bugs** — about validating business hypotheses.

**A/B vs Canary:**
| Aspect | A/B Testing | Canary |
|---|---|---|
| Purpose | Validate business hypothesis | Validate technical stability |
| Metrics | Business KPIs (conversion, engagement) | Error rate, latency, crash rate |
| Duration | Weeks (statistical significance needed) | Hours/days |
| Rollback trigger | Business metric underperforms | Technical failure detected |

**Azure implementation:** App Configuration **TargetingFilter** + Application Insights for metric tracking.

### Progressive Exposure Deployment
- Umbrella term for any strategy that gradually increases the exposure of a new release.
- Includes: canary, ring deployments, percentage-based traffic routing.
- **Goal:** Minimize blast radius while gaining real-world signal.

### Deployment Rings (Exam Favorite)
- Release in concentric "rings" — each ring is a larger audience.
- Inner rings = early adopters/internal users. Outer rings = general population.

**Typical ring structure:**
| Ring | Audience | Size | Purpose |
|---|---|---|---|
| **Ring 0 (Canary)** | Internal team / developers | ~0.1% | First validation |
| **Ring 1 (Early Adopters)** | Opted-in beta users | ~1–5% | Early feedback |
| **Ring 2 (Fast Ring)** | Enthusiast users | ~10–20% | Broader validation |
| **Ring 3 (Slow Ring)** | General users | ~80–100% | Full rollout |

- Each ring has a **gate** — either automated health checks or manual approval.
- Deployment to the next ring only proceeds if current ring is healthy.
- Microsoft uses rings for Windows/Azure updates.

### CI/CD with Deployment Rings
- Each ring maps to an **environment** in Azure DevOps.
- Pipeline has stages per ring with approval gates between them.
- Metrics from each ring (error rate, telemetry) inform progression decisions.

```
[Build] → [Ring 0: Internal] → [Gate] → [Ring 1: Beta] → [Gate] → [Ring 2: General] → [Gate] → [Ring 3: All]
```

---

## 5. Integrate with Identity Management Systems

### GitHub SSO (Single Sign-On)
- Enterprise GitHub organizations can enforce **SAML SSO** via an identity provider (IdP).
- Users must authenticate via the org's IdP (e.g., Microsoft Entra ID / Azure AD) before accessing org resources.
- **Benefits:** Centralized user management, enforce MFA, automatic deprovisioning.
- **SCIM (System for Cross-domain Identity Management):** Automatically provision/deprovision GitHub users when employees join/leave in the IdP.

### GitHub Permissions and Roles (Exam Favorite)
**Repository roles:**
| Role | Access level |
|---|---|
| **Read** | View and clone the repo |
| **Triage** | Read + manage issues/PRs without writing code |
| **Write** | Triage + push branches, manage releases |
| **Maintain** | Write + manage repo settings (no destructive actions) |
| **Admin** | Full control including deletion and security |

**Organization roles:**
- **Owner:** Full org administration.
- **Member:** Default role; access determined by team/repo settings.
- **Outside Collaborator:** Access to specific repos only; not an org member.

**Teams:** Groups of org members. Grant teams access to repos. Easier than per-user management.

> **Exam tip:** Outside collaborators have access to specific repos but are **not** org members. They cannot see org-wide resources. Use for contractors or third parties.

### Azure DevOps Permissions and Security Groups (Exam Favorite)
**Built-in security groups (project level):**
| Group | Permissions |
|---|---|
| **Readers** | View project, work items, builds |
| **Contributors** | Readers + create/edit work items, push code, run pipelines |
| **Project Administrators** | Contributors + manage project settings, permissions |
| **Build Administrators** | Manage build pipelines and agents |
| **Release Administrators** | Manage release pipelines |

**Permission inheritance:**
- Permissions flow from Organization → Project → Repository/Pipeline.
- **Explicit deny overrides inherited allow.**
- Use security groups rather than per-user permissions for manageability.

**Azure DevOps access levels:**
| Level | Cost | Capabilities |
|---|---|---|
| **Stakeholder** | Free | Work items only (no repos, limited pipelines) |
| **Basic** | Paid | Full access except Test Plans |
| **Basic + Test Plans** | Paid | Full access including Azure Test Plans |

> **Exam tip:** Stakeholder access is **free** and allows viewing/creating work items but cannot access source code, pipelines, or full board features. Good for non-technical stakeholders.

### Workload Identities — Service Principals vs Managed Identities (Exam Favorite)

**Service Principal:**
- An identity for **applications, pipelines, and automated tools** in Microsoft Entra ID.
- Requires a **client secret** (password) or **certificate** for authentication.
- Must be manually created and managed.
- Secret rotation required.
- Use when: running **outside Azure** (GitHub Actions, on-premises pipelines, Jenkins) or when you need cross-tenant access.

**Three values required to authenticate a task/app using a service principal:**

| Value | Also called | What it identifies |
|---|---|---|
| **Tenant ID** | Directory ID | The Azure AD / Entra ID tenant where the SP lives |
| **App ID** | Client ID / Application ID | The specific service principal (application registration) |
| **Client Secret** | Password / App secret | The credential that proves identity |

> **Exam tip (tested):** Task authenticating with a service principal needs **Tenant ID + App ID + Client Secret**. Object ID (internal Azure AD identifier, not used for auth) and Subscription ID (Azure resource scope, not an identity credential) are NOT required for authentication itself.

**Managed Identity:**
- A special service principal **automatically managed by Azure**.
- **No credentials to manage** — Azure handles rotation automatically.
- Only works for resources **running inside Azure**.
- Two types:

| Type | Lifecycle | Sharing | Use case |
|---|---|---|---|
| **System-assigned** | Tied to the resource; deleted when resource is deleted | Cannot be shared | Single resource needs identity (e.g., one VM) |
| **User-assigned** | Independent lifecycle; must be deleted explicitly | Can be assigned to multiple resources | Multiple resources share an identity (e.g., multiple VMs in a scale set) |

> **Exam tip (critical):** Managed identity = **Azure resource only, no credential management**. Service principal = **any platform, requires credential**. If the question asks about a self-hosted agent on Azure VM authenticating to Key Vault → **Managed Identity**. If it's a GitHub Actions runner → **Service Principal** (or Workload Identity Federation).

> **Exam tip — minimize permission grants across multiple resources:**
> - **1 resource** needs Key Vault access → **system-assigned** managed identity (1 identity, 1 grant)
> - **Multiple resources** need the **same** Key Vault access → **user-assigned** managed identity (1 shared identity = 1 grant covers all resources; system-assigned would require one grant per resource)

### Workload Identity Federation (WIF)
- Modern approach: eliminates secrets entirely for OIDC-compatible platforms.
- **GitHub Actions** → use OIDC token to authenticate to Azure (no stored secret).
- **Azure Pipelines** → Workload Identity Federation service connection (no client secret).
- Token is short-lived, signed by the platform's OIDC provider.
- Microsoft's **recommended** approach over service principal with client secrets.

### Microsoft Entra ID Integration with Azure DevOps
- Connect Azure DevOps organization to **Microsoft Entra ID** (formerly Azure AD).
- Enables: conditional access policies, MFA enforcement, SSO, SCIM provisioning.
- Users from Entra ID can be added to Azure DevOps.
- Entra groups can be mapped to Azure DevOps security groups.
- Requirement: organization owner must be an Entra ID member.

#### Azure AD Feature Comparison (Exam Tested)

| Feature | Purpose | Exam Clue |
|---|---|---|
| **Conditional Access** | Enforce policies (MFA, block) based on conditions: location, device, user, app | "MFA from untrusted networks" / "require MFA based on location" |
| **Access reviews** | Periodic audits of who has access | "review and recertify access" |
| **Managed identities** | Service identities for Azure resources (no passwords) | "app accessing Azure service without credentials" |
| **Entitlement management** | Access package lifecycle (request, approve, expire) | "self-service access requests" |

> **Exam tip:** "Require MFA when accessing from untrusted networks" = **Conditional Access**. It is the only Azure AD feature that evaluates *conditions* (network location, device compliance, sign-in risk) and *enforces* authentication requirements accordingly.

---

## 6. Manage Application Configuration Data

### Rethinking Application Configuration
- **Anti-pattern:** Hardcode configuration (connection strings, secrets) in source code or app binaries.
- **Better:** Externalize configuration — separate config from code.
- Benefits: deploy the same binary to dev/QA/prod; change config without redeploying.
- 12-Factor App principle: **Store config in the environment**.

### Separation of Concerns
- **Code** (logic) should be separate from **configuration** (environment-specific settings) and **secrets** (sensitive credentials).
- Three tiers:

| Tier | What it contains | Where to store |
|---|---|---|
| **Code** | Application logic | Source control |
| **Configuration** | Non-sensitive settings (feature flags, timeouts, URLs) | Azure App Configuration |
| **Secrets** | Sensitive values (passwords, API keys, certificates) | Azure Key Vault |

### External Configuration Store Pattern
- Configuration is stored externally (not in the app binary or config files).
- App reads config from the external store at startup or dynamically.
- Enables: environment-specific config, runtime changes, centralized management.
- Azure solution: **Azure App Configuration** for settings + **Azure Key Vault** for secrets.

### Azure Secure Files
- **Secure files** in Azure Pipelines Library store sensitive files (certificates, provisioning profiles, SSH keys).
- Uploaded to Azure DevOps and encrypted at rest.
- Used in pipelines via the `DownloadSecureFile@1` task.
- Files are downloaded to the agent at runtime; not accessible after the pipeline run.
- Access controlled per pipeline via pipeline permissions.

```yaml
- task: DownloadSecureFile@1
  name: mySecureFile
  inputs:
    secureFile: 'my-cert.p12'
- script: |
    echo "File path: $(mySecureFile.secureFilePath)"
```

> **Exam tip:** Secure files = for **files** (certificates, keys). Variable groups = for **values** (strings, secrets). Both are stored in Library.

### Azure App Configuration (Exam Favorite)
- A **managed service** for centralizing application configuration.
- Stores **key-value pairs** with support for labels, content types, and etags.
- Complements Azure Key Vault — stores non-secret config; Key Vault stores secrets.

**Key features:**
- **Key-value pairs:** Configuration stored as `key = value` pairs.
- **Labels:** Tag configurations for environments (e.g., label = `Production`, `Development`).
- **Snapshots:** Point-in-time capture of configuration for safe deployments.
- **Feature flags:** Built-in feature management with filters.
- **Key Vault references:** App Configuration stores a **reference** to a Key Vault secret (the actual secret stays in Key Vault).
- **Dynamic configuration:** Apps can refresh config without restarting (sentinel key pattern).
- **Geo-replication:** Replicate to multiple regions for high availability.

**Key naming convention:**
```
AppName:Environment:SettingName
e.g., MyApp:Production:DatabaseTimeout
```

**Labels for environment separation:**
```
Key: ConnectionString    Label: Development  →  "Server=dev-sql..."
Key: ConnectionString    Label: Production   →  "Server=prod-sql..."
```

### App Configuration Feature Management
- Feature flags stored in App Configuration with `.appconfig.featureflag/` prefix.
- **Feature filters** control when a flag is enabled:
  - **TimeWindowFilter:** Enable during a specific time window.
  - **TargetingFilter:** Enable for a % of users or specific user groups.
  - **Custom filters:** Implement `IFeatureFilter` interface.
- Integrate into pipelines: read feature flag state and conditionally run steps.

### Azure Key Vault
- **Azure Key Vault** securely stores and manages:
  - **Secrets:** Passwords, connection strings, API keys.
  - **Keys:** Cryptographic keys (RSA, EC) for encryption/signing.
  - **Certificates:** SSL/TLS certificates with auto-renewal.

**Access control:**
- **RBAC (recommended):** Assign roles like `Key Vault Secrets User` to a managed identity or service principal. Modern, flexible.
- **Access Policies (legacy):** Per-vault policies for get/list/set operations. Still works but RBAC preferred.

**Key Vault Roles (RBAC):**
| Role | Access |
|---|---|
| **Key Vault Administrator** | Full management of vault and all objects |
| **Key Vault Secrets Officer** | Create, read, update, delete secrets |
| **Key Vault Secrets User** | Read secrets only (for apps consuming secrets) |
| **Key Vault Reader** | Read vault metadata; cannot read secret values |

> **Exam tip:** RBAC is the **recommended** approach for Key Vault access control. Access Policies are legacy. For an app/pipeline reading secrets → assign `Key Vault Secrets User` role.

### Integrating Key Vault with Azure Pipelines

**Method 1: Variable Group linked to Key Vault**
1. Create a Variable Group in Library.
2. Toggle "Link secrets from an Azure Key Vault."
3. Select Key Vault and map secrets to variable names.
4. Pipeline references variables normally; values fetched from Key Vault at runtime.

**Method 2: AzureKeyVault task**
```yaml
- task: AzureKeyVault@2
  inputs:
    azureSubscription: 'MyServiceConnection'
    KeyVaultName: 'mykeyvault'
    SecretsFilter: 'DatabasePassword,ApiKey'
    RunAsPreJob: true
```
Secrets become pipeline variables after this task runs.

### Secrets, Tokens, and Certificates Best Practices
| Type | Storage | Access method |
|---|---|---|
| **Passwords / API keys** | Azure Key Vault Secrets | Managed Identity or Variable Group |
| **Encryption keys** | Azure Key Vault Keys | Managed Identity |
| **SSL/TLS Certificates** | Azure Key Vault Certificates or Secure Files | Key Vault task or DownloadSecureFile |
| **PATs / OAuth tokens** | Azure DevOps Library (variable, secret) | Pipeline variable |
| **SSH keys** | Azure DevOps Secure Files | DownloadSecureFile task |

**Never:**
- Store secrets in source control (even encrypted).
- Echo secret variables in pipeline logs.
- Pass secrets as plain-text command-line arguments.

### DevOps Inner and Outer Loop
| Loop | Who | Activities | Feedback speed |
|---|---|---|---|
| **Inner loop** | Developer | Code → Build → Unit test → Commit | Minutes |
| **Outer loop** | Pipeline/System | CI → Integration test → Deploy → Monitor | Hours/days |

- **Inner loop** optimizations: fast local builds, code completion, lint on save.
- **Outer loop** optimizations: fast CI pipelines, caching, parallel stages.
- Azure App Configuration and feature flags live at both loops — developers test flags locally (inner) and manage them in production (outer).

### Dynamic Configuration and Feature Flags
- **Dynamic config refresh:** App watches a **sentinel key** in App Configuration. When it changes, the app reloads all config without restarting.
- Sentinel key pattern: update the sentinel after all config changes are deployed → app picks up the change.
- Enables: **configuration updates without redeployment**.
- Use with feature flags to enable/disable features in real-time.

---

## 7. Additional Exam Topics (Beyond MS Learn)

### Deployment Pattern Decision Guide (Exam Favorite)
Use this to pick the right strategy from a scenario description:

| Scenario keyword | Strategy |
|---|---|
| "Zero downtime, instant rollback" | **Blue-Green** (slot swap) |
| "Small % of users first, then increase" | **Canary** |
| "Validate business metrics / conversion" | **A/B testing** |
| "Internal users first, then external" | **Ring deployment** |
| "Deploy code but hide from users" | **Dark launching / Feature flag** |
| "Turn off a feature without redeployment" | **Ops feature toggle** |
| "Enable for premium users only" | **Permission toggle** |
| "Gradually shift traffic with DNS control" | **Azure Traffic Manager (weighted)** |
| "Failover only, not gradual" | **Azure Traffic Manager (priority)** |

### GitHub Actions Authentication to Azure
Two options for GitHub Actions to authenticate to Azure:

**Option 1: Service Principal + Client Secret (Legacy)**
```yaml
- uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}
```
Requires storing JSON credentials as a GitHub secret. Secret must be rotated.

**Option 2: Workload Identity Federation / OIDC (Recommended)**
```yaml
- uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```
No secret stored — GitHub issues an OIDC token; Azure validates it. No rotation needed.

### Key Vault Reference in App Service
- Instead of storing a secret value in App Settings, store a **reference** to Key Vault.
- Format: `@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/MySecret/)`
- App Service resolves the reference at runtime using its managed identity.
- The actual secret never leaves Key Vault.

### Azure DevOps PATs (Personal Access Tokens)
- PATs authenticate users/scripts to Azure DevOps REST API or Git over HTTPS.
- Should have **minimal scope** (don't use full-access tokens).
- Should have **expiry dates** — not indefinite.
- Can be created per-user; org admins can revoke all PATs for a user.
- Store PATs in Key Vault or Secure Files — never hardcode.

### RBAC vs Access Policies in Key Vault
| Aspect | RBAC | Access Policies |
|---|---|---|
| Recommended | **Yes (Microsoft recommended)** | No (legacy) |
| Scope | Subscription/RG/Vault/individual secret | Vault only |
| Audit | Full Azure RBAC audit trail | Limited |
| Management | Via Entra ID role assignments | Via Key Vault portal |
| Flexibility | Fine-grained (individual objects) | Vault-level only |

### GitHub Advanced Security (GHAS) for Secure CD
Relevant to "secure" continuous deployment:
- **Secret scanning:** Detects hardcoded secrets in code before they reach production.
- **Code scanning (CodeQL):** SAST — finds vulnerabilities before deployment.
- **Dependency review / Dependabot:** Prevents deploying code with known-vulnerable dependencies.
- GHAS can **block PRs** from merging if vulnerabilities are detected — securing the deployment pipeline at the source.

---

## 8. Past Exam Scenario Bank

---

**Q1 — Deployment strategy: zero downtime + instant rollback**
> You need to deploy a new version of an Azure App Service web app with zero downtime. If the deployment fails, you must be able to roll back within 1 minute. What is the recommended approach?

**A: Use Azure App Service deployment slots (blue-green).**
Deploy to the staging slot → validate → swap to production. Rollback = swap back. The `AzureAppServiceManage@0` task with `action: 'Swap Slots'` handles this in a pipeline.

---

**Q2 — Deployment slots: slot-specific setting**
> You have a connection string that must point to the dev database in the staging slot and the production database in the production slot. It must NOT be swapped. How do you configure this?

**A: Mark the connection string as a "Slot setting" in App Service.**
Slot settings are pinned to a slot and are not swapped. The connection string stays with its slot regardless of swaps.

---

**Q3 — Feature toggle type: disable if issues arise**
> You deploy a new payments feature. If issues are detected in production, you want to disable it immediately without redeploying. Which feature toggle type should you use?

**A: Ops toggle (operational toggle / kill switch).**
Designed to disable a feature in production instantly without code changes or redeployment.

---

**Q4 — Canary with Traffic Manager: routing method**
> You want to gradually shift traffic from an old version to a new version of your app, starting with 10% to the new version. You are using Azure Traffic Manager. Which routing method should you configure?

**A: Weighted routing.**
Set the new version's endpoint weight to 10 and the old version's to 90. Adjust weights as confidence grows.

---

**Q5 — Dark launching vs canary**
> You want to run a new search algorithm in production alongside the existing one, but only use the new algorithm's results internally for logging/analysis — users always see the old algorithm's results. What is this pattern?

**A: Dark launching (shadow deployment).**
The new code runs but its output is not shown to users. Used to validate performance/correctness under real traffic without user impact.

---

**Q6 — A/B testing vs canary**
> You want to determine whether a new checkout flow increases conversion rate compared to the existing one. 50% of users see version A, 50% see version B. After 2 weeks you analyze conversion data. What is this pattern?

**A: A/B testing.**
A/B testing validates business hypotheses using user segmentation and business metrics. (Canary is for technical validation, not business KPI comparison.)

---

**Q7 — Ring deployment: who gets the first ring?**
> Your organization uses ring-based deployment. Which group should be in Ring 0?

**A: Internal developers/the engineering team.**
Ring 0 is the smallest, most trusted internal group. Issues here have no customer impact and can be quickly fixed.

---

**Q8 — Managed identity vs service principal: on-premises pipeline**
> A self-hosted Azure Pipelines agent runs on an on-premises machine (not an Azure VM). It needs to access Azure Key Vault. Which identity type should you use?

**A: Service principal.**
Managed identities are only available for resources running inside Azure. An on-premises machine cannot have a managed identity. Use a service principal with Workload Identity Federation or client secret.

---

**Q9 — Managed identity: system-assigned vs user-assigned**
> You have 5 Azure VMs in a scale set, and all of them need the same access to Azure Key Vault. Which managed identity type should you use?

**A: User-assigned managed identity.**
One user-assigned identity can be assigned to all 5 VMs. System-assigned creates a separate identity per VM and cannot be shared.

---

**Q21 — Managed identity type: minimize permission grants across multiple servers**
> You have a Key Vault (KV1) and 3 web servers. App1 is deployed to all web servers and needs to retrieve a secret from KV1. Requirements: minimize permission grants, principle of least privilege. What should you include?
> A. RBAC permissions  B. System-assigned managed identity  C. User-assigned managed identity  D. Service principal

**A: C — User-assigned managed identity.** *(Note: some exam dumps record B — this is incorrect)*
With 3 web servers, system-assigned would require 3 separate KV1 permission grants (one per server identity). User-assigned creates **one shared identity** assigned to all 3 servers — requiring only **1 KV1 permission grant**. This directly minimizes the number of grants while maintaining least privilege (no credentials to manage).

---

**Q10 — Key Vault access: RBAC vs access policy**
> You are configuring an application's managed identity to read secrets from Azure Key Vault. What is the Microsoft-recommended access control method?

**A: RBAC — assign the `Key Vault Secrets User` role to the managed identity.**
RBAC is the recommended approach over access policies. It provides fine-grained control and a full audit trail.

---

**Q11 — Azure App Configuration: environment-specific config**
> You want to store the same configuration key with different values for Development and Production environments in Azure App Configuration. How should you differentiate them?

**A: Use labels.**
Create the key twice — once with label `Development` and once with label `Production`. The app requests config with the specific label for its environment.

---

**Q12 — Key Vault Reference in App Service**
> You want your Azure App Service to read a secret from Key Vault without storing the secret value in App Settings. What should you configure?

**A:** Store a **Key Vault reference** in App Settings using the format:
`@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/MySecret/)`.
The App Service resolves the reference at runtime using its system-assigned managed identity. The secret never leaves Key Vault.

---

**Q13 — Azure DevOps permissions: developer cannot push**
> A developer needs to push code to a repository in Azure DevOps but currently has Read access only. What group should you add them to?

**A: Add them to the Contributors security group.**
Contributors can push code, create branches, create PRs, and manage work items.

---

**Q14 — GitHub outside collaborator**
> You need to give a contractor access to one specific private repository in your GitHub organization without making them an org member. How should you grant access?

**A: Add them as an Outside Collaborator** on that specific repository.
Outside collaborators access specific repos only and are not org members.

---

**Q15 — Secure files vs variable groups**
> You need to store an SSL certificate file and a database password in Azure Pipelines Library. Where should each go?

**A:**
- SSL certificate (file) → **Secure Files** in Library (use `DownloadSecureFile@1`).
- Database password (string value) → **Variable Group** with secret variable (or linked to Key Vault).

---

**Q18 — VMSS health check for upgrade eligibility**
> You deploy App1 (HTTPS only, requires mutual TLS client certificate) to VMSS1 behind a Standard Load Balancer. You need a health check that identifies whether individual VMSS instances are eligible for an upgrade operation, with minimum admin effort. What should you use?
>
> A. Custom Script Extension  B. Application Health Extension  C. Azure Monitor autoscale  D. Azure Load Balancer health probe

**A: B — Application Health Extension.**
The Application Health Extension is installed on each VMSS instance and probes the application endpoint locally (from within the VM). Its health signal is used by VMSS to determine per-instance upgrade eligibility. Because it probes from inside the instance, mutual TLS (client certificate requirement) is not a blocker. The LB health probe (D) only controls traffic routing — not upgrade eligibility — and cannot present client certificates. Custom Script Extension (A) requires custom code (high admin effort). Azure Monitor autoscale (C) is for scaling, not health/upgrade control.

---

**Q17 — Service principal authentication: which three values are required?**
> You are configuring a build pipeline task (Task1) that will authenticate using an Azure AD service principal. Which three values must you configure?
> A. Object ID  B. Tenant ID  C. App ID  D. Client Secret  E. Subscription ID

**A: B, C, D — Tenant ID, App ID, Client Secret.**
- **Tenant ID** — identifies the Azure AD directory where the service principal is registered.
- **App ID (Client ID)** — identifies the specific service principal / application registration.
- **Client Secret** — the credential (password) that proves the identity of the service principal.

Object ID (A) is an internal Azure AD identifier not used in authentication flows. Subscription ID (E) is an Azure resource scope identifier — it tells Azure *where* to operate, not *who* is authenticating.

---

**Q22 — Service principal sign-in: on-premises app upgrade**
> You have an on-premises app (App1) that accesses Azure resources using credentials stored in a configuration file. You plan to upgrade App1 to use an Azure service principal. What is required for App1 to programmatically sign in to Azure AD?
> A. Application ID, client secret, and object ID
> B. Client secret, object ID, and tenant ID
> C. Application ID, client secret, and tenant ID
> D. Application ID, client secret, and subscription ID

**A: C — Application ID, client secret, and tenant ID.**
The OAuth 2.0 client credentials flow requires:
- **Application (Client) ID** — identifies the app/service principal
- **Client Secret** — the credential proving the service principal's identity
- **Tenant ID** — the Azure AD directory endpoint: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`

**Object ID** is the internal directory object identifier — not used in authentication requests. **Subscription ID** identifies an Azure resource scope — it is not an identity credential and not needed to sign in to Azure AD.

---

**Q19 — Key Vault: secret type and pipeline access**
> You have an Azure subscription with a Key Vault named Vault1, a pipeline named Pipeline1, and an Azure SQL database named DB1. Pipeline1 deploys an app that authenticates to DB1 using a password. You need to store the password in Vault1 so Pipeline1 can access it.
>
> - Store the password as a: [Certificate / Key / Secret]
> - Grant Pipeline1 access to Vault1 by modifying the: [Access control (IAM) settings / Access policies / Security settings]

**A:**
- **Store as: Secret** — Passwords and connection strings are Secrets. Keys = cryptographic keys. Certificates = X.509 certs.
- **Grant access via: Access policies** — Key Vault Access Policies grant a service principal (Pipeline1's service connection) `Get`/`List` permissions on secrets. "Access control (IAM)" is Azure RBAC — a separate model; both work but Access Policies is what the exam tests here.

> **Note:** Microsoft now recommends RBAC over Access Policies for new deployments, but exam questions based on older content may reference Access Policies as the expected answer.

---

**Q20 — Minimize credential leakage in pipeline**
> Your company identifies security as the highest priority for a new web application in Azure DevOps. You need to recommend a solution to minimize the likelihood that infrastructure credentials will be leaked. What should you recommend?
>
> A. Add a Run Inline Azure PowerShell task  B. Add a PowerShell task and run Set-AzureKeyVaultSecret  C. Add an Azure Key Vault task  D. Add Azure Key Vault references to ARM templates

**A: C — Add an Azure Key Vault task to the pipeline.**
The Azure Key Vault task retrieves secrets from Key Vault at runtime and injects them as **masked pipeline variables** — credentials never appear in YAML, logs, or source control. This is the purpose-built Microsoft solution for preventing credential leakage in pipelines.

- B (`Set-AzureKeyVaultSecret`) **writes** a secret INTO Key Vault — wrong direction; doesn't protect pipeline execution.
- A (Inline PowerShell) increases leak risk — credentials could be hardcoded inline.
- D (Key Vault references in ARM templates) solves app runtime config, not pipeline credential protection.

> **Note:** Some exam dumps record B as the answer, but C is the technically correct answer per Microsoft documentation.

---

**Q23 — Azure AD: MFA for untrusted networks**
> You have several Azure Active Directory accounts. You need to ensure that users use multi-factor authentication (MFA) to access Azure apps from untrusted networks. What should you configure in Azure AD?
>
> A. access reviews  B. managed identities  C. entitlement management  D. conditional access

**A: D — Conditional Access.**
Conditional Access evaluates conditions (network location, device compliance, user group, sign-in risk) and enforces policies such as requiring MFA. You define named locations (trusted IP ranges) and create a policy: "if location = untrusted → require MFA." Access reviews = audit who has access. Managed identities = service-to-service auth (no user MFA). Entitlement management = access request/approval lifecycle.

---

**Q24 — Reference Key Vault secrets across all project pipelines without storing values**
> You have an Azure DevOps project (Project1), an Azure subscription (Sub1), and an Azure Key Vault (vault1). You need to ensure you can reference vault1 secret values in all pipelines of Project1. The solution must prevent values from being stored in the pipelines.
> A. Create a variable group in Project1  B. Add a secure file  C. Modify pipeline security settings  D. Configure the security policy of Contoso

**A: A — Create a variable group in Project1.**
Variable groups in Azure DevOps Library can be **linked directly to Azure Key Vault**. Secrets are fetched from Key Vault at runtime and injected as masked pipeline variables — values are never stored in the pipeline definition. All pipelines in Project1 can reference the same variable group. **Secure files** store binary files (certificates, keystores), not string secrets. **Pipeline security settings** control access permissions, not secret sourcing. **Contoso security policy** is org-level governance, not Key Vault integration.

---

**Q16 — Dynamic config: no restart needed**
> You want to update a configuration value in Azure App Configuration and have the running application pick it up without restarting. How do you implement this?

**A:** Use the **sentinel key pattern**. The application polls a designated sentinel key. When the sentinel key changes, the app reloads all configuration from App Configuration. This enables runtime config updates without redeployment or restart.

---

**Q25 — HOTSPOT: Secrets storage for mobile app calling service with basic auth over HTTPS**
> You need to configure a cloud service to store the secrets required by mobile applications to call a share pricing service. The service only supports basic authentication over HTTPS.
>
> - Required secrets: [Certificate / Personal access token / Shared Access Authorization token / Username and password]
> - Storage location: [Azure Data Lake / Azure Key Vault / Azure Storage with HTTP access / Azure Storage with HTTPS access]

**A:**
- **Required secrets: Username and password** — basic authentication uses a username + password credential pair. Certificate = certificate-based auth. PAT = Azure DevOps/GitHub API tokens. Shared Access Authorization = Azure Storage SAS.
- **Storage location: Azure Key Vault** — purpose-built Azure service for securely storing and retrieving secrets (passwords, connection strings, certificates) with access policies, audit logging, and secret versioning. Azure Data Lake = big data storage. Azure Storage with HTTP = insecure (plaintext transport). Azure Storage with HTTPS = secure file storage but no secrets management capabilities.

> **Exam tip:** Secrets (passwords, API keys) → **Azure Key Vault**. Configuration values → Azure App Configuration. Files (certs, keystores) → Secure Files in Azure Pipelines Library. Basic auth = username + password (not certificate, not token).

---

## Quick Reference: Exam Checklist

### Deployment Patterns
- [ ] Know all 6 patterns and when to use each: Big Bang, Blue-Green, Rolling, Canary, Ring, Feature Flag
- [ ] Blue-green = swap slots; instant rollback; slot-specific settings don't swap
- [ ] Canary = small % first; Traffic Manager weighted routing; App Service traffic %
- [ ] Dark launching = code runs in prod but invisible to users; shadow deployment
- [ ] A/B testing = business KPI comparison; user segmentation; weeks-long
- [ ] Ring deployment = internal → beta → general; gates between rings
- [ ] Feature toggle types: Release, Experiment, Ops, Permission, Infrastructure
- [ ] Toggle debt = accumulated unused toggles; must clean up promptly

### Identity Management
- [ ] GitHub roles: Read, Triage, Write, Maintain, Admin
- [ ] GitHub outside collaborator = specific repo access, not org member
- [ ] Azure DevOps groups: Readers, Contributors, Project Administrators
- [ ] Stakeholder = free; work items only; no repos/pipelines
- [ ] Service principal = any platform, requires credential, manual management
- [ ] Managed identity = Azure resources only, no credentials, Azure-managed
- [ ] System-assigned = one resource; User-assigned = shareable across resources
- [ ] Workload Identity Federation = OIDC; no secrets; recommended for GitHub Actions + Azure Pipelines
- [ ] Managed identity → Key Vault: assign `Key Vault Secrets User` RBAC role

### App Configuration & Key Vault
- [ ] Azure App Configuration = non-secret config (key-value, labels, feature flags)
- [ ] Azure Key Vault = secrets, keys, certificates
- [ ] Labels in App Configuration = environment differentiation (Dev/Prod labels on same key)
- [ ] Key Vault RBAC (recommended) vs Access Policies (legacy)
- [ ] Key Vault Reference in App Settings = secret stays in KV; App Service resolves at runtime
- [ ] Secure Files = certificate/key files; Variable Groups = string values/secrets
- [ ] Dynamic config = sentinel key pattern; no restart needed
- [ ] Feature flags in App Configuration: TimeWindowFilter, TargetingFilter, CustomFilter
- [ ] Inner loop = developer; outer loop = pipeline/system

---

*Sources: [MS Learn Secure CD Path](https://learn.microsoft.com/en-us/training/paths/az-400-implement-secure-continuous-deployment/) | [AZ-400 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-400)*
