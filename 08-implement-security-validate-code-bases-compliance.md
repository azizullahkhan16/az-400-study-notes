# AZ-400 Study Notes: Implement Security and Validate Code Bases for Compliance

**Learning Path:** Implement security and validate code bases for compliance
**Exam Weight:** "Implement security and validate code bases for compliance" = **10–15%** of exam
**MS Learn Path:** [AZ-400: Implement security and validate code bases for compliance](https://learn.microsoft.com/en-us/training/paths/az-400-implement-security-validate-code-bases-compliance/)
**Modules Covered:** 4

---

## Table of Contents

1. [Introduction to Secure DevOps](#1-introduction-to-secure-devops)
2. [Implement Open-Source Software](#2-implement-open-source-software)
3. [Software Composition Analysis](#3-software-composition-analysis)
4. [Security Monitoring and Governance](#4-security-monitoring-and-governance)
5. [Additional Exam Topics](#5-additional-exam-topics)
6. [Past Exam Scenario Bank](#6-past-exam-scenario-bank)
7. [Quick Reference Checklist](#7-quick-reference-checklist)

---

## 1. Introduction to Secure DevOps

### What is DevSecOps?

**DevSecOps** = integrating security into every stage of the DevOps lifecycle — not bolted on at the end.

**Shift-left security:** Move security testing earlier in the pipeline (to the left on the timeline). Bugs found early are cheaper to fix.

```
Shift-left:  Design → Code → Build → Test → Deploy → Operate
Security at:  ↑         ↑      ↑       ↑       ↑         ↑
             Threat   SAST   SCA    DAST   Secrets   Monitor
             Modeling
```

> **Exam tip:** DevSecOps is about making security **everyone's responsibility** throughout development, not just the security team's job at the end.

### Common Vulnerability: SQL Injection

**SQL injection** is one of the most common and dangerous web vulnerabilities.

**How it works:** Attacker inserts malicious SQL code into user input that gets executed by the database.

```sql
-- Vulnerable query (string concatenation)
SELECT * FROM Users WHERE username = '" + username + "' AND password = '" + password + "'

-- Attacker input: ' OR '1'='1
-- Result: SELECT * FROM Users WHERE username = '' OR '1'='1' ...
-- This bypasses authentication entirely
```

**Prevention:**
- **Parameterized queries** (prepared statements)
- **Stored procedures** with parameters
- **Input validation** (whitelist, not blacklist)
- **ORM frameworks** (Entity Framework, Hibernate)

> **Exam tip:** SQL injection is specifically called out in the module. The fix is **parameterized queries** (not string concatenation). CodeQL can detect SQL injection patterns automatically.

### Secure DevOps Pipeline Stages

Security validation at each pipeline stage:

| Stage | Security Activity | Tool |
|---|---|---|
| **Pre-commit** | Secrets detection, lint | git-secrets, detect-secrets, pre-commit hooks |
| **Pull request** | Static code analysis (SAST) | CodeQL, SonarQube — runs on code changes before merge |
| **Build (CI)** | Static code analysis (SAST) + SCA | CodeQL, SonarQube; WhiteSource Bolt, Dependabot |
| **Test (DAST)** | Dynamic testing of running application | OWASP ZAP, Burp Suite |
| **Continuous delivery** | Penetration testing | Requires a deployed target (staging environment) |

> **Exam tip (DRAG DROP):** Match security tool to pipeline stage: **Pull request → Static code analysis** (SAST before merge); **Continuous integration → Static code analysis** (automated SAST in build); **Continuous delivery → Penetration testing** (tests running deployed app). **Threat modeling** is a design-phase activity — not a CI/CD pipeline stage tool.
| **Deploy** | IaC security scanning | Checkov, Terrascan, tfsec |
| **Operate** | Runtime monitoring, threat detection | Defender for Cloud, Application Insights |

### Security Testing Types

| Type | Description | When | Example Tool |
|---|---|---|---|
| **SAST** (Static Application Security Testing) | Analyze source code without running it | Build time | CodeQL, SonarQube |
| **DAST** (Dynamic Application Security Testing) | Test the running application from outside | Post-deploy | OWASP ZAP, Burp Suite |
| **IAST** (Interactive Application Security Testing) | Instruments running app from inside during testing | Test time | Contrast Security |
| **SCA** (Software Composition Analysis) | Scan dependencies for vulnerabilities and licenses | Build time | **WhiteSource Bolt** (Mend), Dependabot, Snyk |
| **Secrets Scanning** | Detect hardcoded credentials in code | Pre-commit / CI | GitHub Secret Scanning, detect-secrets |

> **Exam tip:** SAST = code is **not running** (analyzes source code). DAST = application **is running** (tests from outside, like an attacker). Know the difference — it's a common exam question.

> **Exam tip:** "Continuous inspection of the codebase to locate common code patterns known to be problematic" = **SonarCloud** (cloud-hosted) or **SonarQube** (self-hosted). These are the purpose-built tools for continuous static code quality inspection — detecting code smells, bugs, and security hotspots. Visual Studio test plans = manual test management. Gradle = Java build tool. JavaScript task runners (Grunt/Gulp) = build automation.

### WhiteSource Bolt (Mend) — Azure DevOps Integration

**WhiteSource Bolt** (now rebranded as **Mend Bolt**) is a free SCA tool specifically designed for Azure DevOps integration:

- Scans open-source libraries for **known security vulnerabilities** (CVE database)
- Also checks **open-source license compliance**
- Integrates into Azure DevOps as a **build task** added to the pipeline
- Generates a vulnerability report as part of the build results

**How to set it up in Azure DevOps:**
1. Install the WhiteSource Bolt extension from the Azure DevOps Marketplace
2. **Create a build task** in your pipeline that runs WhiteSource Bolt
3. The task scans all open-source dependencies during the build

| Object to create | Service to use |
|---|---|
| **A build task** | **WhiteSource Bolt** |

> **Exam tip (tested — HOTSPOT):** "Scan open-source libraries for known security vulnerabilities in an Azure DevOps build pipeline" → create **a build task** using **WhiteSource Bolt**. It is NOT a deployment task (no deployment needed to scan), NOT an artifacts repository (that stores packages, doesn't scan them). Bamboo, CMake, and Chef are unrelated tools (CI server, build system, configuration management respectively).

**Filtering WhiteSource Bolt to scan production dependencies only (Node.js):**

WhiteSource Bolt scans whatever is present in `node_modules`. To limit scanning to production dependencies only:

| Step | Action | Why |
|---|---|---|
| 1 | **Modify `devDependencies` in package.json** | Ensure all dev-only libraries are classified under `devDependencies`, not `dependencies` |
| 2 | **Run `npm install --production`** | Installs only `dependencies`, skipping `devDependencies` — `node_modules` contains only production packages |

WhiteSource Bolt then scans only the production `node_modules` contents.

> **Exam tip:** These two actions work together — package.json classification + `--production` flag. Setting a WhiteSource policy action to "Reassign" (B) re-categorizes the violation but does NOT exclude libraries from scanning. Scanning `node_modules` directly (D) is irrelevant if devDependencies are still installed there.

### Key Validation Points in the Pipeline

**Continuous security validation** means security checks run automatically at multiple points:

1. **Code commit** — pre-commit hooks for secrets and linting
2. **Pull request** — CodeQL scan, dependency review, required reviewers
3. **Build** — SAST, SCA, container image scanning
4. **Test environment deploy** — DAST scan against running application
5. **Production deploy** — gates requiring security sign-off
6. **Production** — runtime monitoring, anomaly detection

### Threat Modeling

**Threat modeling** identifies security risks **before** implementation begins.

**STRIDE framework** — categorizes threats:

| Letter | Threat | Violated Property | Example |
|---|---|---|---|
| **S** | Spoofing | Authentication | Attacker impersonates a legitimate user |
| **T** | Tampering | Integrity | Attacker modifies data in transit |
| **R** | Repudiation | Non-repudiation | User denies performing an action |
| **I** | Information Disclosure | Confidentiality | Sensitive data leaked in error messages |
| **D** | Denial of Service | Availability | Flood requests to crash the service |
| **E** | Elevation of Privilege | Authorization | Attacker gains admin rights from user role |

**Threat modeling process:**
1. **Diagram** the system (data flow diagrams)
2. **Identify** threats using STRIDE
3. **Mitigate** each threat (controls, architecture changes)
4. **Validate** mitigations are effective

> **Exam tip:** STRIDE is the key threat modeling framework tested. Know what each letter stands for, which security property it violates, and a simple example.

### GitHub CodeQL

**CodeQL** is GitHub's semantic code analysis engine — it queries code like a database to find vulnerability patterns.

**How it works:**
1. Builds a CodeQL database from source code
2. Runs pre-built or custom queries (`.ql` files) against the database
3. Reports findings as code scanning alerts

**What CodeQL detects:**
- SQL injection
- Cross-site scripting (XSS)
- Path traversal
- Insecure deserialization
- Hardcoded credentials
- Many other CWE patterns

**CodeQL in GitHub Actions:**
```yaml
name: CodeQL Analysis
on: [push, pull_request]

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: csharp  # or javascript, python, java, cpp, go, ruby
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

> **Exam tip:** CodeQL requires `permissions: security-events: write` in the GitHub Actions workflow to upload results to the Security tab. Without this permission, the scan runs but results won't appear.

**Supported languages:** C/C++, C#, Go, Java/Kotlin, JavaScript/TypeScript, Python, Ruby, Swift

---

## 2. Implement Open-Source Software

### Why Open Source is Ubiquitous

Modern software is built largely on open-source components:
- ~70–90% of code in typical enterprise apps is open-source
- Open source = faster development, community-tested components
- But it introduces: security vulnerabilities, license obligations, supply chain risks

### Corporate Concerns with Open-Source Software

| Concern | Description |
|---|---|
| **Security vulnerabilities** | OSS components may contain known CVEs that attackers exploit |
| **License compliance** | Using OSS with incompatible licenses can create legal exposure |
| **Supply chain attacks** | Malicious actors compromise legitimate packages (e.g., npm typosquatting) |
| **Outdated dependencies** | Teams use old versions with known vulnerabilities |
| **Transitive dependencies** | Indirect dependencies (dependencies of dependencies) also carry risk |

### Open-Source Licenses (Exam Favorite)

Licenses fall into two main categories:

| Category | Description | Examples | Commercial Use |
|---|---|---|---|
| **Permissive** | Few restrictions; use freely in commercial products | MIT, Apache 2.0, BSD | Safe for commercial use |
| **Copyleft (viral)** | Derivative works must also be open-source under same license | GPL (v2, v3), LGPL, AGPL | Risky for commercial proprietary software |

**Key licenses in detail:**

| License | Type | Key Requirements |
|---|---|---|
| **MIT** | Permissive | Include copyright notice + license text. No other restrictions. |
| **Apache 2.0** | Permissive | Include copyright + license + NOTICE file. Includes patent grant. |
| **BSD 2-Clause** | Permissive | Include copyright + license text. |
| **BSD 3-Clause** | Permissive | BSD 2-Clause + cannot use author's name for endorsement. |
| **GPL v2** | Copyleft | Derivative works must be GPL v2 licensed and source released. |
| **GPL v3** | Copyleft | Like GPL v2 + anti-tivoization clause. Not compatible with GPL v2. |
| **LGPL** | Weak copyleft | Link to LGPL library OK in proprietary code; modifications to the library itself must be open-sourced. |
| **AGPL** | Strong copyleft | Like GPL v3 + using over network = must provide source (closes "SaaS loophole"). |
| **MPL 2.0** | Weak copyleft | File-level copyleft; can combine with proprietary code in different files. |

> **Exam tip:** GPL is the most commonly tested license. **Using a GPL library in a commercial product** = you must open-source your entire product under GPL. LGPL allows linking without this requirement. MIT and Apache 2.0 are the "safe" choices for commercial products.

### License Ratings / Risk Levels

Organizations rate licenses by risk:

| Risk Level | Licenses | Action |
|---|---|---|
| **Green (Safe)** | MIT, Apache 2.0, BSD | Can use freely in commercial products |
| **Yellow (Caution)** | LGPL, MPL | Review carefully; usually OK with proper usage |
| **Red (Restricted)** | GPL, AGPL | Legal review required; may prohibit commercial use |

### Managing OSS in Enterprise

**Strategies:**
1. **Inventory all OSS components** — know what you're using and at what version
2. **License compliance check** — automated scanning (Mend, FOSSA, Black Duck)
3. **Vulnerability scanning** — GitHub Dependabot, Snyk, OWASP Dependency-Check
4. **Approved package list** — maintain a curated list of permitted OSS components
5. **Private feed with upstream** — Azure Artifacts proxies public registries, giving control over what enters the environment
6. **Automated updates** — Dependabot opens PRs for security updates automatically

---

## 3. Software Composition Analysis

### What is Software Composition Analysis (SCA)?

**SCA** automatically identifies open-source components in your codebase and checks them against:
- Known vulnerabilities (CVE databases — NVD, OSV)
- License compliance requirements

**SCA vs SAST:**
- **SAST** = analyzes *your* custom code for security issues
- **SCA** = analyzes *third-party dependencies* for vulnerabilities and licenses

### CVE and CVSS

**CVE (Common Vulnerabilities and Exposures):** A unique identifier for a publicly disclosed security vulnerability (e.g., `CVE-2021-44228` = Log4Shell).

**CVSS (Common Vulnerability Scoring System):** Numeric severity score for a CVE.

| CVSS Score | Severity |
|---|---|
| 0.0 | None |
| 0.1–3.9 | Low |
| 4.0–6.9 | Medium |
| 7.0–8.9 | High |
| 9.0–10.0 | Critical |

> **Exam tip:** Prioritize remediation by CVSS score — Critical (9.0–10.0) first. Also consider exploitability and whether the vulnerability is reachable in your specific usage.

### GitHub Dependabot

**Dependabot** is GitHub's automated dependency management tool with three capabilities:

| Feature | What It Does |
|---|---|
| **Dependabot Alerts** | Notifies about vulnerable dependencies (based on GitHub Advisory Database) |
| **Dependabot Security Updates** | Automatically opens PRs to update vulnerable dependencies |
| **Dependabot Version Updates** | Opens PRs to keep dependencies up to date (not just security) |

**Configuration file:** `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "daily"
```

> **Exam tip:** Dependabot needs the `.github/dependabot.yml` file to configure version update schedules and ecosystems. Security alerts work without this file (enabled in repo settings). The `open-pull-requests-limit` controls how many PRs Dependabot opens at once.

### GitHub Dependency Review

**Dependency Review** is a GitHub feature that shows the security impact of dependency changes in a pull request — before merging.

- Shows new dependencies added, removed, or version changed
- Highlights new vulnerabilities introduced
- Shows license changes

Enable via the `dependency-review-action` in GitHub Actions:
```yaml
- uses: actions/dependency-review-action@v4
  with:
    fail-on-severity: high  # Fail if high/critical vulnerabilities found
    deny-licenses: GPL-2.0, GPL-3.0  # Block specific licenses
```

### SCA Tools Comparison

| Tool | Type | Key Feature |
|---|---|---|
| **GitHub Dependabot** | SaaS (GitHub native) | Automated PRs for vulnerability fixes; GitHub Advisory Database |
| **Snyk** | SaaS / CI plugin | Deep transitive dependency analysis; fix suggestions; IDE integration |
| **Mend (formerly WhiteSource)** | SaaS / CI plugin | License compliance + vulnerability; policy enforcement; remediation guidance |
| **OWASP Dependency-Check** | Open-source / CLI | Scans dependencies against NVD CVE database; free |
| **Black Duck** | Enterprise SaaS | Comprehensive OSS governance; legal review workflow |
| **FOSSA** | SaaS | License compliance focused; legal team workflows |

> **Exam tip:** For AZ-400, know these tools and their primary focus:
> - **Dependabot** = GitHub-native, automatic PRs
> - **Snyk** = developer-friendly, IDE + CI, fix suggestions
> - **Mend/WhiteSource** = enterprise license compliance + vulnerabilities
> - **OWASP Dependency-Check** = free, open-source, NVD-based
> - **Black Duck** = enterprise OSS governance and **license compliance** — the exam answer when the question asks about ensuring open source libraries comply with licensing standards. NuGet, Maven, and Helm are package managers, not compliance tools.

### Integrating SCA into Azure Pipelines

**OWASP Dependency-Check in Azure Pipelines:**
```yaml
- task: dependency-check-build-task@6
  displayName: 'OWASP Dependency Check'
  inputs:
    projectName: '$(Build.DefinitionName)'
    scanPath: '$(Build.SourcesDirectory)'
    format: 'HTML,JUNIT'
    failOnCVSS: '7'  # Fail build if any finding >= 7.0 CVSS
```

**Mend (WhiteSource) in Azure Pipelines:**
```yaml
- task: WhiteSource@21
  inputs:
    cwd: '$(System.DefaultWorkingDirectory)'
```

**Snyk in Azure Pipelines:**
```yaml
- task: SnykSecurityScan@1
  inputs:
    serviceConnectionEndpoint: 'SnykConnection'
    testType: 'app'
    severityThreshold: 'high'
    failOnIssues: true
```

### Container Image Scanning

Container images can contain vulnerable base OS packages, libraries, and application code.

**Scanning approaches:**

| Approach | Description | Tool |
|---|---|---|
| **Registry scanning** | Scan images when pushed to registry | Microsoft Defender for Container Registry, Trivy |
| **Pipeline scanning** | Scan during build before pushing | Trivy, Snyk Container, Aqua |
| **Runtime scanning** | Monitor running containers | Defender for Containers, Falco |

**Trivy in Azure Pipelines:**
```yaml
- script: |
    trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:$(Build.BuildId)
  displayName: 'Trivy Container Scan'
```

> **Exam tip:** Container image scanning should happen **before pushing to the registry** (in the pipeline) so vulnerable images never reach the registry. Defender for Container Registry also scans on push and on a schedule.

### Interpreting SCA Alerts

When prioritizing remediation from SCA scan results:

1. **CVSS score** — Critical/High first
2. **Exploitability** — Is there a known exploit? Is it being actively exploited?
3. **Reachability** — Is the vulnerable code path actually called by your application?
4. **Fix available** — Does a patched version exist?
5. **Breaking change** — Will updating to the fix cause compatibility issues?

> **Exam tip:** "A vulnerability with CVSS 9.5 exists in a transitive dependency but the vulnerable function is never called in our app" — the vulnerability is still real, but **reachability** analysis de-prioritizes it vs. directly callable vulnerabilities.

---

## 4. Security Monitoring and Governance

### Pipeline Security Best Practices

**Service connections:**
- Use **Service Principal** or **Managed Identity** (not personal credentials)
- Apply **least privilege** — grant only permissions needed for the specific task
- **Do NOT** select "Grant access permission to all pipelines" — authorize each pipeline individually
- Scope service connections per environment (separate connections for Dev/Staging/Prod)
- Regularly audit and rotate service principal credentials

**Pipeline permissions:**
- Use **pipeline-level approval gates** before deploying to production
- Restrict who can edit pipeline YAML files (branch policies on pipeline files)
- Protect variable groups with **pipeline permissions**
- Store secrets in **Azure Key Vault**, not pipeline variables

**Agent security:**
- Microsoft-hosted agents: fresh VM per job, no persistence between runs
- Self-hosted agents: risk of **poison pipeline execution** — compromised shared state between pipeline runs
- Apply branch policies to prevent untrusted code from running on self-hosted agents

> **Exam tip:** **Poison pipeline execution (PPE)** = attacker modifies pipeline code to exfiltrate secrets or compromise the build agent. Mitigate by restricting who can push to branches that trigger pipelines, and using `pull_request_target` carefully on public repos.

### Azure AD Privileged Identity Management (PIM)

**PIM** provides just-in-time (JIT) privileged access management for Azure AD roles and Azure resource roles.

**Key PIM capabilities:**
- **Time-limited access** — privileged roles are active only for a defined time window
- **Approval workflow** — activation requires approval from a designated approver
- **MFA on activation** — require MFA before elevating to a privileged role
- **Access reviews** — periodic review of who has privileged access
- **Audit history** — full log of all role activations

**License requirement — critical exam fact:**

| Feature | Required License |
|---|---|
| Azure AD Free / P1 | Basic role assignments only — no JIT, no approval, no time limits |
| **Azure AD Premium Plan 2** | **Full PIM** — JIT access, approval workflows, time limits, access reviews |

> **Exam tip (tested):** "Deploy PIM with time limits and approval requirements — tenant has Azure AD Premium Plan 1 — what should you do first?" → **Upgrade the license to Azure AD Premium Plan 2**. PIM requires P2. Configuring alerts, MFA, or notifications are PIM settings that can only be configured *after* P2 is in place. Without the upgrade, PIM is not available at all — so the first step is always the license upgrade.

### Microsoft Defender for Cloud

**Microsoft Defender for Cloud** (formerly Azure Security Center + Azure Defender) is a cloud security posture management (CSPM) and threat protection platform.

**Key capabilities:**

| Capability | Description |
|---|---|
| **Secure Score** | Numeric score (0–100) representing your security posture. Higher = more secure. |
| **Security Recommendations** | Actionable steps to improve secure score. Each recommendation has a score impact. |
| **Security Alerts** | Real-time threat detection alerts (e.g., suspicious process, brute force attempt) |
| **Regulatory Compliance** | Compliance dashboard for standards: PCI DSS, ISO 27001, SOC 2, CIS |
| **Defender Plans** | Enhanced threat protection per workload type (Servers, Containers, SQL, etc.) |

**Secure Score:**
- Calculated from completed security controls
- Each control has a max score; completing all recommendations in a control gives full points
- Score = (points achieved / total possible points) × 100

> **Exam tip:** Secure Score is NOT based on the number of recommendations completed — it's based on **security controls**. Multiple recommendations can belong to one control. Completing any recommendation in a control gives partial credit.

### Defender for Cloud Usage Scenarios

| Scenario | How Defender for Cloud Helps |
|---|---|
| Azure VM has vulnerability | Recommendations: enable Defender for Endpoint, patch OS, enable JIT |
| SQL server without TDE | Recommendation: enable Transparent Data Encryption |
| Storage account allows public access | Recommendation: disable public blob access |
| Suspicious login activity | Alert: "Possible brute force attack on Azure SQL" |
| Check compliance with PCI DSS | Regulatory Compliance blade: see passing/failing controls |

### Defender for Cloud — Vulnerability Assessment Coverage (Exam Favorite)

Not all Azure resource types support vulnerability assessments in Defender for Cloud. The exam tests which resource types are covered:

| Resource Type | Vulnerability Assessment? | Notes |
|---|---|---|
| **Azure VMs (Windows / Linux)** | ✅ Yes | Via Qualys or Microsoft Defender Vulnerability Management integration |
| **Container images in ACR** | ✅ Yes | Defender for Containers scans images for OS + package CVEs |
| Azure Key Vault | No | Defender for Key Vault provides **threat protection** (anomalous access alerts) — not VA |
| Azure Log Analytics workspace | No | Monitoring/logging service — no vulnerability surface |
| Azure Active Directory | No | Microsoft-managed; identity threats handled by AAD Identity Protection |
| Azure SQL Database | ✅ Yes | Defender for SQL includes vulnerability assessment |
| App Service | No | Defender for App Service provides threat detection, not VA |

> **Exam tip:** Vulnerability assessments in Defender for Cloud apply to **VMs** and **container images in ACR**. Key Vault gets threat protection (alerts), not vulnerability assessment. Azure AD and Log Analytics workspaces are not assessed.

### Azure Policy

**Azure Policy** enforces organizational standards and assesses compliance across Azure resources.

**Policy components:**
- **Policy Definition** — a rule (JSON) that defines what to check and what to do
- **Initiative (Policy Set)** — a group of related policy definitions (e.g., "CIS Azure Benchmark")
- **Assignment** — applying a definition or initiative to a scope (management group, subscription, resource group)
- **Compliance** — reports how many resources are compliant/non-compliant

### Azure Policy Effects (Exam Favorite)

| Effect | Behavior | When Applied |
|---|---|---|
| **Disabled** | Policy is off — no evaluation | Never |
| **Audit** | Log non-compliant resources; does NOT block | Post-creation |
| **AuditIfNotExists** | Audit if a related resource does NOT exist | Post-creation |
| **Deny** | Block resource creation/update if non-compliant | At creation/update time |
| **Append** | Add fields to a resource (e.g., add tags) | At creation/update |
| **Modify** | Add/change tags on existing resources | At creation/update |
| **DeployIfNotExists** | Deploy a related resource if it doesn't exist | Post-creation |
| **DenyAction** | Block specific resource actions | At action time |

**Policy effect evaluation order:**
`Disabled → Append/Modify → Deny → Audit/AuditIfNotExists/DeployIfNotExists`

> **Exam tip (most tested effects):**
> - **Deny** = preventive; blocks the operation before it happens
> - **Audit** = detective; allows it but logs it as non-compliant
> - **DeployIfNotExists** = remediation; deploys something (e.g., enable Defender for SQL) if missing
> - **AuditIfNotExists** = detective; audit if a child resource (e.g., diagnostic setting) is missing

**Example scenarios:**

| Requirement | Policy Effect |
|---|---|
| Prevent VMs from being created without encryption | **Deny** |
| Report which storage accounts don't have HTTPS-only enabled | **Audit** |
| Automatically enable diagnostic settings on new resources | **DeployIfNotExists** |
| Report resources that don't have a specific child resource | **AuditIfNotExists** |

### Policy Initiatives

An **initiative** (policy set) groups multiple related policy definitions:
- Assign one initiative instead of dozens of individual policies
- Built-in initiatives: "Enable Azure Security Center" (ASC default), CIS benchmarks, PCI DSS, HIPAA

```json
// Initiative assignment example
{
  "properties": {
    "displayName": "CIS Azure Foundations Benchmark",
    "policyDefinitions": [
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/..." },
      { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/..." }
    ]
  }
}
```

### Resource Locks

**Resource locks** prevent accidental deletion or modification of critical Azure resources.

| Lock Type | Prevents | Allows |
|---|---|---|
| **CanNotDelete (Delete lock)** | Deleting the resource | Read and modify |
| **ReadOnly** | Deleting AND modifying the resource | Read only |

**Lock hierarchy:** Locks are inherited. A lock on a resource group applies to all resources in it.

**Who can manage locks:** Only Owner or User Access Administrator roles can create/delete locks.

> **Exam trap:** A **ReadOnly** lock on a storage account prevents listing access keys (a write operation internally). Users cannot add/remove resources in a locked resource group with ReadOnly.

**Apply lock via Azure CLI:**
```bash
# Apply Delete lock to a resource group
az lock create \
  --name "LockRG" \
  --resource-group "myRG" \
  --lock-type CanNotDelete

# Apply ReadOnly lock
az lock create \
  --name "ReadOnlyLock" \
  --resource-group "myRG" \
  --lock-type ReadOnly
```

> **Exam tip:** Locks can be applied at **subscription, resource group, or resource** level. Child resources inherit parent locks. Locks override RBAC — even an Owner cannot delete a resource with a Delete lock without first removing the lock.

### Microsoft Defender for Identity

**Microsoft Defender for Identity** (formerly Azure ATP — Advanced Threat Protection) detects threats in **on-premises Active Directory** environments.

**What it detects:**
- Lateral movement attacks (Pass-the-Hash, Pass-the-Ticket)
- Domain dominance (Golden Ticket, DCSync)
- Reconnaissance attacks
- Compromised credentials

**How it works:**
- Deploy **Defender for Identity sensors** on domain controllers
- Sensors capture and analyze Active Directory traffic
- Cloud service analyzes for attack patterns using behavioral analytics

> **Exam tip:** Defender for Identity is specifically for **Active Directory / on-premises identity** threats. It's different from:
> - **Defender for Cloud** = cloud resource security posture
> - **Microsoft Entra ID Protection** = cloud identity risk (Azure AD)
> - **Defender for Endpoint** = endpoint/device threats

### GitHub Advanced Security (GHAS)

**GitHub Advanced Security** provides three security features:

| Feature | What It Does | Where Results Appear |
|---|---|---|
| **Code Scanning (CodeQL)** | SAST — analyzes code for vulnerabilities using CodeQL queries | Security → Code scanning alerts |
| **Secret Scanning** | Detects hardcoded secrets (API keys, tokens, passwords) in code and commits | Security → Secret scanning alerts |
| **Dependency Review** | Shows vulnerable dependencies added in PRs before merging | PR check / Dependency review tab |

**Secret Scanning push protection:**
- When enabled, **blocks pushes** that contain detected secrets before they're committed
- Developer gets an error message and must remove the secret to push
- Can bypass push protection with a reason (for false positives)

> **Exam tip:** Secret scanning has TWO modes:
> 1. **Alert mode** — scans historical commits; notifies but doesn't block
> 2. **Push protection** — blocks new pushes containing secrets BEFORE they're committed

**GHAS for Azure DevOps:**
- Also available for Azure Repos (not just GitHub)
- Provides: Code Scanning, Secret Scanning, Dependency Scanning in Azure DevOps

### Defender for DevOps (Microsoft Security DevOps)

**Microsoft Defender for DevOps** connects Defender for Cloud with your DevOps environment (GitHub, Azure DevOps) to provide unified security posture.

**Key capabilities:**
- Scans IaC templates (ARM, Bicep, Terraform) for misconfigurations
- Integrates findings into Azure portal Security Center
- Shows security posture across all connected repos
- Annotates PRs with security findings

**MicrosoftSecurityDevOps@1 pipeline task:**
```yaml
- task: MicrosoftSecurityDevOps@1
  displayName: 'Microsoft Security DevOps'
  inputs:
    categories: 'IaC,secrets'  # or: code,artifacts,containers
```

**Tools it runs internally:**
| Tool | Purpose |
|---|---|
| **Bandit** | Python SAST |
| **ESLint (security rules)** | JavaScript SAST |
| **Terrascan** | IaC security (Terraform, ARM, k8s) |
| **Trivy** | Container + IaC scanning |
| **Credscan** | Secret detection |

### Integrating GHAS with Defender for Cloud

When you connect GitHub organization to Defender for Cloud:
1. Navigate to Defender for Cloud → Environment Settings → Add GitHub
2. Authorize Defender for Cloud GitHub App
3. Findings from CodeQL, secret scanning, and Dependabot flow into Defender for Cloud
4. Unified security posture view across cloud resources AND DevOps repos

---

## 5. Additional Exam Topics

### Technical Debt Management (Exam Series)

**Technical debt** is accumulated shortcuts, poor design decisions, and code quality issues that slow down future development — adding new features takes longer over time.

**Primary causes of technical debt:**
- High code coupling (modules tightly interdependent)
- Dependency cycles (circular dependencies between modules)
- Lack of automated tests
- No code reviews or quality gates
- Outdated dependencies

**Solutions that reduce accumulated technical debt:**

| Proposed solution | Meets goal? | Reason |
|---|---|---|
| **Reduce code coupling + dependency cycles** | ✅ **Yes** | Directly reduces complexity that slows feature development; decoupled modules are independently modifiable |
| **Implement code reviews** | ✅ Yes | Prevents new debt from accumulating; catches quality issues early |
| **Use SonarQube/SonarCloud** | ✅ Yes | Measures technical debt objectively; tracks quality gates; enforces standards |
| **Implement automated security testing** | ❌ No | Detects security vulnerabilities — not a technical debt reduction strategy |
| **Implement CI** | ✅ Yes (with quality gates) | CI with code analysis tools surfaces debt continuously |

> **Exam tip:** "Reducing code coupling and dependency cycles" **does** meet the goal of reducing technical debt — high coupling is a primary source of debt. Some exam dumps incorrectly mark this as No. Coupling makes features harder to add because changes cascade through tightly-linked modules.

---

### OWASP Top 10 (2021) — Most Tested Security Risks

| Rank | Risk | Description |
|---|---|---|
| A01 | **Broken Access Control** | Users can act outside intended permissions |
| A02 | **Cryptographic Failures** | Sensitive data exposed due to weak/missing encryption |
| A03 | **Injection** | SQL injection, command injection, LDAP injection |
| A04 | **Insecure Design** | Flaws in design/architecture (not just implementation) |
| A05 | **Security Misconfiguration** | Default settings, open cloud storage, verbose error messages |
| A06 | **Vulnerable/Outdated Components** | Using libraries/frameworks with known vulnerabilities |
| A07 | **Auth Failures** | Broken authentication, weak passwords, missing MFA |
| A08 | **Software/Data Integrity Failures** | CI/CD pipeline integrity, insecure deserialization |
| A09 | **Security Logging Failures** | Insufficient logging/monitoring to detect breaches |
| A10 | **SSRF** | Server-Side Request Forgery — server fetches attacker-controlled URL |

> **Exam tip:** A06 (Vulnerable/Outdated Components) is directly addressed by SCA tools. A03 (Injection) is detected by SAST/CodeQL. A08 (CI/CD integrity) relates to pipeline security.

### Secrets Management in Pipelines

**Problem:** Secrets (passwords, API keys, tokens) must never be hardcoded in code or pipeline YAML.

**Solution hierarchy (best to worst):**

1. **Azure Key Vault** + Key Vault task in pipeline (best practice)
2. **Variable groups** linked to Key Vault in Azure DevOps
3. **Secret pipeline variables** (stored encrypted in Azure DevOps)
4. **Hardcoded in YAML** — NEVER acceptable

**Key Vault in Azure Pipelines:**
```yaml
- task: AzureKeyVault@2
  inputs:
    azureSubscription: 'MyServiceConnection'
    KeyVaultName: 'myKeyVault'
    SecretsFilter: 'connectionString,apiKey'
    RunAsPreJob: true

# Secrets are now available as pipeline variables
- script: echo "$(connectionString)"  # Masked in logs
```

### Pipeline Security — Protecting Agent Pools

**Self-hosted agents** require extra care:
- Run in a **dedicated agent pool** for sensitive pipelines (separate from general-purpose pools)
- Apply **pipeline permissions** to restrict which pipelines can use sensitive agent pools
- **Clean workspace** between builds: configure `workspace: clean: all`
- Do NOT cache secrets to disk on self-hosted agents

### Supply Chain Security

**Software supply chain attacks** target the development toolchain to inject malicious code:

**Examples:**
- **SolarWinds attack** — malicious code injected into build process
- **XZ Utils backdoor** — compromised maintainer added backdoor to widely used library
- **npm typosquatting** — malicious packages with names similar to popular ones

**Defenses:**
- Use Azure Artifacts upstream sources (control what packages enter your environment)
- Pin dependency versions (no floating `^` or `~` in production)
- Use lock files (`package-lock.json`, `packages.lock.json`)
- Sign packages (NuGet package signing, npm provenance)
- Code signing for build artifacts
- **SBOM (Software Bill of Materials)** — complete inventory of all components

### Azure Policy vs RBAC

| | Azure Policy | RBAC |
|---|---|---|
| **Purpose** | Enforce resource properties/state compliance | Control who can perform actions on resources |
| **Focus** | What resources look like | Who can do what |
| **Example** | "All VMs must use managed disks" | "Alice can create VMs in this RG" |
| **Evaluation** | After or during resource operations | At time of user request |
| **Override** | Deny policy blocks even Owners | RBAC permissions define capability |

> **Exam tip:** Azure Policy and RBAC are **complementary**. RBAC controls permissions; Policy controls properties. A user with Owner RBAC can still be blocked by a Deny policy.

### Regulatory Compliance Frameworks

Common compliance frameworks referenced in Defender for Cloud:

| Framework | Industry |
|---|---|
| **PCI DSS** | Payment card industry |
| **ISO 27001** | Information security management |
| **SOC 2** | Service organizations (SaaS) |
| **HIPAA** | Healthcare (US) |
| **CIS Benchmarks** | General security hardening |
| **NIST SP 800-53** | US government |
| **Azure Security Benchmark** | Microsoft's own best practices |

---

## 6. Past Exam Scenario Bank

**Q7.** Your Azure DevOps build pipeline uses approximately 50 open source libraries. You need to ensure all libraries comply with your company's licensing standards. Which service should you use?
> A. NuGet  B. Maven  C. Black Duck  D. Helm

**A: C — Black Duck.**
Black Duck is an enterprise SCA (Software Composition Analysis) tool that scans open source dependencies and reports license types, compliance violations, and security vulnerabilities. NuGet, Maven, and Helm are package managers — they do not perform license compliance analysis.

---

**Q6.** Your company uses: Windows Server 2019 container images in ACR / Azure VMs running Ubuntu / Azure Log Analytics workspace / Azure AD / Azure Key Vault. For which two resources can you receive vulnerability assessments in Azure Security Center?
> A. Azure Log Analytics workspace  B. Azure Key Vault  C. Azure VMs (Ubuntu)  D. Azure Active Directory  E. Windows Server 2019 container images in ACR

**A: C and E.**
Vulnerability assessments in Defender for Cloud (Security Center) cover **VMs** (via Qualys/MDVM) and **ACR container images** (via Defender for Containers). Key Vault provides threat protection (anomalous access alerts) — not VA. Log Analytics and Azure AD have no vulnerability assessment surface in Security Center.

---

**Q1.** A developer hardcoded a database connection string in a C# source file and pushed it to GitHub. GitHub sent a secret scanning alert. The fix was to remove it from the latest commit. Is the secret safe?

**A:** No. Even after removing from the latest commit, the secret still exists in **Git history**. The correct actions are:
1. **Rotate/invalidate** the credential immediately (assume it's compromised)
2. Use `git filter-repo` to purge the secret from all commit history
3. Force-push the cleaned history to GitHub
4. Move the secret to **Azure Key Vault** and reference it via the Key Vault task

---

**Q2.** You need to prevent any Azure storage account from being created in your subscription with public blob access enabled. Which Azure Policy effect do you use?

**A:** **Deny**. The Deny effect blocks the resource creation/update operation before it happens. Audit would allow the storage account to be created and then report it as non-compliant. Deny is the preventive control.

---

**Q3.** You use a GPL v2 library in your commercial .NET application. You plan to sell the application as a proprietary product. What license obligation applies?

**A:** GPL v2 is **copyleft (viral)**. Any software that uses a GPL v2 library and is distributed must also be released under GPL v2 with source code made available. Using GPL in a proprietary commercial product creates a significant legal risk. The correct action is to find an equivalent library with a **permissive license** (MIT, Apache 2.0) or obtain a commercial license.

---

**Q4.** Your team needs to block pushes to GitHub repositories if they contain AWS access keys, Azure SAS tokens, or GitHub PATs. Which feature provides this?

**A:** **GitHub Advanced Security — Secret Scanning with Push Protection**. Push protection intercepts the push before it completes, alerts the developer, and blocks the commit from being pushed. Without push protection, secret scanning only alerts AFTER the secret is already in the repo.

---

**Q5.** A DAST scan of your web application found a SQL injection vulnerability in `/api/search`. The code uses string concatenation to build SQL queries. What is the correct fix?

**A:** Replace string concatenation with **parameterized queries** (prepared statements):
```csharp
// Vulnerable
var query = "SELECT * FROM Products WHERE name = '" + searchTerm + "'";

// Fixed: parameterized query
var query = "SELECT * FROM Products WHERE name = @searchTerm";
cmd.Parameters.AddWithValue("@searchTerm", searchTerm);
```

---

**Q6.** Your team needs to automatically report which Azure VMs do not have the Log Analytics agent installed, without blocking VM creation. Which Azure Policy effect do you use?

**A:** **AuditIfNotExists**. This effect audits a resource if a related resource (the Log Analytics agent extension on the VM) does NOT exist. It reports non-compliance without blocking the VM creation. `DeployIfNotExists` would automatically install the agent, which may or may not be desired.

---

**Q7.** An attacker is testing your application's login page by submitting thousands of username/password combinations. According to STRIDE, what type of threat is this?

**A:** **Spoofing (S)** — the attacker is attempting to authenticate as a legitimate user. The specific attack is a **brute force / credential stuffing attack**. It violates the **Authentication** security property.

---

**Q8.** You want to prevent accidental deletion of a production Azure SQL Database while allowing regular read and write operations. Which resource lock type should you apply?

**A:** **CanNotDelete (Delete lock)**. This prevents deletion while allowing all read and write/modify operations. A ReadOnly lock would prevent writes to the database, making it unusable.

---

**Q9.** CodeQL is configured to scan your repository on every pull request. A PR is submitted with a SQL injection vulnerability. Where do the CodeQL findings appear?

**A:** Findings appear in two places:
1. As a **pull request check failure** (blocking merge if required)
2. In the repository's **Security → Code scanning alerts** tab

The workflow requires `permissions: security-events: write` to upload results to the Security tab.

---

**Q10.** Your organization's Azure Policy compliance report shows a storage account as "Non-compliant" for the "Require secure transfer" policy. The policy has an Audit effect. What happened to the storage account?

**A:** Nothing — it was **created successfully**. An **Audit** effect does NOT block resource creation. The storage account exists without secure transfer enabled, and it's logged as non-compliant in the compliance report. To prevent it from being created, the policy effect must be **Deny**.

---

**Q11.** You need to automatically deploy a Log Analytics agent to every new Azure VM in your subscription. Which Azure Policy effect achieves this?

**A:** **DeployIfNotExists**. This effect triggers a template deployment to install the agent if it's not already present. `AuditIfNotExists` would only report missing agents without taking action. `Deny` would block VM creation without the agent.

---

**Q12.** A penetration tester runs an OWASP ZAP scan against your application. What type of security testing is this?

**A:** **DAST (Dynamic Application Security Testing)**. OWASP ZAP sends requests to the running application from outside, simulating an attacker, without access to source code. SAST analyzes source code statically.

---

**Q13.** Your GitHub Actions workflow uses CodeQL but scanning results never appear in the Security tab. What is the most likely missing permission?

**A:** The workflow is missing `permissions: security-events: write`. This permission is required for the `github/codeql-action/analyze` step to upload results to GitHub's Security tab:
```yaml
permissions:
  security-events: write
  actions: read
  contents: read
```

---

**Q14.** Which Defender for Cloud feature shows a numeric score representing overall security posture and provides prioritized recommendations to improve it?

**A:** **Secure Score**. It's a 0–100 score based on completed security controls. Each security recommendation belongs to a control. Completing all recommendations in a control earns the control's full score contribution.

---

**Q15.** You need to detect lateral movement attacks and Pass-the-Hash attempts in your on-premises Active Directory environment. Which Microsoft security product should you deploy?

**A:** **Microsoft Defender for Identity** (formerly Azure ATP). It monitors on-premises Active Directory traffic via sensors deployed on domain controllers and detects identity-based attacks like Pass-the-Hash, Pass-the-Ticket, and Golden Ticket. This is different from Defender for Cloud (cloud posture) or Entra ID Protection (cloud identity).

---

**Q16.** Your team has an MIT-licensed library and an Apache 2.0-licensed library as dependencies. Can you include both in a commercial proprietary product?

**Q18.** Your Azure AD tenant is licensed for Azure AD Premium Plan 1. You need to deploy a privileged access management solution that enforces time limits on privileged access and requires approval to activate privileged access, while minimizing costs. What should you do first?

A. Configure alerts for the activation of privileged roles.
B. Enforce MFA for role activation.
C. Configure notifications when privileged roles are activated.
D. Upgrade the license of the Azure AD tenant.

**A: D — Upgrade the license to Azure AD Premium Plan 2.**
Azure AD Privileged Identity Management (PIM) — which provides JIT access, time limits, and approval workflows — requires **Azure AD Premium Plan 2**. The tenant currently has Plan 1, so PIM is not available at all. Options A, B, and C are all PIM configuration settings that can only be set up *after* PIM is enabled, which requires the P2 license. The first step is the license upgrade.

---

**A:** **Yes**. Both **MIT** and **Apache 2.0** are **permissive licenses** — they allow use in commercial proprietary products. The only requirements are: include the copyright notice and license text. Apache 2.0 also requires including a NOTICE file if one exists.

---

**Q17.** You have an Azure DevOps project with a build pipeline that uses approximately 50 open-source libraries. You need to ensure the project can be scanned for known security vulnerabilities in the open-source libraries. What should you do?

- Object to create: A build task / A deployment task / An artifacts repository
- Service to use: WhiteSource Bolt / Bamboo / CMake / Chef

**A: Object = A build task; Service = WhiteSource Bolt.**
WhiteSource Bolt (Mend) integrates into Azure DevOps as a **build task** that scans open-source dependencies against a CVE database during the build. A deployment task runs after build (wrong stage). An artifacts repository stores packages but does not scan them. Bamboo is a CI server, CMake is a build system, and Chef is a configuration management tool — none scan for vulnerabilities.

---

**Q19.** You use WhiteSource Bolt to scan a Node.js application. The scan identifies numerous libraries with invalid licenses. These libraries are used only during development and are not part of a production deployment. You need to ensure that WhiteSource Bolt only scans production dependencies. Which two actions should you perform?

A. Run `npm install` and specify the `--production` flag
B. Modify the WhiteSource Bolt policy and set the action for the licenses used by the development tools to Reassign
C. Modify the `devDependencies` section of the project's package.json file
D. Configure WhiteSource Bolt to scan the `node_modules` directory only

**A: A and C.**
- **C** — Ensure dev-only libraries are correctly listed under `devDependencies` in package.json (not under `dependencies`).
- **A** — Run `npm install --production` to install only `dependencies`, skipping `devDependencies`. This means `node_modules` only contains production packages, so WhiteSource Bolt only scans those.
- B is wrong: "Reassign" policy action re-categorizes violations but does not exclude libraries from being scanned.
- D is wrong: Scanning `node_modules` directly doesn't help if devDependencies are still installed there.

---

**Q20.** The lead developer reports that adding new application features takes longer than expected due to accumulated technical debt. You need to recommend changes to reduce it. Solution: You recommend reducing the code coupling and the dependency cycles. Does this meet the goal?

A. Yes  B. No

**A: A — Yes.** *(Note: some exam dumps record B — this is incorrect)*
High code coupling and dependency cycles are primary sources of technical debt — they cause cascading changes when adding features. Reducing coupling makes modules independently modifiable; breaking dependency cycles eliminates circular fragility. Both directly reduce the technical debt that slows feature development.

---

**Q21.** DRAG DROP — You need to increase the security of your team's development process. Which type of security tool should you recommend for each stage?

- Pull request: ?
- Continuous integration: ?
- Continuous delivery: ?

Tools: Penetration testing / Static code analysis / Threat modeling

**A:**
- **Pull request → Static code analysis** — SAST scans code changes before merge, catching vulnerabilities at review time
- **Continuous integration → Static code analysis** — automated SAST runs on every build in the CI pipeline
- **Continuous delivery → Penetration testing** — requires a running deployed application (staging environment) as a target

**Threat modeling is not used** — it is a design-phase activity performed before coding begins, not a CI/CD pipeline stage tool.

---

**Q22.** You are designing the development process for your company. You need to recommend a solution for continuous inspection of the code base to locate common code patterns that are known to be problematic. What should you include in the recommendation?

> A. Microsoft Visual Studio test plans  B. Gradle wrapper scripts  C. SonarCloud analysis  D. the JavaScript task runner

**A: C — SonarCloud analysis.**
SonarCloud performs **continuous static code analysis (SAST)**, detecting code smells, bugs, security hotspots, and common problematic patterns across many languages. It integrates directly into CI pipelines for continuous inspection. Visual Studio test plans = test management (not static analysis). Gradle = Java build tool. JavaScript task runners = build/task automation tools.

---

**Q23.** Your company is building a new solution in Java. The company currently uses a SonarQube server to analyze the code of .NET solutions. You need to analyze and monitor the code quality of the Java solution. Which task type should you add to the build pipeline?

A. Chef  B. Gradle  C. Octopus  D. Gulp

**A: B — Gradle.**
Gradle is the standard Java build tool and has native SonarQube integration via the `sonarqube` Gradle plugin. Adding the Gradle task to the Azure Pipelines build pipeline enables SonarQube code quality analysis for Java solutions, reusing the existing SonarQube server. Chef = infrastructure configuration management. Octopus = deployment tool (Octopus Deploy). Gulp = JavaScript task runner.

> **Exam tip:** Java + SonarQube in Azure Pipelines = **Gradle** or **Maven** task (both are valid Java build tools with native SonarQube plugin support). Choose whichever appears in the options. .NET + SonarQube = `SonarQubePrepare` / `SonarQubeAnalyze` tasks.

---

**Q24.** Your company is building a new solution in Java. The company currently uses a SonarQube server to analyze the code of .NET solutions. You need to analyze and monitor the code quality of the Java solution. Which task type should you add to the build pipeline?

A. Octopus  B. Chef  C. Maven  D. Grunt

**A: C — Maven.**
Maven is a standard Java build tool with native SonarQube integration via the `sonarqube-maven-plugin` (`mvn sonar:sonar`). This is a variant of Q23 — when Gradle is not an option, Maven is the correct Java build/analysis tool. Octopus = deployment tool. Chef = infrastructure configuration management. Grunt = JavaScript task runner.

---

**Q25. (MULTI-SELECT)** Your company uses Azure DevOps for the build pipelines and deployment pipelines of Java-based projects. You need to recommend a strategy for managing technical debt. Which two actions should you include?

A. Integrate Azure DevOps and SonarQube.  B. Integrate Azure DevOps and Azure DevTest Labs.  C. Configure post-deployment approvals in the deployment pipeline.  D. Configure pre-deployment approvals in the deployment pipeline.

**A: A and C.**
- **SonarQube (A)** — the primary tool for measuring, tracking, and remediating technical debt. Analyzes code quality, code smells, bugs, and vulnerabilities; produces a debt score visible in Azure DevOps.
- **Post-deployment approvals (C)** — after deploying to a stage (e.g., staging), a reviewer examines the SonarQube/quality report before approving promotion to the next environment. This creates a quality gate checkpoint backed by actual debt metrics.

DevTest Labs (B) provisions environments — unrelated to code quality or debt. Pre-deployment approvals (D) control *when* a deployment starts, not *what quality was found after it ran*.

> **Exam tip:** Technical debt management = SonarQube (measure debt) + post-deployment approvals (human review of metrics before stage promotion). Pre-deployment = control entry; post-deployment = review exit quality.

---

**Q26.** Your company is concerned that when developers introduce open source libraries, it creates licensing compliance issues. You need to add an automated process to the build pipeline to detect when common open source libraries are added to the code base. What should you use?

A. Code Style  B. Microsoft Visual SourceSafe  C. Black Duck  D. Jenkins

**A: C — Black Duck.**
Black Duck is an enterprise OSS governance tool that automatically scans for open source libraries introduced to the code base, checks them against license compliance policies, and flags violations during the build. Visual SourceSafe is a legacy version control system (unrelated). Code Style is a code formatting tool. Jenkins is a CI server — it does not scan for license compliance. Note: WhiteSource/Mend is also a valid answer for this category but is not an option here.

---

**Q27.** You have 50 Node.js-based projects that you scan by using WhiteSource. Each project includes Package.json, Package-lock.json, and Npm-shrinkwrap.json files. You need to minimize the number of libraries reported by WhiteSource to only the libraries that you explicitly reference. What should you do?

A. Configure the File System Agent plug-in  B. Delete Package-lock.json  C. Configure the Artifactory plug-in  D. Add a devDependencies section to Package-lock.json

**A: D — Add a devDependencies section to Package-lock.json.**
Separating dev-only libraries into a `devDependencies` section allows WhiteSource to distinguish production (explicitly referenced) libraries from development tooling. WhiteSource respects the devDependencies marker and can be configured to report only production dependencies. Note: The underlying concept applies to `package.json` (where devDependencies is defined), which then propagates to the lock files. Deleting Package-lock.json (B) would break deterministic installs. The File System Agent (A) and Artifactory plug-in (C) are for different integration scenarios unrelated to dependency filtering.

---

## 7. Quick Reference Checklist

### DevSecOps & Security Testing
- [ ] Shift-left = move security earlier in pipeline
- [ ] SAST = static analysis (code NOT running) — CodeQL, SonarQube
- [ ] DAST = dynamic analysis (application IS running) — OWASP ZAP, Burp Suite
- [ ] SCA = third-party dependency scanning — Dependabot, Snyk, Mend
- [ ] STRIDE: Spoofing, Tampering, Repudiation, Information Disclosure, DoS, Elevation of Privilege
- [ ] SQL injection fix = parameterized queries (not string concatenation)
- [ ] CodeQL needs `permissions: security-events: write` to upload findings

### Open-Source Licenses
- [ ] Permissive (safe for commercial): MIT, Apache 2.0, BSD
- [ ] Copyleft/viral (risky for commercial): GPL v2, GPL v3, AGPL
- [ ] Weak copyleft (usually OK to link): LGPL, MPL 2.0
- [ ] GPL = use in commercial product → must open-source entire product under GPL
- [ ] Apache 2.0 = includes patent grant (advantage over MIT)
- [ ] AGPL = closes SaaS loophole — using over network = must provide source

### Software Composition Analysis (SCA)
- [ ] CVE = identifier for known vulnerability; CVSS score = severity (0–10)
- [ ] Critical CVSS = 9.0–10.0; High = 7.0–8.9
- [ ] Dependabot: alerts (reports), security updates (auto-PR), version updates (keep current)
- [ ] `.github/dependabot.yml` required for version update scheduling
- [ ] Dependency Review action: `fail-on-severity`, `deny-licenses` parameters
- [ ] Container scanning: before pushing to registry (pipeline) + registry scan on push

### Azure Policy Effects (highest tested)
- [ ] **Deny** = blocks operation BEFORE resource is created
- [ ] **Audit** = allows creation, reports as non-compliant
- [ ] **AuditIfNotExists** = audit if a child/related resource is MISSING
- [ ] **DeployIfNotExists** = deploy a related resource if MISSING (auto-remediation)
- [ ] **Append/Modify** = add/change properties on resources
- [ ] Effect evaluation order: Disabled → Append/Modify → Deny → Audit/AuditIfNotExists/DeployIfNotExists

### Resource Locks
- [ ] CanNotDelete = prevents deletion; allows read + modify
- [ ] ReadOnly = prevents deletion AND modification; read only
- [ ] Locks are inherited (RG lock applies to all resources in RG)
- [ ] Only Owner / User Access Administrator can manage locks
- [ ] Locks override RBAC — Owner cannot delete a resource with Delete lock without removing lock first

### Defender for Cloud
- [ ] Secure Score = 0–100; based on security controls, not number of recommendations
- [ ] Recommendations → improve Secure Score
- [ ] Alerts → real-time threat detection
- [ ] Regulatory Compliance blade → PCI DSS, ISO 27001, SOC 2, HIPAA, CIS

### GitHub Advanced Security (GHAS)
- [ ] Code Scanning (CodeQL): SAST on code → Security → Code scanning alerts
- [ ] Secret Scanning: detect secrets in commits → Security → Secret scanning alerts
- [ ] Push Protection: BLOCKS pushes containing secrets BEFORE they're committed
- [ ] Dependency Review: shows vulnerable deps in PRs before merge
- [ ] GHAS available for both GitHub AND Azure DevOps

### Defender for Identity vs Defender for Cloud vs Entra ID Protection
- [ ] Defender for Identity = on-premises Active Directory threats (Pass-the-Hash, Golden Ticket)
- [ ] Defender for Cloud = Azure cloud resource security posture + recommendations
- [ ] Entra ID Protection = cloud identity risk (Azure AD sign-in risk, user risk)

### Pipeline Security
- [ ] Least privilege for service connections — scope per environment, not org-wide
- [ ] Never "Grant access to all pipelines" for sensitive service connections
- [ ] Store secrets in Azure Key Vault, not pipeline YAML
- [ ] Self-hosted agents: risk of PPE (Poison Pipeline Execution) — restrict branch triggers
- [ ] MicrosoftSecurityDevOps@1 task: runs Bandit, ESLint, Terrascan, Trivy, Credscan

---

*Sources: [MS Learn AZ-400 Security Path](https://learn.microsoft.com/en-us/training/paths/az-400-implement-security-validate-code-bases-compliance/) | [AZ-400 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-400) | [Azure Policy effect basics](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-basics) | [GHAS and Defender for DevOps](https://www.mindmeshacademy.com/certifications/azure/az-400-designing-and-implementing-microsoft-devops-solutions/study-guide/4-1-3-2-github-advanced-security-and-defender-for-devops)*
