# AZ-400 — Git for Enterprise DevOps

> **Exam Weight:** "Design and implement a source control strategy" = **10–15%** | "Design and implement processes and communications" = **10–15%**
> **MS Learn Path:** [AZ-400: Development for Enterprise DevOps](https://learn.microsoft.com/en-us/training/paths/az-400-work-git-for-enterprise-devops/)

---

## Table of Contents
1. [Introduction to DevOps](#1-introduction-to-devops)
2. [Agile Planning — GitHub Projects & Azure Boards](#2-agile-planning--github-projects--azure-boards)
3. [Branch Strategies & Workflows](#3-branch-strategies--workflows)
4. [Pull Requests & Code Review in Azure Repos](#4-pull-requests--code-review-in-azure-repos)
5. [Git Hooks](#5-git-hooks)
6. [Inner Source & Fork Workflow](#6-inner-source--fork-workflow)
7. [Manage & Configure Repositories](#7-manage--configure-repositories)
8. [Technical Debt & Code Quality](#8-technical-debt--code-quality)
9. [Additional Exam Topics (Beyond MS Learn)](#9-additional-exam-topics-beyond-ms-learn)

---

## 1. Introduction to DevOps

### What is DevOps?
- **Definition:** Union of **people, process, and products** to enable continuous delivery of value.
- DevOps is not a tool or a role — it is a culture and set of practices.
- Goal: Shorten the **systems development lifecycle** while delivering features, fixes, and updates frequently.

### Core Metrics (Exam Favorite)
| Metric | Definition |
|---|---|
| **Lead Time** | Time from idea/request to production delivery |
| **Cycle Time** | Time from work starting to work being delivered |
| **Time to Recovery (MTTR)** | Time to restore service after an incident |
| **Change Failure Rate** | % of changes that result in degraded service |

> **Exam tip:** These four are the DORA metrics. Know what each measures and how to reduce them.

### DevOps Transformation
- Identify a **transformation team** — small, cross-functional group dedicated to the change.
- Set **shared goals** and define timelines collaboratively.
- Use **validated learning** — data-driven feedback loops to confirm progress.
- **Cycle time optimization** = focus on reducing waste in the pipeline.

### Organization Structure for Agile
- **Vertical teams** (feature teams) are preferred over horizontal/functional silos.
- Align teams around products/services, not technology layers.
- Each team should own end-to-end delivery.

### Azure DevOps vs GitHub
| Feature | Azure DevOps | GitHub |
|---|---|---|
| Work tracking | Azure Boards | GitHub Projects / Issues |
| Source control | Azure Repos | GitHub Repos |
| CI/CD | Azure Pipelines | GitHub Actions |
| Artifact management | Azure Artifacts | GitHub Packages |
| Test management | Azure Test Plans | (limited) |

> **Exam tip:** Azure DevOps is preferred for enterprises needing full ALM (Application Lifecycle Management). GitHub is preferred for open-source and developer-centric workflows.

### Source Control Fundamentals
- **Centralized VCS** (e.g., TFVC): Single server, direct commits. Simple but single point of failure.
- **Distributed VCS** (e.g., Git): Every developer has a full repo copy. Offline work possible. Industry standard.
- Git operations: `clone`, `commit`, `push`, `pull`, `fetch`, `merge`, `rebase`

### License Management Strategy
- Azure DevOps access levels: **Basic**, **Basic + Test Plans**, **Stakeholder** (free, limited)
- **Stakeholder access**: Can view/create work items but cannot access pipelines or full repos.
- GitHub access: Organization members, outside collaborators, teams with repository permissions.

---

## 2. Agile Planning — GitHub Projects & Azure Boards

### Azure Boards
- Work item types: **Epic → Feature → User Story → Task → Bug**
- **Process templates:** Agile, Scrum, CMMI (Capability Maturity Model Integration)
- **Backlogs:** Product backlog, sprint backlog, task board
- **Kanban boards:** Visualize flow, set WIP limits
- **Queries:** Custom queries to filter/report on work items
- **Velocity charts:** Measure team throughput per sprint
- **Cumulative Flow Diagram (CFD):** Shows bottlenecks and flow health

### GitHub Projects
- GitHub Projects is a spreadsheet/kanban hybrid for managing work.
- Supports custom fields, views (table, board, roadmap), and automation.
- **Project Boards:** Kanban-style columns (To Do, In Progress, Done)
- Items can be issues, pull requests, or draft items.

### Linking Azure Boards and GitHub
- Install the **Azure Boards App** from GitHub Marketplace.
- Once connected, commit messages can reference work items: `AB#<work-item-id>`
- PRs mentioning `AB#123` will update the Azure Boards work item status.
- Enables **traceability**: commit → PR → work item → deployment.

**Where AB# syntax triggers automatic work item linking (Exam Favorite):**

| Location | AB# linking works? |
|----------|--------------------|
| **PR title** | ✅ Yes |
| **PR description (body)** | ✅ Yes |
| **Commit message** | ✅ Yes |
| PR comment | No |
| Milestone | No |
| Label | No |

> **Exam tip:** `AB#<WORKITEMNUMBER>` automatically links a GitHub PR to an Azure Boards work item only when placed in the PR **title** or **description**, or in a **commit message**. Comments, milestones, and labels are not parsed for AB# references.

### GitHub + Azure DevOps Integration — Which App to Install? (Exam Favorite)
This is a frequently tested trick question. Know the difference:

| Goal | App to Install | Why |
|---|---|---|
| **Display commit status on Azure Boards** (link commits/PRs to work items) | **Azure Pipelines app** on GitHub | The Pipelines app establishes the GitHub↔Azure DevOps connection that surfaces build/commit status in Boards |
| **Link GitHub issues/PRs to Azure Boards work items** | **Azure Boards app** on GitHub | Allows AB# mentions to update work item state |
| **Trigger Azure Pipelines from GitHub events** | **Azure Pipelines app** on GitHub | Enables CI/CD from GitHub repos |

> **Exam tip (tested scenario):** "You have a private GitHub repository. You need to display the commit status on Azure Boards. What should you do **first**?" → **Add the Azure Pipelines app to the GitHub repository** (not the Azure Boards app). The Pipelines app creates the integration channel; without it, commit status cannot flow into Azure DevOps.

### Feedback Cycles Strategy
- **Inner feedback loops:** Developer tests, code reviews, unit tests (fast, low cost).
- **Outer feedback loops:** Production monitoring, user telemetry, A/B tests (slower, broader).
- Design notifications for: build failures, PR reviews, deployment completions.
- GitHub Issues can be used for external/community feedback capture.

### Source, Bug, and Quality Traceability
- Every code change should trace back to a work item (story, bug, task).
- Use **branch naming conventions** tied to work items (e.g., `feature/AB#123-login-page`).
- Quality gates in pipelines enforce that tests pass before merge.
- Bug traceability: Bug → fix commit → PR → deployment → verification.

### DevOps Dashboard Metrics
- **Flow metrics:** Cycle time, lead time, throughput, WIP count
- **Azure DevOps Analytics views:** Built-in reporting for boards, pipelines, and repos
- Configurable dashboards in Azure DevOps using widgets

### Team Discussions (GitHub)
- GitHub **Team Discussions** allow async communication scoped to a team or organization.
- Use for architectural decisions, announcements, and knowledge sharing.

---

## 3. Branch Strategies & Workflows

### Branch Workflow Types

#### 1. Feature Branch Workflow
- Each feature/fix is developed in its own branch.
- Main branch is always deployable (stable).
- PRs used to merge feature branches back to `main`.
- **Best for:** Small to medium teams, continuous delivery.

#### 2. GitFlow
- Defined branches: `main` (or `master`), `develop`, `feature/*`, `release/*`, `hotfix/*`
- `develop` = integration branch; `main`/`master` = production-ready
- Release branches cut from `develop`; hotfixes cut from `main`
- **Best for:** Teams with scheduled releases, longer release cycles.
- **Drawback:** Complex; long-lived branches cause merge conflicts.

**GitFlow branch roles (Exam Favorite — HOTSPOT):**

| Branch | Role | Contains |
|---|---|---|
| `main` / `master` | **Production code** | Stable, released code deployed to production |
| `develop` | **Preproduction code** | Integrated features ready for the next release; merged from feature branches |
| `feature/*` | Feature work | Individual feature development; merged into develop when complete |
| `release/*` | Release preparation | Bug fixes only; branched from develop, merged to main + develop |
| `hotfix/*` | Emergency fixes | Branched from main; merged back to main + develop |

> **Exam tip (tested — HOTSPOT):** "GitFlow — which branch is production code? Which is preproduction?" → **Production = Master/Main**, **Preproduction = Develop**. Feature branches are not production or preproduction — they are individual work-in-progress branches.

#### 3. GitHub Flow
- Only `main` + short-lived feature branches.
- Merge to `main` triggers deployment.
- Simple: branch → commit → PR → review → merge → deploy
- **Best for:** Continuous delivery, web applications, frequent deployments.

#### 4. Trunk-Based Development
- All developers commit directly to `main` (trunk) or very short-lived branches.
- Requires feature flags to hide incomplete features.
- **Best for:** High-performing DevOps teams, CI/CD, microservices.
- Reduces integration debt; requires good test coverage.

#### 5. Fork Workflow
- Contributors fork the repo to their own account.
- Work done in fork, then PR submitted to upstream.
- **Best for:** Open-source, inner source, external contributors.

### Comparison Table (Exam Favorite)
| Strategy | Branch Lifespan | Release Cadence | Complexity | Use Case |
|---|---|---|---|---|
| Feature Branch | Short | Continuous | Low | Standard teams |
| GitFlow | Long | Scheduled | High | Enterprise releases |
| GitHub Flow | Very short | Continuous | Very low | CD web apps |
| Trunk-Based | Near-zero | Continuous | Medium | High-performance teams |
| Fork Workflow | Variable | Any | Medium | OSS / inner source |

### Git Branch Model for Continuous Delivery
- Aim for **short-lived branches** (< 1–2 days) to reduce merge conflicts.
- Use **feature flags** to decouple deployment from release.
- Release branches should only receive bug fixes, not new features.

### Branch Merging Restrictions (Azure Repos)
Configure via **Branch Policies** in Azure Repos:
- **Require a minimum number of reviewers** — e.g., at least 2 approvals
- **Check for linked work items** — enforce traceability
- **Check for comment resolution** — all PR comments must be resolved
- **Limit merge types** — allow only squash, rebase, or merge commit
- **Require CI build to pass** — mandatory build validation
- **Require successful deployments** — deployment gates before merge

### Branch Protections (GitHub)
- Configured per branch (e.g., `main`) under repository settings.
- Options: require PR reviews, require status checks, restrict who can push, require signed commits, enforce for admins.

### Version Control with Git in Azure Repos
- Azure Repos supports both **Git** and **TFVC** (Team Foundation Version Control).
- Recommended: **Git** for all new projects.
- Clone URL formats: HTTPS or SSH.
- Default branch: configurable (typically `main`).

---

## 4. Pull Requests & Code Review in Azure Repos

### Pull Request (PR) Workflow
1. Developer creates a feature branch.
2. Pushes commits and opens a PR targeting the main/develop branch.
3. Reviewers are notified (auto-assigned or manually added).
4. Reviewers comment, request changes, or approve.
5. All required policies pass (build, reviews, tests).
6. PR is completed (merged) using the allowed merge strategy.

### PR Best Practices
- **Small PRs** are easier to review and less risky.
- Write a clear PR description with context and testing notes.
- Link to the related work item.
- Use **draft PRs** for work-in-progress that is not ready for review.

### Branch Policies for PR Governance (Azure Repos)
| Policy | Purpose |
|---|---|
| Minimum reviewer count | Prevents self-merge; ensures peer review |
| Require linked work item | Enforces traceability |
| Resolve all comments | Ensures no feedback is ignored |
| Build validation | Automated CI check must pass |
| Status checks | External service checks (e.g., SonarCloud) |
| Merge strategy restrictions | Enforce squash/rebase for clean history |
| Required reviewers (specific users/groups) | Ensures domain experts review |

### Build Policy vs Status Policy (Exam Favorite)
This distinction is frequently tested. The question typically asks: *"What policy type ensures X before merging?"*

| Requirement | Policy Type | Explanation |
|---|---|---|
| Code must **compile successfully** | **Build policy** (Build Validation) | Triggers an Azure Pipelines build; fails if build fails |
| SonarCloud **Quality Gate must be "Passed"** | **Status policy** | Requires an external service to post a pass/fail status to the PR |
| Tests must pass | **Build policy** | Tests run as part of the build |
| Security scan must pass (external tool) | **Status policy** | External service posts status via API |

> **Exam tip:** "Build Validation" = internal Azure Pipelines build. "Status policy" = external service result (SonarCloud, third-party scanner, etc.). These are two separate policy types in Azure Repos settings.

**Bypassing branch policies — least privilege approach:**

A branch policy (e.g., build must succeed) blocks ALL merges unless the policy passes. To allow a specific user to merge even when the policy fails, grant them the **"Bypass policies when completing pull requests"** permission.

| Permission location | Scope | Least privilege? |
|---|---|---|
| **Branch Security settings** (Repos → Branches → … → Security) | That branch only | **Yes** — most targeted |
| Repository Security settings | All branches in the repo | No — too broad |
| Build Administrators group | Build pipeline management only | No — wrong permission |
| Project Administrators group | Full project control | No — far too broad |

> **Exam tip:** "Allow a specific user to bypass a branch policy (e.g., merge even if build fails) using least privilege" → **branch-level Security settings**. Repository-level security applies to all branches (too broad). Project Administrators = full project control (violates least privilege). Build Administrators manage build definitions, not branch policy bypasses.

### Merge Strategies
| Strategy | History impact | Reduces volume? | Use Case |
|---|---|---|---|
| **Merge commit** (three-way merge) | Preserves ALL commits + adds a merge commit | ❌ No — increases history | Default; full audit trail |
| **Squash merge** | Collapses all PR commits into **one commit** on target | ✅ Yes — minimises history volume | Clean history; individual commits lost |
| **Rebase and fast-forward** | Replays PR commits linearly, no merge commit | Partially — no extra merge commit, but all individual commits kept | Clean linear log; rewrites history |

> **Exam tip:** Squash merge is commonly tested — it combines all PR commits into a single commit on the target branch.
>
> **Exam tip (tested — three-way merge):** A **three-way merge** (standard merge commit) **preserves** all individual commits and adds a merge commit — it **increases** history volume, not reduces it. To **reduce** history volume → use **squash merge**. Three-way merge ≠ squash.

### Code Review Strategy
- **Purpose:** Catch bugs early, share knowledge, enforce standards.
- **Reviewer selection:** Domain experts + team rotation for knowledge spread.
- **Review checklist:** Logic correctness, security, performance, test coverage, naming.
- **Automated code analysis** reduces reviewer burden on style/lint issues.

### PR Actions in Azure Repos (Exam Favorite)
Beyond approve/reject, PRs in Azure Repos have specific actions. Know these:

| Action | When to Use |
|---|---|
| **Cherry-pick** | Copy specific commits from a PR into a new branch (only a **portion** of the PR code). Creates a new topic branch with copied changes. |
| **Revert** | Create a new PR that undoes all changes from a completed PR. Does NOT delete the commit — it adds a reverting commit. |
| **Set as default branch** | Change the repo's default branch. |
| **Reactivate** | Reopen an abandoned PR. |
| **Approve with suggestions** | Approve but leave optional comments for the author. |

> **Exam tip (tested scenario):** "You need to create a new branch from an existing PR using **only a portion** of the code." → Answer: **Cherry-pick**. Cherry-pick selects specific commits; Revert undoes an entire completed PR.

### Compliance & Audit Trails
- All PR activity is logged: who reviewed, when, comments, approvals.
- Azure DevOps audit logs capture security-relevant events.
- Required reviewers + branch policies create an auditable governance framework.

---

## 5. Git Hooks

### What are Git Hooks?
- Scripts that run **automatically** at specific points in the Git workflow.
- Located in the `.git/hooks/` directory of a repository.
- Can be written in any scripting language (Bash, PowerShell, Python).
- **Client-side hooks:** Run on developer's machine.
- **Server-side hooks:** Run on the remote server (e.g., Azure Repos).

### Client-Side Hook Types (Exam Favorite)
| Hook | Trigger | Common Use |
|---|---|---|
| `pre-commit` | Before commit is created | Lint checks, secrets scanning, formatting |
| `commit-msg` | After commit message is entered | Enforce commit message format (e.g., must reference AB#ID) |
| `post-commit` | After commit is created | Notifications, local logging |
| `pre-push` | Before `git push` runs | Run unit tests, prevent push if tests fail |
| `prepare-commit-msg` | Before commit message editor opens | Auto-populate message template |

### Server-Side Hook Types
| Hook | Trigger | Common Use |
|---|---|---|
| `pre-receive` | Before refs are updated on push | Enforce push policies |
| `update` | Per branch on push | Branch-specific validations |
| `post-receive` | After push completes | Trigger CI/CD, send notifications |

### Practical Examples
```bash
# pre-commit hook: block commits with TODO comments
#!/bin/bash
if grep -r "TODO" --include="*.cs" .; then
  echo "ERROR: Remove TODO comments before committing."
  exit 1
fi
```

```bash
# commit-msg hook: enforce AB# work item tag
#!/bin/bash
MSG=$(cat "$1")
if ! echo "$MSG" | grep -qE "AB#[0-9]+"; then
  echo "ERROR: Commit message must contain a work item tag (AB#ID)."
  exit 1
fi
```

### Bypassing Git Hooks (Exam Favorite)
- Use `git commit --no-verify` to skip **both** the `pre-commit` and `commit-msg` hooks.
- Use `git push --no-verify` to skip the `pre-push` hook.
- **This is a frequently tested scenario:** "You create a `commit-msg` hook requiring a work item tag. You need to commit without a tag. Which parameter do you use?" → **`--no-verify`**
- `--no-verify` does NOT bypass server-side hooks (`pre-receive`, `post-receive`).

> **Exam tip:** `--squash`, `--message`, and `--no-post-rewrite` are common wrong answers. Only `--no-verify` bypasses hooks.

### Secrets Prevention with Hooks
- `pre-commit` hooks can scan for patterns matching credentials (passwords, API keys, connection strings).
- Tools like **detect-secrets**, **git-secrets**, or **truffleHog** can be integrated.
- Prevents accidental secret commits before they reach the remote.

### Sharing Hooks with a Team
- `.git/hooks/` is NOT committed to the repo (it's inside `.git/`).
- Solutions:
  - Store hooks in a `/hooks` folder in the repo and configure Git: `git config core.hooksPath ./hooks`
  - Use tools like **Husky** (Node.js) or **pre-commit** framework.
  - Use Azure DevOps Service Hooks for server-side event triggers.

### Git Hooks vs Azure DevOps Service Hooks
| Aspect | Git Hooks | Azure DevOps Service Hooks |
|---|---|---|
| Location | Client or server | Azure DevOps platform |
| Scope | Individual repo | Org/project wide |
| Language | Shell scripts | Webhooks, external services |
| Use case | Enforce local standards | Integrate with external tools (Teams, Slack, Jenkins) |

---

## 6. Inner Source & Fork Workflow

### What is Inner Source?
- Applying **open-source collaboration practices** within an organization.
- Teams contribute to each other's codebases like open-source contributors.
- Promotes **code reuse**, **knowledge sharing**, and **cross-team collaboration**.
- Not to be confused with open source — the code stays internal.

### Benefits of Inner Source
- Breaks down silos between teams.
- Reduces duplicate code across the org.
- Increases code quality through broader review.
- Enables contributors outside the owning team.

### Fork Workflow
1. **Upstream repo:** The official, canonical repository (owned by a team).
2. **Fork:** A copy of the upstream repo in a different namespace (another user/team).
3. Developer clones the fork, makes changes, and submits a PR to the upstream.
4. Upstream maintainers review and merge (or reject).

```
[Upstream Repo] ← PR ← [Fork (Contributor)]
```

### Branches vs Forks
| Aspect | Branch | Fork |
|---|---|---|
| Ownership | Same repo | Separate repo copy |
| Visibility | Team members | Contributor controls their fork |
| Use case | Internal team dev | External/cross-team contribution |
| PR target | Same repo | Upstream (original) repo |

> **Exam tip:** Forks are the preferred model for inner source. Use branches within a team, forks for cross-team contributions.

### Sharing Code Between Forks
- Sync a fork with the upstream using `git fetch upstream` + `git merge upstream/main`.
- Keep fork up-to-date to minimize merge conflicts.
- In Azure Repos, forks are created at the project level and can be PR'd back to the original.

### Fork Workflow in Azure DevOps
- Create forks via Azure Repos: Repository → Fork.
- Fork inherits policies from the upstream but can have its own settings.
- Contributors can only push to their fork; they PR to upstream.
- Upstream team maintains code quality via PR policies.

### Fork Workflow Setup — 3-Step Sequence (Exam Favorite)
This exact sequence is tested. Scenario: *"Team2 needs to submit PRs for Project2. They must work independently, and intermediary changes must pass the same build policy."*

**Correct sequence:**
1. **Create a repository** for Team2 (if they don't have one yet)
2. **Create a fork** of Project2 into Team2's project
3. **Add a build validation policy** on the fork's branch (so changes must build before PR to upstream)

> **Why this sequence?** Team2 has no direct access to Project2 — they must work on a fork. The build policy on the fork enforces the same quality gate before a PR is raised to the upstream. Direct branch access would bypass the separation requirement.

---

## 7. Manage & Configure Repositories

### Monorepo vs Multiple Repos
| Aspect | Monorepo | Multiple Repos (Polyrepo) |
|---|---|---|
| All code in one repo | Yes | No |
| Dependency management | Easier | Harder (versioning needed) |
| CI/CD complexity | Higher (selective builds needed) | Lower per-repo |
| Code sharing | Easy | Requires packages/artifacts |
| Used by | Google, Facebook, Microsoft | Most standard orgs |

> **Exam tip:** Know the trade-offs. Monorepo needs tooling like Scalar/VFS for Git to scale.

### Large Repository Solutions

#### Git Large File Storage (LFS)
- Stores large binary files (images, videos, datasets) outside the Git repo.
- Git stores a **pointer file** instead of the actual binary.
- Actual files stored on LFS server (Azure Repos or GitHub LFS).
- Configure: `git lfs track "*.psd"` → adds to `.gitattributes`
- **Exam tip:** Use LFS for binary assets; NOT for source code.

#### Scalar
- A Git extension by Microsoft for very large repositories.
- Uses **partial clone**, **sparse checkout**, and **background prefetch**.
- Replaces the older **VFS for Git (GVFS)** for most scenarios.
- `scalar clone <url>` — clones a large repo efficiently.
- Uses **SSH URL format** for Azure Repos: `git@ssh.dev.azure.com:v3/organisation/project/repo`

#### git sparse-checkout — Minimize Transferred Data (Exam Favorite)

Two patterns appear on the exam depending on whether `git clone` or `scalar clone` is used:

**Pattern 1 — `git clone` + sparse-checkout (plain Git):**
```bash
cd repos
git clone https://dev.azure.com/organisation/_git/Repo1
git sparse-checkout set directory1
```

**Pattern 2 — `scalar clone` + cd + sparse-checkout (Scalar / large repos):**
```bash
cd repos
scalar clone git@ssh.dev.azure.com:v3/organisation/project/repo1
cd repo1/src/web
git sparse-checkout set src/web
```

| Command | Purpose |
|---------|---------|
| `git clone <https-url>` | Clones via HTTPS |
| `scalar clone <ssh-url>` | Clones via SSH using Scalar (large repo optimized) |
| `git sparse-checkout set <dir>` | Limits working tree to specified directory only |

> **Exam tip:** When the script uses `scalar clone`, the URL is the **SSH format** (`git@ssh.dev.azure.com:v3/...`), not the HTTPS format. When the script uses `git clone`, the URL is HTTPS (`https://dev.azure.com/...`). Both patterns end with `git sparse-checkout set` to limit the checkout scope.

#### VFS for Git (GVFS)
- Virtualizes the filesystem so only accessed files are downloaded.
- Used by Microsoft for the Windows source repo.
- Requires a GVFS-protocol-enabled server.

### Cleaning Up Repository Data

#### Recover Specific Data (Git Commands)
| Command | Purpose |
|---|---|
| `git reflog` | Show history of HEAD movements; recover lost commits |
| `git cherry-pick <SHA>` | Apply a specific commit to current branch |
| `git revert <SHA>` | Create a new commit that undoes a previous commit |
| `git reset --hard <SHA>` | Move HEAD to specific commit (destructive) |
| `git stash` | Temporarily shelve uncommitted changes |

#### Recovering a File Deleted and Committed (Exam Favorite — DRAG DROP)

Scenario: you deleted a file, committed the deletion, and continued working. You need to recover the file.

**Three commands in sequence:**

| Step | Command | Why |
|---|---|---|
| 1 | `git log` | Find the commit hash (SHA) of the commit that deleted the file |
| 2 | `git checkout [hash]-1 -- path/to/file` | Restore the file from the commit **before** the deletion commit (`[hash]~1` or `[hash]-1` = parent commit). This restores the file to the working tree **and stages it**. |
| 3 | `git commit -m 'undeleted the file'` | Commit the restored file to preserve the recovery |

> **Why NOT `git restore path/to/file`?** `git restore path/to/file` (without `--source`) restores from HEAD — which is the state **after** deletion, so it would re-delete the file. It is a distractor.
>
> **Why NOT `git stash` or `git tag`?** Stash saves uncommitted changes; tag creates a version label. Neither recovers a committed deletion.
>
> **Exam tip (tested — DRAG DROP):** "Deleted a file, committed, need to recover" → **`git log` → `git checkout [hash]-1 -- path/to/file` → `git commit -m 'undeleted the file'`** in that order.

#### Remove Data from Source Control (Exam Favorite)
- **Problem:** Secrets or large files accidentally committed need to be purged.
- Tools:
  - **`git filter-repo`** (recommended): Rewrites Git history to remove files/content.
  - **BFG Repo Cleaner:** Fast alternative to `git filter-branch`. Removes large files or sensitive data from all commits in history.

> **Exam tip:** "Large file accidentally committed, need to reduce repository size, must remove the file" → **BFG Repo Cleaner**. Git LFS tracks large files but does not remove history. GVFS/Scalar optimizes repo access for large repos but does not remove files from history.
  - `git filter-branch` (legacy, slow, not recommended).
- After purging, must **force-push** all branches and notify all contributors to re-clone.
- In Azure Repos: use "Destroy" or "Obliterate" via API to remove commits from server.

> **Exam tip:** After removing secrets with filter-repo, the secret must be rotated/invalidated — purging history doesn't protect against anyone who already cloned the repo.

### Release Management with GitHub

#### Managing Releases
- GitHub **Releases** are based on Git **tags**.
- A release packages source code + binary artifacts at a specific commit.
- Release notes can be auto-generated from merged PRs and commits.

#### Tags
- **Lightweight tag:** Just a pointer to a commit. `git tag v1.0`
- **Annotated tag:** Full object with message, tagger info, date. `git tag -a v1.0 -m "Release 1.0"`
- Annotated tags are preferred for releases (contain metadata).
- Push tags: `git push origin --tags`

> **Exam tip (DRAG DROP):** "Finalize a release in GitHub, apply labels: Name, Email, Release v3.0, Release date" → `git tag -a v3.0 -m "Release v3.0"`. The `-a` flag creates an **annotated** tag which automatically stores name, email, and date from git config. Without `-a`, a lightweight tag stores no metadata.

### Automated Release Notes
- GitHub can **auto-generate** release notes based on merged PRs since last release.
- Customize via `.github/release.yml` — group PRs by label (bug fix, feature, etc.)
- Azure DevOps: Use Wiki pages or pipeline tasks to auto-document releases.

### Repository Permissions (GitHub)
| Role | Permissions |
|---|---|
| **Read** | View and clone repo |
| **Triage** | Read + manage issues/PRs (no code changes) |
| **Write** | Read + push branches, manage releases |
| **Maintain** | Write + manage repo settings (no destructive actions) |
| **Admin** | Full control including deletion and security settings |

### Repository Permissions (Azure Repos)
- Set at Organization, Project, or Repo level.
- Use **Security Groups** (Readers, Contributors, Project Administrators, Build Administrators).
- Fine-grained permissions: Contribute, Manage Permissions, Force Push, Create Branch, Delete Branch.

### GitHub Tags for Organizing Repositories
- **Topics/Tags** label repositories for discoverability (e.g., `azure`, `devops`, `api`).
- Useful in large organizations with many repositories.
- Set under repo Settings → About section.

### API Documentation
- Generate API docs from code comments using tools: **Swagger/OpenAPI**, **DocFX**, **Doxygen**.
- Automate doc generation as part of the CI pipeline.
- Publish to GitHub Pages or Azure Static Web Apps.

### Automate Git History Documentation
- `git log --oneline --graph` — visual commit history
- Tools like **git-changelog** or GitHub's auto-release notes generate changelogs from commit history.
- Conventional commits format (`feat:`, `fix:`, `chore:`) enables automated changelog generation.

---

## 8. Technical Debt & Code Quality

### What is Technical Debt?
- **Definition:** The cost of rework caused by choosing an easy solution now instead of a better approach that would take longer.
- Types: Code debt, architecture debt, test debt, documentation debt, infrastructure debt.
- Technical debt **accumulates interest** — the longer it's left, the more expensive to fix.

### Code Quality Dimensions
| Dimension | Description |
|---|---|
| **Reliability** | Code works correctly under expected conditions |
| **Maintainability** | Easy to read, modify, and extend |
| **Security** | Free of vulnerabilities |
| **Performance** | Meets response time and throughput targets |
| **Test Coverage** | Percentage of code tested by automated tests |

### Complexity & Quality Metrics
| Metric | Definition |
|---|---|
| **Cyclomatic Complexity** | Number of linearly independent paths through code. Higher = harder to test/maintain. |
| **Cognitive Complexity** | How difficult code is to understand (SonarQube metric). |
| **Code Coverage** | % of lines/branches exercised by tests. |
| **Code Churn** | How frequently code changes. High churn = instability or rework. |
| **Duplication** | % of duplicated code blocks. |
| **SQALE rating** | Technical debt ratio used by SonarQube. |

### Measuring & Managing Technical Debt
- **SonarQube / SonarCloud:** Static analysis platform. Identifies bugs, code smells, security vulnerabilities, and technical debt estimates.
- **Code smells:** Long methods, duplicated code, magic numbers, deep nesting.
- Debt is measured in **estimated remediation time**.
- Prioritize debt based on: effort to fix vs. risk/impact.
- Include tech debt items in the backlog alongside feature work.

### GitHub Advanced Security (GHAS)
- Available for GitHub Enterprise and Azure DevOps (GitHub Advanced Security for Azure DevOps).
- Three main capabilities:

| Feature | Purpose |
|---|---|
| **Code Scanning (CodeQL)** | Analyzes code for security vulnerabilities. Runs as CI check. |
| **Secret Scanning** | Detects hardcoded secrets (API keys, tokens, passwords) in code. |
| **Dependency Review / Dependabot** | Identifies vulnerable dependencies; opens PRs to update them. |

> **Exam tip:** GHAS is heavily tested in the Security section of AZ-400. Know what each feature detects and how to configure it.

### Code Quality Tools Integration
| Tool | Purpose | Integration |
|---|---|---|
| **SonarQube/SonarCloud** | Comprehensive static analysis | Azure Pipelines extension, PR decoration |
| **ESLint / Pylint / StyleCop** | Language-specific linting | Pre-commit hooks, pipeline tasks |
| **CodeQL** | Security-focused analysis | GitHub Actions, Azure DevOps |
| **Dependabot** | Dependency vulnerability alerts | GitHub (automatic PRs) |
| **WhiteSource/Mend** | Open-source license + vulnerability scanning | Azure Pipelines integration |
| **Coverlet / JaCoCo** | Code coverage for .NET / Java | Pipeline test tasks |

### Planning Effective Code Reviews
- **Goals:** Find bugs, share knowledge, enforce standards, improve design.
- **Review checklist:**
  - Does the code do what it claims?
  - Are there security issues?
  - Is test coverage adequate?
  - Is it readable and well-named?
  - Is there duplication?
- **Automation first:** Use linters and static analysis to catch style issues automatically, freeing reviewers for logic review.
- **Review size:** Keep PRs small (< 400 lines) for effective review.
- **Author checklist:** Self-review before requesting review.

---

## 9. Additional Exam Topics (Beyond MS Learn)

### Trunk-Based Development Deep Dive
- **Feature flags** (feature toggles) are required to enable trunk-based development.
- Types of feature flags:
  - **Release toggles:** Hide incomplete features until ready.
  - **Experiment toggles:** A/B test new features.
  - **Ops toggles:** Kill switches for production issues.
  - **Permission toggles:** Enable features for specific users/roles.
- **Azure App Configuration Feature Manager** is the Azure-native solution for feature flags.

### Git Credential Management

**Authentication methods for Git with Azure DevOps — comparison:**

| Method | Works with Git? | Minimizes credential prompts? | Notes |
|---|---|---|---|
| **Personal Access Token (PAT)** | ✅ Yes — HTTPS Git | ✅ Yes — stored by Git Credential Manager; enter once, cached thereafter | Recommended for Git auth to Azure DevOps |
| **SSH key** | ✅ Yes — SSH Git | ✅ Yes — key-pair, no password prompt after setup | Alternative to PAT |
| **Managed Identity (Azure AD)** | ❌ No — not usable for interactive Git operations | N/A | For Azure resources / pipeline service principals, not human Git auth |
| **User account (Azure AD)** | ✅ Yes (via OAuth/browser) | ❌ No — may prompt for MFA/re-auth frequently | Interactive login, less suitable for scripted/automated Git |
| **Alternate credentials** | ✅ Yes | ❌ No — basic username/password, re-entered each time | **Deprecated** — Microsoft has removed this option from Azure DevOps |

> **Exam tip (tested):** "Authenticate from Git + minimize credential prompts" → **Personal Access Token (PAT)**. PATs are stored by Git Credential Manager after first entry — no repeated prompts. Managed identities don't support Git authentication. Alternate credentials are deprecated and require re-entering. Azure AD user accounts may trigger MFA repeatedly.

- **Git Credential Manager (GCM):** Cross-platform tool that stores PATs/OAuth tokens securely after first use — this is what enables "minimize credentials" with PATs.
- **Service Connections** in Azure DevOps: Used by pipelines to authenticate to external services.

### Conventional Commits
- Specification for structuring commit messages to enable automated tooling.
- Format: `<type>[optional scope]: <description>`
- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`
- `feat:` triggers a MINOR version bump in SemVer; `fix:` triggers PATCH.
- `BREAKING CHANGE:` in footer triggers MAJOR version bump.

### Semantic Versioning (SemVer)
- Format: `MAJOR.MINOR.PATCH` (e.g., `2.1.0`)
- **MAJOR:** Breaking changes.
- **MINOR:** New backwards-compatible features.
- **PATCH:** Backwards-compatible bug fixes.
- Commonly used with automated release pipelines and Git tags.

### Git Rebase vs Merge
| Aspect | Merge | Rebase |
|---|---|---|
| History | Non-linear (merge commits) | Linear (no merge commits) |
| Safety | Safe; preserves all history | Rewrites history; avoid on shared branches |
| Use case | Merging to main, PRs | Cleaning up feature branch before PR |

> **Exam tip:** Never rebase a public/shared branch — it rewrites commit SHAs and breaks others' branches.

### Azure Repos vs GitHub Repos — Key Differences
| Feature | Azure Repos | GitHub |
|---|---|---|
| Branch policies | Granular, enterprise-grade | Branch protection rules |
| PR templates | Supported | Supported (`.github/pull_request_template.md`) |
| Fork workflow | Supported within Azure DevOps | Native to GitHub |
| GHAS | Via "GitHub Advanced Security for Azure DevOps" | Native GHAS |
| Service hooks | Azure DevOps Service Hooks | GitHub Webhooks + Actions |

### Webhooks & Service Hooks
- **Webhooks:** HTTP callbacks triggered by Git events (push, PR, merge).
- **Azure DevOps Service Hooks:** Send event notifications to external services.
  - Supported targets: Microsoft Teams, Slack, Jenkins, Service Bus, Webhooks, etc.
  - Configure in: Project Settings → Service hooks.
- **GitHub Webhooks:** Configure under repo/org Settings → Webhooks. Sends JSON payloads to a URL.

### Documentation Automation
- **Wikis in Azure DevOps:** Markdown-based, version-controlled (stored as a Git repo).
  - Two types: **Project Wiki** (auto-provisioned) and **Code Wiki** (publish a folder from a repo).
- **Mermaid diagrams:** Supported in Azure DevOps and GitHub wikis. Create flowcharts, sequence diagrams, ERDs in Markdown.
- Example Mermaid flowchart:

**Optimizing documentation for Git version control (Exam Tested — HOTSPOT):**

When process documentation (Word/.docx) and flow charts (.bmp) are stored in GitHub and need to be optimized for plain text, versioning, merging, and minimizing file count:

| Convert from | Convert to | Why |
|---|---|---|
| **.docx (Word)** | **Markdown (.md)** | Plain text — Git-diffable, mergeable, human-readable. LaTeX is also plain text but too complex. PDF is binary. |
| **.bmp (flow charts)** | **Mermaid graph diagrams (.md)** | Diagram-as-code embedded **inside** the .md file — plain text, Git-diffable, no separate image file needed |

> **Exam tip:** Mermaid diagrams embedded in Markdown satisfy ALL requirements simultaneously: plain text ✅, minimizes file count (diagram lives in the doc file) ✅, simplifies merging (text-based diff/merge) ✅. PNG/TIFF are binary image files — separate files, not diffable.
  ```
  graph LR
    A[Feature Branch] --> B[PR Review]
    B --> C{Policies Pass?}
    C -->|Yes| D[Merge to Main]
    C -->|No| E[Request Changes]
  ```
- **Automate docs from Git:** `git log --pretty=format:"%h %s" > CHANGELOG.md`

### Shift-Left Testing Philosophy
- Move testing **earlier** in the development lifecycle (to the left on the timeline).
- Developers write and run tests locally before pushing.
- CI pipeline runs tests on every commit.
- Reduces cost of bug detection — bugs found earlier are cheaper to fix.
- Contrast with shift-right: testing in production (canary, A/B, chaos engineering).

---

## 10. Past Exam Scenario Bank

Exact scenarios that have appeared in real AZ-400 exam dumps for this learning path.

---

**Q1 — Git Hooks: Bypass hook on commit**
> You create a client-side `commit-msg` hook requiring each commit message to contain a work item tag. You need to make a commit without a work item tag. Which `git commit` parameter do you use?

**A: `--no-verify`**
Bypasses both `pre-commit` and `commit-msg` hooks. Wrong answers: `--squash`, `--message ''`, `--no-post-rewrite`.

---

**Q2 — Branch Policy: Compile + SonarCloud Quality Gate**
> You need pull requests to satisfy two requirements before merging: (1) code must compile successfully, (2) SonarCloud Quality Gate must be "Passed". Which policy type for each?

**A:**
- Compile successfully → **Build Validation policy** (Build policy)
- SonarCloud Quality Gate → **Status policy**

---

**Q3 — PR Action: Only a portion of PR code**
> You plan to create a new branch from an existing pull request using only a portion of the code. Which pull request action should you use?

**A: Cherry-pick**
Cherry-pick copies only selected commits (not the entire PR). Revert undoes a completed PR entirely.

---

**Q4 — Fork Workflow: 3-step sequence**
> Team2 needs to submit pull requests for Project2, work independently on changes, and have all intermediary changes subject to the same build policy. Which three actions in sequence?

**A:**
1. Create a repository (for Team2's project)
2. Create a fork (of Project2)
3. Add a build validation policy (on the fork's branch)

---

**Q5 — GitHub + Azure Boards: Commit status integration**
> You have a private GitHub repository. You need to display the commit status of the repository on Azure Boards. What should you do first?

**A: Add the Azure Pipelines app to the GitHub repository**
(Not the Azure Boards app — the Pipelines app creates the integration that surfaces commit/build status in Azure DevOps.)

---

**Q6 — Version Control: Offline full history**
> Multiple developers will work offline frequently and require access to the full project history while offline. Which version control solution should you use?

**A: Git** (distributed VCS — every developer has a full local copy including complete history. TFVC is centralized and requires server connectivity.)

---

**Q7 — Code Quality: Unused variables & empty catch blocks**
> During code review, many modules contain unused variables and empty catch blocks. What is the best solution to improve code quality?

**A: Integrate a static code analysis tool** (e.g., SonarQube/SonarCloud) into the pipeline. These automatically detect code smells like unused variables, empty catch blocks, and dead code without manual review effort.

---

**Q8 — Git Distributed vs Centralized**
> Your team needs a version control system where developers can work on features without network connectivity to the server. Which type of VCS is best suited?

**A: Distributed VCS (Git)** — each clone is a full repository. Centralized VCS (TFVC, SVN) requires server connection for history and commits.

---

**Q12 — GitFlow: production vs preproduction branch (HOTSPOT)**
> Your company uses a Git repository and implements GitFlow as a workflow strategy. Which branch type is used for production code? Which for preproduction code?
>
> Options for each: Master / Feature / Develop

**A: Production code = Master; Preproduction code = Develop.**
In GitFlow, `master`/`main` always reflects production-ready code. `develop` is the integration branch holding completed features for the next release — it is preproduction. Feature branches are short-lived work-in-progress branches that merge into develop; they are neither production nor preproduction in the GitFlow definition.

---

**Q11 — Recover a deleted and committed file (DRAG DROP)**
> You use Git for source control. You delete a file, commit the changes, and continue to work. You need to recover the deleted file. Which three commands should you run in sequence?
> Available: `git restore path/to/file` / `git log` / `git commit -m 'undeleted the file'` / `git checkout [hash]-1 -- path/to/file` / `git stash` / `git tag`

**A (in order):**
1. `git log` — find the SHA of the deletion commit
2. `git checkout [hash]-1 -- path/to/file` — restore the file from the commit before deletion; stages it
3. `git commit -m 'undeleted the file'` — commit the recovery

`git restore path/to/file` without `--source` restores from HEAD (the deleted state) — wrong. `git stash` and `git tag` are irrelevant to file recovery.

---

**Q10 — Git authentication: supports Git + minimizes credential prompts**
> You have an Azure DevOps organization. You need to recommend an authentication mechanism that supports authentication from Git and minimizes the need to provide credentials during authentication. What should you recommend?
>
> A. Managed identities in Azure AD
> B. Personal access tokens (PATs) in Azure DevOps
> C. User accounts in Azure AD
> D. Alternate credentials in Azure DevOps

**A: B — Personal access tokens (PATs).**
PATs authenticate over HTTPS Git and are stored by the Git Credential Manager after first entry — no repeated credential prompts. Managed identities (A) cannot be used for Git operations. Azure AD user accounts (C) may trigger MFA/re-authentication repeatedly. Alternate credentials (D) are deprecated in Azure DevOps and require re-entering username/password each time.

---

**Q9 — PR merge strategy: reduce history volume**
> Your company uses Azure DevOps and a Git repository. You need to implement a pull request strategy that reduces the history volume in the master branch. Solution: implement a pull request strategy that uses a three-way merge. Does this meet the goal?
>
> A. Yes  B. No

**A: B — No.**
A three-way merge (standard merge commit) **preserves all individual commits** from the feature branch and **adds** an additional merge commit — it increases history volume. To **reduce** history volume, use **squash merge**, which collapses all PR commits into a single commit on the master branch.

| Strategy | Effect on history volume |
|---|---|
| Three-way merge (merge commit) | ❌ Increases — keeps all commits + adds merge commit |
| Squash merge | ✅ Reduces — all commits become one |
| Rebase and fast-forward | Neutral — no extra merge commit, but keeps individual commits |

---

**Q13 — Optimize documentation for Git versioning (HOTSPOT)**
> Your company uses GitHub. Process documentation is saved as Word (.docx) files containing simple flow charts stored as .bmp files. You need to optimize integration and versioning to: store as plain text, minimize number of files, simplify modification/merging/reuse of both charts and documents.
> - Convert .docx files to: [LaTex / **Markdown** / PDF]
> - Convert flow charts to: [**Mermaid graph diagrams (.md)** / PNG / TIFF]

**A:**
- **.docx → Markdown (.md):** Plain text, Git-diffable/mergeable, simple syntax.
- **.bmp → Mermaid graph diagrams (.md):** Diagram-as-code embedded inside the .md file — plain text, no separate file needed (minimizes file count), fully diffable/mergeable in Git.

PNG and TIFF are binary image files requiring separate files and cannot be meaningfully diffed. PDF is binary. LaTeX is plain text but complex.

---

**Q14 — Branch policy bypass: least privilege**
> You have a branch policy requiring code to build successfully before merging to master. You need to ensure a specific user can always merge to master even if the code fails to compile. The solution must use the principle of least privilege. What should you do?
> A. From the Security setting of the repository, modify the access control for the user.
> B. From the Security settings of the branch, modify the access control for the user.
> C. Add the user to the Build Administrators group.
> D. Add the user to the Project Administrators group.

**A: B — From the Security settings of the branch, modify the access control for the user.**
Grant the user the **"Bypass policies when completing pull requests"** permission at the **branch level** (Repos → Branches → … → Security). This applies only to the master branch — the narrowest possible scope, satisfying least privilege. Repository-level security applies to all branches (too broad). Build Administrators manage build pipeline definitions — not branch policy bypasses. Project Administrators have full project control — a severe violation of least privilege.

---

**Q17 — GitHub PR to Azure Boards work item linking via AB#**
> You use GitHub for source control and Azure Boards for project management. You plan to create a pull request in GitHub. You need to automatically link the request to an existing Azure Boards work item using the text AB#<WORKITEMNUMBER>. To which two elements can you add the text?
> A. Milestone  B. Comment  C. Title  D. Description  E. Label

**A: C and D — Title and Description.**
The `AB#<WORKITEMNUMBER>` syntax automatically links a GitHub PR to an Azure Boards work item when placed in the PR **title** or **description (body)**. It also works in commit messages. Comments, milestones, and labels are not parsed for AB# references.

---

**Q21 — Azure Boards + GitHub: commit message to link multiple work items and set one to done**
> You manage a project using Azure Boards and code using GitHub. You have work items 456, 457, and 458. You need to create a PR linked to all work items. The solution must set the state of work item 456 to done. What should you add to the commit message?
>
> A. `Fixes #456, #457, #458`
> B. `Fixes #AB456, #AB457, #AB458`
> C. `#456, #457, #458 / Completed #456`
> D. `#AB456, #AB457, #AB458`

**A: D — `#AB456, #AB457, #AB458`.**
The exam uses `#AB<ID>` format for Azure Boards work item references in GitHub commit messages. This syntax links all three work items to the PR. Options A and C use `#<ID>` which is GitHub issues syntax — not Azure Boards. Option B uses `Fixes` which would attempt to transition all three items.

> **Exam tip:** In exam questions using GitHub + Azure Boards commit message syntax, the format `#AB<ID>` (e.g., `#AB456`) is used to reference Azure Boards work items. Note: official Microsoft docs show `AB#<ID>` — be aware both formats may appear in exam scenarios.

---

**Q16 — DRAG DROP: Clone only src/web directory using Scalar**
> You have an Azure Repos repository named repo1. You need to clone repo1. The solution must clone only a directory named src/web. Complete the script:
> ```
> cd repos
> scalar clone  [Box 1]
> cd            [Box 2]
> git sparse-checkout set  [Box 3]
> ```
> Values: https://dev.azure.com/organization/project/_git/repo1 / git@ssh.dev.azure.com:v3/organization/project/repo1 / repo1/src / src/web / repo1/src/web / web

**A:**
- **Box 1: `git@ssh.dev.azure.com:v3/organization/project/repo1`** — Scalar uses the SSH URL format for Azure Repos
- **Box 2: `repo1/src/web`** — after cloning, `scalar clone` leaves you in `repos/`; navigate from `repos/` into `repos/repo1/src/web`
- **Box 3: `src/web`** — set sparse checkout to `src/web` relative to the repo root (`git` commands work from any subdirectory within the repo)

The HTTPS URL is used with plain `git clone`; SSH URL (`git@ssh.dev.azure.com:v3/...`) is used with `scalar clone`.
> Note: Dump answer used `cd src/web` and `sparse-checkout set repo1/src` — both are incorrect. `repos/src/web` doesn't exist after cloning, and `repo1/src` is the repo name not an internal path.

---

**Q15 — DRAG DROP: Clone large repo minimizing transferred data**
> You have a large repository named Repo1 that contains a directory named directory1. You plan to modify files in directory1. You need to create a clone of Repo1. The solution must minimize the amount of transferred data. Complete the script:
> ```
> cd repos
> [Box 1]  https://dev.azure.com/organisation/_git/Repo1
> [Box 2]  set directory1
> ```
> Values: git clone / git fetch / git sparse-checkout / git worktree / scalar clone / scalar run

**A:**
- **Box 1: `git clone`** — clones Repo1 from Azure DevOps
- **Box 2: `git sparse-checkout`** — configures the working tree to only check out `directory1`, minimizing transferred and local data

`git fetch` downloads refs but does not create a working clone. `git worktree` creates additional working trees from an existing repo. `scalar clone`/`scalar run` are Scalar-specific commands (different syntax). `git sparse-checkout set <dir>` is the correct subcommand to limit checkout scope.

---

**Q18 — DRAG DROP: git tag command to apply release labels**
> You are finalizing a release in GitHub. You need to apply the following labels to the release: Name, Email, Release v3.0, Release date. How should you complete the `git` command?
> `git [Box1] [Box2] v3.0 [Box3] "Release v3.0"`
> Options: add / commit / push / tag | -a / -b / -c / -m

**A:**
- **Box 1: `tag`** — creates a tag on the current commit
- **Box 2: `-a`** — annotated tag; stores name, email, and date automatically from git config
- **Box 3: `-m`** — inline message flag

`git tag -a v3.0 -m "Release v3.0"` — annotated tags are required to capture the Name, Email, and Release date metadata. A lightweight tag (`git tag v3.0`) stores none of that.

---

**Q19 — Remove large file accidentally committed to repository**
> You manage source code control and versioning using GitHub. A large file was committed to a repository accidentally. You need to reduce the size of the repository. The solution must remove the file from the repository. What should you use?
>
> A. bfg  B. lfs  C. gvfs  D. init

**A: A — BFG Repo Cleaner.**
BFG Repo Cleaner rewrites git history to remove large or unwanted files from all commits. It is fast and purpose-built for this task. `git filter-repo` is the other valid tool. **lfs** (Git Large File Storage) tracks large files in future commits but does not remove existing history. **gvfs/Scalar** optimizes access to large repos but does not remove files from history. **init** initializes a new repo — irrelevant.

---

**Q20 — DRAG DROP: Azure Repos policy types for PR requirements**
> You create a Git repository named Repo1 in Azure Repos. You need to configure it to meet: (1) Work items must be linked to a pull request. (2) Pull requests must complete a code review by a third-party tool (minimize admin effort). (3) Pull requests must have a minimum of two reviewers.
>
> Policy types: Branch / Build / Check-in / Status

**A:**
- **Work items linked to PR → Branch** — "Require linked work items" is a built-in Branch policy
- **Third-party code review → Status** — Status policies allow external services to post a pass/fail result to the PR; specifically designed for third-party integrations; minimizes admin effort (automated)
- **Minimum two reviewers → Branch** — "Require a minimum number of reviewers" is a built-in Branch policy

> **Exam tip:** Build policy = runs an Azure Pipelines build (first-party). Status policy = external/third-party service posts a status result via API (SonarCloud, custom tools, etc.). Both "work items" and "reviewer count" are Branch policy settings.

---

## Quick Reference: Exam Checklist

### Source Control Strategy (10–15%)
- [ ] Know all branching strategies: trunk-based, feature branch, GitFlow, GitHub Flow, fork
- [ ] Configure branch policies in Azure Repos (required reviewers, build validation, merge types)
- [ ] Configure branch protections in GitHub
- [ ] Understand PR workflow and merge strategies (merge commit, squash, rebase)
- [ ] Know Git LFS and Scalar use cases
- [ ] Know how to recover data: `git reflog`, `git cherry-pick`, `git revert`
- [ ] Know how to remove data: `git filter-repo`, BFG Repo Cleaner
- [ ] Configure repository permissions in Azure Repos and GitHub
- [ ] Know Git tag types (lightweight vs annotated) and release management

### Processes & Communications (10–15%)
- [ ] Know DORA metrics: lead time, cycle time, MTTR, change failure rate
- [ ] Link Azure Boards to GitHub (`AB#` syntax in commits/PRs)
- [ ] Configure wikis (Azure DevOps) and Mermaid diagrams
- [ ] Automate release notes via GitHub releases
- [ ] Configure webhooks and service hooks
- [ ] Understand feedback cycle design (inner vs outer loops)
- [ ] Source, bug, and quality traceability through work items

### Technical Debt & Code Quality
- [ ] Identify technical debt types and prioritization
- [ ] Know SonarQube/SonarCloud and key metrics (cyclomatic complexity, code coverage, duplication)
- [ ] Know GitHub Advanced Security: CodeQL, secret scanning, Dependabot
- [ ] Plan effective code reviews
- [ ] Integrate code quality tools into pipelines

### Git Hooks
- [ ] Know client-side hooks: `pre-commit`, `commit-msg`, `pre-push`
- [ ] Know server-side hooks: `pre-receive`, `post-receive`
- [ ] Know how to share hooks across a team
- [ ] Enforce commit message format with `commit-msg` hook
- [ ] Use `pre-commit` to prevent secret commits

### Inner Source
- [ ] Define inner source and its benefits
- [ ] Fork workflow vs branch workflow — when to use each
- [ ] How to sync a fork with upstream
- [ ] Fork workflow in Azure DevOps

---

*Sources: [MS Learn AZ-400 Path](https://learn.microsoft.com/en-us/training/paths/az-400-work-git-for-enterprise-devops/) | [AZ-400 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-400) | [AZ-400 Exam Page](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-400/)*
