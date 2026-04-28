# AZ-400 Study Notes: Design and Implement a Dependency Management Strategy

**Learning Path:** Design and Implement a Dependency Management Strategy
**Exam Weight:** Part of "Implement and manage infrastructure" / "Implement CI with Azure Pipelines and GitHub Actions" domains
**Modules Covered:** 5

---

## Table of Contents

1. [Explore Package Dependencies](#1-explore-package-dependencies)
2. [Understand Package Management](#2-understand-package-management)
3. [Migrate, Consolidate, and Secure Artifacts](#3-migrate-consolidate-and-secure-artifacts)
4. [Implement a Versioning Strategy](#4-implement-a-versioning-strategy)
5. [Introduction to GitHub Packages](#5-introduction-to-github-packages)
6. [Additional Exam Topics](#6-additional-exam-topics)
7. [Past Exam Scenario Bank](#7-past-exam-scenario-bank)
8. [Quick Reference Checklist](#8-quick-reference-checklist)

---

## 1. Explore Package Dependencies

### What is Dependency Management?

**Dependency management** is the practice of identifying, tracking, and managing external libraries, components, and services that your software depends on.

**Goals:**
- Reproducible builds (same versions every time)
- Avoid "works on my machine" problems
- Prevent supply chain security issues
- Enable consistent CI/CD builds

### Types of Dependencies

| Type | Description | Example |
|---|---|---|
| **Package dependencies** | Third-party libraries consumed via a package manager | NuGet, npm, Maven packages |
| **Source dependencies** | Code from another team/repo included as source | Shared utility library |
| **Binary dependencies** | Pre-compiled binaries consumed directly | DLLs, JAR files |
| **Service dependencies** | External APIs or services your app calls | Azure Cognitive Services, Stripe API |

### Package Componentization

**Componentization** means decomposing a system into smaller, independently versioned, and independently deployable units (packages/components).

**Benefits:**
- Reuse components across multiple projects
- Version and release components independently
- Teams can work in parallel without blocking each other
- Reduces duplication across the codebase

### Source vs Package Componentization

| Approach | Description | When to Use |
|---|---|---|
| **Source componentization** | Code included directly (e.g., git submodules, shared folder) | Internal code, rapid iteration |
| **Package componentization** | Code published as a versioned package and consumed via package manager | Stable shared libraries, cross-team sharing |

> **Exam tip:** Package componentization is preferred for stable, shared code. Source componentization suits code that changes frequently and is tightly coupled to the consuming project.

### Identifying Dependencies

**Scanning your codebase for dependencies:**

| Tool / File | Package Type | What it Lists |
|---|---|---|
| `packages.config` / `*.csproj` | NuGet (.NET) | All referenced NuGet packages |
| `package.json` / `package-lock.json` | npm (Node.js) | npm package dependencies and lock file |
| `pom.xml` / `build.gradle` | Maven / Gradle (Java) | Java library dependencies |
| `requirements.txt` / `Pipfile` | pip (Python) | Python package dependencies |
| `go.mod` | Go modules | Go dependencies |

**Automated dependency scanning tools:**
- **GitHub Dependabot** — automatically opens PRs to update vulnerable dependencies
- **OWASP Dependency-Check** — scans for known CVEs in dependencies
- **WhiteSource/Mend** — license compliance + vulnerability scanning
- **Snyk** — security-focused dependency scanning

### Decomposing a System

Steps for decomposing a monolith into components:

1. **Identify cohesive groups** of functionality
2. **Define clear interfaces** (APIs) between components
3. **Eliminate circular dependencies** (A → B → A)
4. **Version each component** independently using SemVer
5. **Publish components** to a package feed

### Dependency Management Strategy Elements

A complete strategy includes:
- **Standardize** package formats per technology (NuGet for .NET, npm for Node.js)
- **Private feed** for internal packages (Azure Artifacts)
- **Upstream sources** to proxy and cache public registries
- **Versioning policy** (SemVer, automatic version bumping in CI)
- **Security scanning** of dependencies in the pipeline
- **Retention policies** to clean up old package versions

---

## 2. Understand Package Management

### What is a Package?

A **package** is a versioned, self-contained bundle of code and metadata (name, version, description, dependencies) that can be published and consumed via a package manager.

### Package Types Supported by Azure Artifacts

| Format | Technology | Public Registry |
|---|---|---|
| **NuGet** | .NET / C# | nuget.org |
| **npm** | Node.js / JavaScript | npmjs.com |
| **Maven** | Java | maven.org / mvnrepository.com |
| **Python (pip)** | Python | pypi.org |
| **Universal Packages** | Any binary/file | Azure Artifacts only |

> **Exam tip:** Universal Packages are unique to Azure Artifacts — they allow publishing any type of file (binaries, scripts, tools) as a versioned artifact. There is no public registry equivalent.

### Package Feeds

A **package feed** (also called a **package registry** or **package repository**) is the server where packages are published and from which they are consumed.

**Feed types:**

| Type | Examples | Description |
|---|---|---|
| **Public SaaS** | nuget.org, npmjs.com, pypi.org | Open, publicly accessible, free |
| **Private SaaS** | Azure Artifacts, GitHub Packages | Hosted service, access-controlled |
| **Self-hosted** | ProGet, Nexus, Artifactory | On-premises or custom-hosted |

### Package Feed Managers (Clients)

| Manager | Package Format | Commands |
|---|---|---|
| `nuget` / `dotnet` | NuGet | `dotnet add package`, `nuget push` |
| `npm` | npm | `npm install`, `npm publish` |
| `mvn` | Maven | `mvn install`, `mvn deploy` |
| `pip` | Python | `pip install`, `twine upload` |

### Azure Artifacts

**Azure Artifacts** is Azure DevOps's private package feed service.

**Key features:**
- Host NuGet, npm, Maven, Python, and Universal packages
- Upstream sources (proxy public registries)
- Feed-level and package-level permissions
- Integration with Azure Pipelines
- Package retention policies
- Views for package promotion (@Local, @Prerelease, @Release)

### Publishing a NuGet Package Privately — Sequence (Exam Tested)

To create a NuGet package, distribute it privately to a team, and verify it can be consumed:

| Step | Action | Purpose |
|---|---|---|
| 1 | **Create a new Azure Artifacts feed** | Set up the private feed to host the package |
| 2 | **Connect to an Azure Artifacts feed** | Configure the NuGet client to point to the feed as a package source |
| 3 | **Publish a package** | Push the NuGet package to the feed |
| 4 | **Install a package** | Have team members install from the feed to verify consumption works |

> **Exam tip:** "Configure a self-hosted agent" is a common distractor in this scenario — it is not needed to privately distribute a NuGet package. You must **connect** before you can **publish** (NuGet needs to know the feed URL), and you must **publish** before anyone can **install**.

### Consuming Packages from Azure Artifacts

**NuGet (dotnet restore):**
```xml
<!-- NuGet.Config -->
<configuration>
  <packageSources>
    <add key="MyFeed" value="https://pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

**npm — two .npmrc files pattern (exam favorite):**

Azure DevOps recommends using **two separate `.npmrc` files** to minimize credential leakage:

| File | Location | Contains | Committed to git? |
|---|---|---|---|
| **Project `.npmrc`** | Root of git repo (next to `package.json`) | Registry URL only (`registry=...`) | **Yes** — safe, no credentials |
| **Home folder `.npmrc`** | `$home/.npmrc` (Linux/Mac) or `$env.HOME/.npmrc` (Windows) | Credentials for all registries | **No** — stays on developer's machine |

> **Exam tip (DRAG DROP):** "Registry information" → `.npmrc file in the project`; "Credentials" → `.npmrc file in the user's home folder`. The npm client reads the project `.npmrc` to discover the registry, then fetches matching credentials from the home `.npmrc`. Credentials never enter source control.

```bash
# Project .npmrc (committed to git — registry only, NO credentials)
registry=https://pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}/npm/registry/
always-auth=true
```

**Maven:**
```xml
<!-- pom.xml repository section -->
<repository>
  <id>azure-artifacts</id>
  <url>https://pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}/maven/v1</url>
</repository>
```

### Publishing Packages to Azure Artifacts

**In Azure Pipelines (NuGet example):**
```yaml
- task: NuGetAuthenticate@1
- task: NuGetCommand@2
  inputs:
    command: 'push'
    packagesToPush: '$(Build.ArtifactStagingDirectory)/**/*.nupkg'
    nuGetFeedType: 'internal'
    publishVstsFeed: 'MyFeed'
```

**npm publish:**
```yaml
- task: npmAuthenticate@0
  inputs:
    workingFile: .npmrc
- script: npm publish
```

### Upstream Sources

**Upstream sources** allow a feed to act as a **proxy** for public registries. When a package is requested that doesn't exist in the feed, Azure Artifacts fetches it from the upstream source and **caches** it.

**Benefits:**
- Single source of truth — developers point to one feed
- Resilience — if public registry goes down, cached packages still available
- Security — you control which packages are accessible
- Compliance — all consumed packages flow through your controlled feed

**Supported upstream sources — package type mapping (exam favorite):**

| Package type | Language | Upstream source to select |
|---|---|---|
| NuGet | .NET / C# | **nuget.org** |
| npm | JavaScript / Node.js | **npmjs.com** |
| Python (pip) | Python | **PyPI** (pypi.org) |
| Maven | Java | **Maven Central** |
| Universal | Any binary | Another Azure Artifacts feed |

> **Exam tip:** Python packages → **PyPI** upstream source. npmjs.org = JavaScript, Maven Central = Java. "Third-party trusted Python" is not a real upstream source type in Azure Artifacts.

> **Exam tip:** When upstream sources are enabled, you **cannot** publish a package with the same name and version as one that exists in the upstream source. For example, if nuget.org is an upstream, you cannot push `Newtonsoft.Json 13.0.1` to your feed.

**Search order:** When a package is requested from a feed with upstream sources:
1. Packages published **directly** to the feed (local)
2. Packages cached **from upstream sources**
3. Fetched from the **upstream source** live (then cached)

### Self-Hosted Package Solutions

| Solution | Type | Key Feature |
|---|---|---|
| **JFrog Artifactory** | Self-hosted / SaaS | Universal binary repository, multi-format |
| **Sonatype Nexus** | Self-hosted | Popular for Java/Maven, proxy + host + group repos |
| **ProGet** | Self-hosted | Windows-focused, NuGet/npm/Docker |
| **MyGet** | SaaS | NuGet/npm/Bower, CI integration |

---

## 3. Migrate, Consolidate, and Secure Artifacts

### Identifying Existing Artifact Repositories

Before migration, audit:
- What package formats are in use (NuGet, npm, Maven, Python)?
- What tools host them currently (Nexus, Artifactory, file shares)?
- Who consumes them (teams, pipelines)?
- What retention/versioning policies exist?

### Migration Strategies

**Approach 1: Fresh start** — Publish only new package versions to Azure Artifacts; keep old versions in legacy store with a time-limit.

**Approach 2: Full migration** — Move all packages to Azure Artifacts. Use automation scripts to download from old feed and push to Azure Artifacts.

**Approach 3: Upstream chaining** — Add the old feed as an upstream source in Azure Artifacts. Packages flow through Azure Artifacts transparently while the old feed is gradually decommissioned.

> **Exam tip:** Upstream chaining (adding the legacy feed as an upstream source) is the **least disruptive** migration approach — developers don't need to change their configuration immediately.

### Securing Package Feeds — Azure Artifacts Roles

Azure Artifacts feeds use **role-based access control**:

| Role | Permissions |
|---|---|
| **Reader** | Download and install packages from the feed |
| **Collaborator** | Reader permissions + save packages from upstream sources |
| **Contributor** | Collaborator permissions + publish new packages to the feed |
| **Owner** | Full control — manage feed settings, permissions, and delete packages |

> **Exam trap — know the distinctions:**
> - **Collaborator** = can save/cache packages from upstream sources, but cannot publish new packages directly
> - **Contributor** = can publish new packages to the feed
> - Need to just download packages → **Reader**
> - Need to publish packages in CI/CD → **Contributor**
> - Need to manage the feed itself → **Owner**

### Feed Visibility Settings

| Visibility | Accessible By |
|---|---|
| **Private** | Only users/groups explicitly granted access |
| **Organization** | All users in the Azure DevOps organization |
| **Project** | All members of the Azure DevOps project |
| **Public** | Anyone (no authentication required) |

> **Exam tip — anonymous/public access with minimum publication points:** To make a specific package available to anonymous users **outside your organization** while minimizing publication points, create a **new feed** set to Public visibility. Publishing to an external registry like nuget.org creates a second publication point. Changing a view or URL does not grant anonymous access. One dedicated public feed = one publication point.

### Granular Permissions

Beyond roles, you can set permissions at multiple levels:
- **Feed level** — applies to all packages in the feed
- **Package level** — applies to a specific package within the feed (more granular)

**Service principals and managed identities** can also be granted feed roles for pipeline automation.

### Authentication Mechanisms

**Personal Access Tokens (PATs):**
- Most common authentication method for Azure Artifacts
- Created in Azure DevOps under User Settings → Personal Access Tokens
- Required scope: `Packaging (read)` or `Packaging (read, write, manage)`
- Use as password in basic auth or NuGet credential provider

**Azure Artifacts Credential Provider:**
- Installed tool that automates NuGet authentication in CI/CD
- Acquires tokens from the pipeline environment automatically
- Prevents interactive prompts during `dotnet restore`

**NuGet API Key:**
- Not used for Azure Artifacts (unlike nuget.org)
- Azure Artifacts uses PAT or service principal authentication

**Pipeline service connection:**
- `NuGetAuthenticate@1` task — configures NuGet authentication from pipeline
- `npmAuthenticate@0` task — configures npm authentication from pipeline
- These tasks use the pipeline's service connection/identity automatically

> **Exam tip:** In Azure Pipelines, use `NuGetAuthenticate@1` before any NuGet restore/push tasks to configure authentication to Azure Artifacts feeds without manually managing PATs in the pipeline.

### Upstream Source Security

- Packages from upstream sources are **immutable once cached** — the same version will always serve the same content
- You can **block** specific packages or versions from upstream sources
- **Include prerelease versions** toggle controls whether prerelease packages are fetched from upstream

---

## 4. Implement a Versioning Strategy

### Why Versioning Matters

- Consumers know exactly what behavior to expect from a version
- Enables reproducible builds (pin to specific version)
- Communicates breaking changes vs. safe upgrades
- Enables rollback if a newer version causes issues

### Semantic Versioning (SemVer 2.0)

Format: **`MAJOR.MINOR.PATCH[-prerelease][+buildmetadata]`**

Examples: `1.0.0`, `2.3.1`, `3.0.0-beta.1`, `1.2.3+build.456`

| Segment | When to Increment | Example |
|---|---|---|
| **MAJOR** | Breaking changes (incompatible API changes) | `1.5.2` → `2.0.0` |
| **MINOR** | New features, backwards-compatible | `1.5.2` → `1.6.0` |
| **PATCH** | Bug fixes, backwards-compatible | `1.5.2` → `1.5.3` |

**Rules:**
- When MAJOR is incremented → reset MINOR and PATCH to 0
- When MINOR is incremented → reset PATCH to 0
- Version `0.x.x` = initial development; no stability guarantees
- Version `1.0.0` = first stable public API

> **Exam trap:** Deprecating an API feature but keeping it functional (backwards-compatible) = **MINOR** version bump, not MAJOR. MAJOR is only for **breaking** (incompatible) changes.

### Prerelease Versions

Prerelease identifiers communicate quality levels:

| Prerelease Tag | Meaning |
|---|---|
| `alpha` | Very early, unstable (e.g., `1.0.0-alpha.1`) |
| `beta` | Feature complete but may have bugs (e.g., `1.0.0-beta.2`) |
| `rc` | Release candidate — near final (e.g., `1.0.0-rc.1`) |

**Version ordering:** `1.0.0-alpha < 1.0.0-beta < 1.0.0-rc.1 < 1.0.0`

### Azure Artifacts Views

Azure Artifacts uses **views** to indicate the quality level of a package version:

| View | Meaning | Who Should Use |
|---|---|---|
| **@Local** | Packages published directly to the feed (all versions) | Default; all published packages land here |
| **@Prerelease** | Packages that have passed initial validation but are not production-ready | QA, staging consumers |
| **@Release** | Packages that are production-ready | Production consumers |

**Key behaviors:**
- A package must be in `@Local` before it can be promoted to `@Prerelease` or `@Release`
- Promotion is a **one-way** operation (cannot un-promote, only deprecate/unlist)
- Consumers can pin their feed to a specific view, ensuring they only receive packages of the desired quality

> **Exam tip:** When a consumer configures their feed connection to `@Release`, they will **only receive packages that have been explicitly promoted to Release**. New versions published to `@Local` are invisible to them until promoted.

> **Exam tip:** "Single feed + limit release of packages in development" = **Views**. Upstream sources are for *consuming* packages from external registries (nuget.org, npmjs.com) — not for controlling visibility of your own packages within a feed.

### Package Promotion Workflow

```
Build Pipeline
     |
     | publishes to @Local
     v
  @Local (all versions)
     |
     | QA validates → promote
     v
  @Prerelease (validated, not production-ready)
     |
     | Final sign-off → promote
     v
  @Release (production-ready)
```

**Promote a package via Azure CLI:**
```bash
az artifacts universal publish \
  --organization https://dev.azure.com/{org} \
  --feed myFeed \
  --name myPackage \
  --version 1.0.0 \
  --path ./dist

# Promote via REST API or Azure DevOps portal UI
```

**Promote via Azure Pipelines:**
```yaml
- task: UniversalPackages@0
  inputs:
    command: 'promote'
    feedsToUse: 'internal'
    vstsFeed: 'myFeed'
    packageName: 'myPackage'
    packageVersion: '$(Build.BuildNumber)'
    viewId: 'Release'
```

### Automated Versioning in CI/CD

**Version numbering strategies:**

| Strategy | Format | Example |
|---|---|---|
| **Build number** | `Major.Minor.BuildNumber` | `1.0.$(Build.BuildId)` |
| **GitVersion** | Semantic, computed from Git history | `2.1.0+5` |
| **Date-based** | `Year.Month.Day.Revision` | `2026.03.24.1` |
| **Manual SemVer** | Developer-set in source | `2.1.0` from `version.props` |

**GitVersion tool:**
- Analyzes Git history (commits, tags, branches) to auto-calculate SemVer
- Commonly used in Azure Pipelines via `gitversion/execute@0` action

### Package Retention Policies

- **Why:** Package feeds accumulate old versions; storage costs money; stale packages create confusion
- Configure in Azure Artifacts feed settings: **Retention policies**
- Options: keep latest N versions, delete versions older than N days
- Applies to `@Local` view by default; packages promoted to `@Release` may be excluded from deletion

### Versioning Best Practices

| Practice | Recommendation |
|---|---|
| **Never modify a published version** | Once published, a version is immutable. Publish a new version instead. |
| **Use prerelease for CI builds** | Append `-ci.BuildNumber` to avoid polluting stable version space |
| **Lock dependencies in production** | Use exact version pins (`1.2.3`), not ranges (`^1.2.0`) for reproducibility |
| **Automate version bumping** | Use GitVersion or pipeline variables to eliminate manual version errors |
| **Communicate breaking changes** | Increment MAJOR version, update changelog, provide migration guide |

---

## 5. Introduction to GitHub Packages

### What is GitHub Packages?

**GitHub Packages** is a package hosting service integrated directly into GitHub. Packages are stored alongside the source code repository.

**Supported package registries:**

| Registry | Package Format |
|---|---|
| **npm registry** | Node.js / JavaScript packages |
| **NuGet registry** | .NET packages |
| **Maven registry** | Java packages |
| **Docker registry** (GHCR) | Container images |
| **RubyGems registry** | Ruby gems |
| **Gradle registry** | Java/Kotlin (via Maven endpoint) |

> **Exam tip:** GitHub Container Registry (GHCR) is the Docker/container-specific registry under GitHub Packages. It supports both Docker and OCI images.

### Publishing Packages to GitHub Packages

**Authentication:** Use a **Personal Access Token (PAT)** with `write:packages` scope, or **GITHUB_TOKEN** in GitHub Actions.

**npm example in GitHub Actions:**
```yaml
- name: Publish to GitHub Packages
  run: npm publish
  env:
    NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**NuGet example:**
```yaml
- name: Publish NuGet package
  run: dotnet nuget push "*.nupkg" \
       --api-key ${{ secrets.GITHUB_TOKEN }} \
       --source "https://nuget.pkg.github.com/{owner}/index.json"
```

### GITHUB_TOKEN vs PAT for GitHub Packages

| Token | Scope | Use Case |
|---|---|---|
| **GITHUB_TOKEN** | Current repository only | Publishing from GitHub Actions within the same repo |
| **PAT (Personal Access Token)** | Cross-repository, configurable | Publishing from external CI or another repo |

> **Exam tip:** `GITHUB_TOKEN` is automatically generated for each GitHub Actions workflow run. It has permissions scoped to the **current repository**. For cross-repository package access, use a PAT with `read:packages` or `write:packages` scope.

### Installing (Consuming) Packages from GitHub Packages

Packages must have at least **Read** access. For **private** packages, authentication is always required.

**npm (.npmrc):**
```
@OWNER:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=TOKEN
```

**NuGet:**
```bash
dotnet nuget add source \
  --username USERNAME \
  --password TOKEN \
  --store-password-in-clear-text \
  --name github \
  "https://nuget.pkg.github.com/OWNER/index.json"
```

### Deleting and Restoring Packages

**Delete a package:**
- Via GitHub UI: Package settings → Delete
- Via GitHub API: `DELETE /orgs/{org}/packages/{package_type}/{package_name}`
- Requires: `delete:packages` PAT scope or org admin

**Restore a deleted package:**
- Deleted packages can be **restored within 30 days**
- After 30 days, permanently gone
- Via GitHub UI or API

> **Exam tip:** Packages can be restored within **30 days** of deletion. After 30 days, they are permanently deleted and cannot be recovered.

### Package Visibility and Access Control

| Visibility | Description |
|---|---|
| **Private** | Only accessible to authenticated users with explicit access |
| **Public** | Accessible to anyone (no authentication required to read) |
| **Internal** | Accessible to all members of the GitHub organization (Enterprise feature) |

**Access inheritance from repository:**
- By default, a package inherits the visibility of its linked repository
- Private repo → private package
- Public repo → public package (can be changed)

**Granular access:**
- Grant specific users or teams access to a package independently of repository access
- Organizations can enforce policies: prevent public packages, require 2FA for publishers

### Comparing GitHub Packages vs Azure Artifacts

| Feature | Azure Artifacts | GitHub Packages |
|---|---|---|
| Package formats | NuGet, npm, Maven, Python, Universal | NuGet, npm, Maven, Docker, RubyGems |
| Views/promotion | @Local, @Prerelease, @Release | No built-in views |
| Upstream sources | Yes (proxy public registries) | No native upstream proxy |
| Retention policies | Yes | Manual deletion only |
| Integration | Azure DevOps / Azure Pipelines | GitHub / GitHub Actions |
| Pricing | 2 GB free, then per GB | 500 MB free (private), unlimited (public) |

> **Exam tip:** Azure Artifacts supports **upstream sources** and **views** for package promotion — GitHub Packages does not. For sophisticated promotion workflows, Azure Artifacts is the right choice.

---

## 6. Additional Exam Topics

### Dependency Vulnerability Management

**Supply chain attacks** target dependencies to inject malicious code. Key defenses:

| Defense | Tool/Approach |
|---|---|
| **Known vulnerability scanning** | Dependabot, Snyk, OWASP Dependency-Check |
| **License compliance** | WhiteSource/Mend, FOSSA |
| **Package integrity** | NuGet package signing, npm package-lock.json |
| **Private feed + upstream** | Control which packages can enter your environment |
| **Upstream source locking** | Upstream saves packages in Azure Artifacts, preventing version changes |

### Continuous Integration for License and Prohibited Library Detection

**CI pipelines can detect both licensing violations and prohibited libraries** by integrating SCA tools as build steps:

- **Licensing violations** → SCA tools (WhiteSource/Mend, FOSSA, Black Duck) scan each dependency's license against a permitted/denied list and fail the build if a violation is found
- **Prohibited libraries** → tools can maintain a blocklist of disallowed packages and alert/fail when they appear in the dependency tree

**Why CI is the right answer:**
- CI runs automatically on every code check-in
- SCA tools run as build tasks → violations are caught immediately, before the code reaches production
- "Shift-left" security — detect issues at the earliest possible stage

> **Exam tip (tested — series question):** This scenario is tested with multiple proposed solutions:
> - **Implement CI** → **Yes** — CI with SCA tools in the build pipeline detects licensing violations and prohibited libraries on every check-in.
> - **Implement automated security testing** → **No** — SAST/DAST detects security vulnerabilities (injection, XSS, CVEs), NOT license compliance or prohibited package usage.
> - **Implement a code review process** → **No** — manual reviews don't automatically detect every prohibited library or license issue at scale.
> The only correct solution is **SCA tools integrated into the CI pipeline**.

### Package Lock Files

Lock files ensure **reproducible installs** by pinning exact versions of all transitive dependencies:

| Lock File | Package Manager |
|---|---|
| `package-lock.json` | npm |
| `yarn.lock` | Yarn |
| `packages.lock.json` | NuGet |
| `Pipfile.lock` | pipenv (Python) |
| `go.sum` | Go modules |

> **Exam tip:** Committing lock files to source control is a best practice for applications (not libraries). Lock files guarantee the same dependency tree across all environments.

### NuGet vs npm Specifics for Azure Artifacts

**NuGet feed URL format:**
```
https://pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}/nuget/v3/index.json
```

**npm feed URL format:**
```
https://pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}/npm/registry/
```

**Project-scoped vs Organization-scoped feeds:**
- **Project-scoped feed:** Only accessible within the project; uses project-level permissions
- **Organization-scoped feed:** Accessible across the entire Azure DevOps organization

> **Exam tip:** Organization-scoped feeds are visible to all projects in the org by default. Project-scoped feeds are isolated to that project, providing better separation for multi-team orgs.

### Azure Pipeline Integration — Key Tasks

| Task | Purpose |
|---|---|
| `NuGetAuthenticate@1` | Authenticates NuGet to Azure Artifacts (replaces credentials) |
| `NuGetCommand@2` | Restore, pack, push NuGet packages |
| `npmAuthenticate@0` | Authenticates npm to Azure Artifacts |
| `Npm@1` | install, publish npm packages |
| `Maven@4` | Build and deploy Maven projects |
| `UniversalPackages@0` | Publish/download Universal Packages |
| `PublishBuildArtifacts@1` | Publish pipeline artifacts (not the same as Azure Artifacts packages) |
| `DownloadBuildArtifacts@1` | Download pipeline artifacts |

> **Exam tip:** `PublishBuildArtifacts` / `PublishPipelineArtifact` publishes **pipeline artifacts** (drop/staging area), not Azure Artifacts feed packages. These are different concepts. Pipeline artifacts are temporary (per-pipeline-run); Azure Artifacts feed packages are versioned and persistent.

### Pipeline Artifacts vs Azure Artifacts

| Concept | Pipeline Artifacts | Azure Artifacts (Feed) |
|---|---|---|
| Purpose | Pass files between pipeline stages/jobs | Store versioned packages for consumption |
| Lifetime | Duration of pipeline run (then configurable) | Persistent, versioned |
| Access | Within the pipeline | Any project/team/pipeline |
| Format | Any file | NuGet, npm, Maven, Python, Universal |
| Versioning | No built-in versioning | Full SemVer versioning |

### Retention Policy Considerations

Azure Artifacts feed retention:
- Can set: keep latest **N** versions per package name
- Can exclude: packages promoted to `@Release` view from deletion
- Runs automatically based on the policy configuration

Pipeline artifact retention:
- Set at pipeline run level or organization level
- Default: retain for 30 days (configurable)

### NuGet Package Creation

```bash
# Create NuGet package from a .NET project
dotnet pack --configuration Release --output ./nupkg

# Push to Azure Artifacts
dotnet nuget push ./nupkg/*.nupkg \
  --source "https://pkgs.dev.azure.com/{org}/_packaging/{feed}/nuget/v3/index.json" \
  --api-key az
```

The `--api-key az` value is a placeholder — Azure Artifacts uses credential provider authentication, not a real API key.

### Feed Scoping in Azure DevOps

**Project-scoped feed:**
- Default for new feeds created in a project
- Permissions inherited from project members
- Better isolation, recommended for team-specific packages

**Organization-scoped feed:**
- Visible to all projects in the organization
- Managed at org level
- Better for shared/common libraries used across all projects

---

## 7. Past Exam Scenario Bank

**Q1.** Your team consumes NuGet packages from both your private Azure Artifacts feed and nuget.org. A developer adds an upstream source pointing to nuget.org. Later, the developer tries to publish `Newtonsoft.Json 13.0.1` to the private feed. What happens?

**A:** The publish **fails**. When an upstream source is configured, you cannot publish a package with the same name and version that already exists in that upstream source. `Newtonsoft.Json 13.0.1` exists on nuget.org, so it cannot be pushed to the feed directly.

---

**Q2.** A CI/CD service principal needs to publish NuGet packages to an Azure Artifacts feed but should NOT be able to manage feed settings or delete packages. Which role should you assign?

**A:** **Contributor**. Contributor can publish (push) packages to the feed but cannot manage feed settings or delete packages. Owner is too permissive; Collaborator cannot publish new packages.

---

**Q3.** Your pipeline produces a new package version on every build. Downstream teams consuming the package should only receive versions that have passed full QA validation. How should you configure this?

**A:** Use **Azure Artifacts views**. Publish all CI builds to `@Local`. After QA validation passes, promote the package to `@Prerelease`, and after final sign-off, promote to `@Release`. Configure downstream teams to consume from the `@Release` view — they only see promoted, production-ready versions.

---

**Q4.** A library version `2.5.0` deprecates a method but keeps it functional (backward-compatible). Which version number should the next release be?

**A:** **`2.6.0`** (MINOR bump). Deprecating a method while keeping it functional is a **new feature** (the deprecation notice) that is backwards-compatible. MAJOR is only incremented for breaking (incompatible) changes. PATCH is only for bug fixes.

---

**Q5.** You need to allow a developer to install packages from a private Azure Artifacts feed but not publish new packages. Which role should you assign?

**A:** **Reader**. Readers can only download and install packages. They cannot publish, manage, or save from upstream sources.

---

**Q6.** Your team is migrating from an on-premises Nexus repository to Azure Artifacts with minimal disruption to developers. Which migration strategy causes the least disruption?

**A:** **Add the Nexus feed as an upstream source** in Azure Artifacts. Developers point to Azure Artifacts, and packages not yet in Azure Artifacts are transparently fetched and cached from Nexus. Teams migrate incrementally without changing their feed configuration.

---

**Q7.** A GitHub Actions workflow needs to publish a package to GitHub Packages. The workflow only needs access to the current repository's packages. Which authentication token should you use?

**A:** **`GITHUB_TOKEN`**. It is automatically available in GitHub Actions and has the permissions needed to publish packages from the current repository. A PAT is only needed when accessing packages from **other** repositories.

---

**Q8.** A developer accidentally published a package containing sensitive data. They delete the package from GitHub Packages. How long do they have to prevent anyone from restoring it?

**A:** **30 days**. Deleted packages on GitHub Packages can be restored within 30 days. After 30 days, they are permanently deleted. The sensitive data must also be rotated/invalidated immediately regardless of deletion.

---

**Q9.** Your Node.js application's CI build fails intermittently because a transitive dependency was updated on the public npm registry between builds. How do you fix this?

**A:** Commit the **`package-lock.json`** to source control and use `npm ci` (instead of `npm install`) in the pipeline. `npm ci` installs exact versions from the lock file, ensuring reproducible builds regardless of registry changes.

---

**Q10.** You want to share a compiled C++ binary tool (not a NuGet/npm/Maven package) across multiple Azure DevOps projects. What Azure Artifacts package type should you use?

**A:** **Universal Packages**. Universal Packages allow publishing any binary file as a versioned artifact to Azure Artifacts. They do not require a specific package format.

---

**Q11.** A package version is currently in the `@Prerelease` view of your Azure Artifacts feed. QA has completed validation. What action promotes it to `@Release`?

**A:** **Promote the package** to the `@Release` view via the Azure Artifacts UI, Azure CLI, or a pipeline task (`UniversalPackages@0` with `command: promote`). Promotion moves the package to the target view; it remains in all lower views as well.

---

**Q12.** Your pipeline uses `NuGetCommand@2` with the push command but fails with authentication errors when publishing to Azure Artifacts. What task should you add before the push task?

**A:** Add `NuGetAuthenticate@1` before the `NuGetCommand@2` task. This task configures the NuGet credential provider to authenticate to Azure Artifacts using the pipeline's identity, eliminating the need to manually manage PATs.

---

**Q13.** In a project with multiple teams, Team A needs access to their own packages and shared org-wide packages, while Team B should not see Team A's packages. How should you structure feeds?

**A:** Use a **project-scoped feed** for Team A's private packages (visible only to Team A's project) and a separate **organization-scoped feed** for shared packages (visible to all teams). Team B cannot see the project-scoped feed of Team A.

---

**Q14.** Your team releases `v3.0.0` of a library with breaking API changes. Six months later they add a new backwards-compatible helper method. What is the next version number?

**A:** **`3.1.0`**. Adding a new backwards-compatible feature = MINOR bump. `3.0.0` → `3.1.0`. MAJOR stays at 3 because there are no new breaking changes. PATCH would only be `3.0.1` for a bug fix.

---

**Q15.** You need to ensure that all packages consumed by your pipelines originate from your Azure Artifacts feed only (never directly from public registries). How do you enforce this?

**A:** Configure **upstream sources** in Azure Artifacts to proxy the public registries (nuget.org, npmjs.com). Then configure all pipelines and developer machines to point **only** to the Azure Artifacts feed URL. Packages from public registries are fetched via the feed and cached. Block direct access to public registries at the network level if needed.

---

**Q16.** You plan to update the Azure DevOps strategy. You need to identify licensing violations and prohibited libraries as they occur during the development process. Solution: you implement continuous integration. Does this meet the goal?

A. Yes  B. No

**A: A — Yes.**
CI pipelines can include SCA (Software Composition Analysis) tools (WhiteSource/Mend, FOSSA, Snyk) as build tasks. These tools automatically scan dependencies on every check-in, identifying licensing violations and prohibited libraries before the code reaches production. Implementing CI is the correct approach for detecting these issues during development.

---

**Q18.** You plan to update the Azure DevOps strategy of your company. You need to identify licensing violations and prohibited libraries as they occur during development. Solution: You implement automated security testing. Does this meet the goal?

A. Yes  B. No

**A: B — No.**
Automated security testing (SAST/DAST tools like CodeQL, SonarQube, OWASP ZAP) detects **security vulnerabilities** — not licensing violations or prohibited library usage. Those require **SCA (Software Composition Analysis)** tools integrated into a CI pipeline. Automated security testing is a different tool category entirely.

---

**Q17.** You are creating a NuGet package and plan to distribute it to your development team privately. You need to share the package and test that it can be consumed. Which four actions should you perform in sequence?

Actions: Create a new Azure Artifacts feed / Configure a self-hosted agent / Publish a package / Install a package / Connect to an Azure Artifacts feed

**A (in order):**
1. Create a new Azure Artifacts feed
2. Connect to an Azure Artifacts feed
3. Publish a package
4. Install a package

"Configure a self-hosted agent" is not needed — it relates to pipeline job execution, not NuGet package distribution. You must connect to the feed before publishing (NuGet client needs the feed URL), and publish before anyone can install.

---

**Q19.** Your company uses Azure Artifacts for package management. You need to configure an upstream source in Azure Artifacts for Python packages. Which repository type should you use as an upstream source?

A. PyPI  B. npmjs.org  C. Maven Central  D. third-party trusted Python

**A: A — PyPI.**
PyPI (Python Package Index, pypi.org) is the official public registry for Python packages and the correct upstream source type in Azure Artifacts for Python/pip. npmjs.org = JavaScript/Node.js; Maven Central = Java. "Third-party trusted Python" is not a valid upstream source type.

---

**Q20. (DRAG DROP)** You are implementing a package management solution for a Node.js application using Azure Artifacts. You need to configure the development environment to connect to the package repository. The solution must minimize the likelihood that credentials will be leaked. Which file should you use to configure each connection?

Files: The .npmrc file in the project / The .npmrc file in the user's home folder / The Package.json file in the project / The Project.json file in the project

| Connection | File |
|---|---|
| Registry information | ? |
| Credentials | ? |

**A:**
- **Registry information → The .npmrc file in the project** — contains the registry URL (`registry=...`) only. Safe to commit to git because it holds no credentials.
- **Credentials → The .npmrc file in the user's home folder** — contains the actual credentials (`$home/.npmrc` on Linux/Mac or `$env.HOME/.npmrc` on Windows). Never committed to source control, so credentials cannot be leaked via git. The npm client reads the project `.npmrc` to discover the registry, then fetches matching credentials from the home `.npmrc`.

Package.json and Project.json are dependency manifest files — they do not configure authentication.

---

**Q21.** You plan to share packages that you wrote, tested, validated, and deployed using Azure Artifacts. You need to release multiple builds of each package using a single feed. The solution must limit the release of packages that are in development.

A. global symbols  B. local symbols  C. upstream sources  D. views

**A: D — Views.**
Azure Artifacts views control which package versions are visible to consumers within a single feed. Packages in development stay in `@Local` (invisible to external consumers); only packages explicitly promoted to `@Prerelease` or `@Release` are visible. **Upstream sources** are for consuming packages from external registries — not for controlling visibility of your own packages. Global/local symbols are for debugging.

---

**Q23. — Anonymous access to Azure Artifacts package, minimize publication points**
> You use Azure Artifacts to host NuGet packages. You need to make one package available to anonymous users outside your organization. The solution must minimize the number of publication points. What should you do?
>
> A. Create a new feed for the package  B. Publish to a public NuGet repository  C. Promote the package to a release view  D. Change the feed URL

**A: A — Create a new feed for the package.**
Create a new feed with Public visibility — anonymous users outside the org can access it with no authentication. This is a single publication point. Publishing to nuget.org (B) creates a second publication point outside Azure Artifacts. Promoting to a release view (C) controls visibility *within* consumers of that feed but does not enable anonymous external access. Changing the feed URL (D) has no effect on access permissions.

---

**Q22.** You plan to use a NuGet package in a project in Azure DevOps. The NuGet package is in a feed that requires authentication. You need to ensure that the project can restore the NuGet package automatically. What should the project use to automate the authentication?

A. an Azure Automation account  B. an Azure Artifacts Credential Provider  C. an Azure AD account with MFA enabled  D. an Azure AD service principal

**A: B — Azure Artifacts Credential Provider.**
The Azure Artifacts Credential Provider is an installable tool that automates NuGet authentication against Azure Artifacts feeds in CI/CD environments. It integrates with `dotnet restore`, `nuget restore`, and `msbuild /t:Restore`, acquiring tokens silently without user interaction. **Azure Automation account** = runbook/process automation, unrelated to NuGet auth. **Azure AD account with MFA** = requires interactive login — cannot be automated. **Azure AD service principal** = valid identity type, but the credential provider is the specific tool that handles NuGet credential acquisition and wires the identity to the feed.

---

## 8. Quick Reference Checklist

### Dependency Management Fundamentals
- [ ] Know difference: source componentization vs package componentization
- [ ] Know package manifest files: `packages.config`, `package.json`, `pom.xml`, `requirements.txt`
- [ ] Identify package types: NuGet, npm, Maven, Python, Universal

### Azure Artifacts
- [ ] 4 roles: Reader, Collaborator, Contributor, Owner — know permissions of each
- [ ] Upstream sources: proxy + cache public registries; cannot publish duplicate versions
- [ ] 3 views: @Local (all), @Prerelease (validated), @Release (production-ready)
- [ ] Promotion = one-way: Local → Prerelease → Release
- [ ] Feed scopes: project-scoped (isolated) vs organization-scoped (org-wide)
- [ ] Use `NuGetAuthenticate@1` before NuGet push/restore in pipelines
- [ ] Universal Packages = any binary file type (no public registry equivalent)

### Semantic Versioning
- [ ] Format: MAJOR.MINOR.PATCH[-prerelease]
- [ ] MAJOR = breaking changes (reset MINOR/PATCH to 0)
- [ ] MINOR = new backwards-compatible features (reset PATCH to 0)
- [ ] PATCH = backwards-compatible bug fixes
- [ ] Deprecation (kept functional) = MINOR, not MAJOR
- [ ] Prerelease order: alpha < beta < rc < release

### GitHub Packages
- [ ] Supported: npm, NuGet, Maven, Docker (GHCR), RubyGems
- [ ] GITHUB_TOKEN = current repo only (automatic in GitHub Actions)
- [ ] PAT with `write:packages` = cross-repo or external CI
- [ ] Deleted packages restorable within **30 days**
- [ ] Visibility: Private / Public / Internal (Enterprise)
- [ ] No built-in views or upstream source proxying (unlike Azure Artifacts)

### Pipeline Artifacts vs Azure Artifacts
- [ ] Pipeline artifacts: temporary, per-run, pass files between jobs/stages
- [ ] Azure Artifacts: versioned, persistent, shared across projects/teams
- [ ] `PublishBuildArtifacts` / `PublishPipelineArtifact` = pipeline artifacts (NOT feed packages)

### Security
- [ ] Commit lock files (`package-lock.json`, `packages.lock.json`) for reproducible builds
- [ ] Use upstream sources to control which external packages enter your environment
- [ ] Scan dependencies: Dependabot, Snyk, OWASP Dependency-Check, WhiteSource/Mend
- [ ] Packages from upstream sources are immutable once cached
