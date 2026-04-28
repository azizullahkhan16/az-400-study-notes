# AZ-400 Study Notes: Manage Infrastructure as Code Using Azure

**Learning Path:** Manage infrastructure as code using Azure and DSC
**Exam Weight:** Part of "Implement and manage infrastructure" domain (~10–15% of exam)
**Modules Covered:** 6

---

## Table of Contents

1. [Explore Infrastructure as Code and Configuration Management](#1-explore-infrastructure-as-code-and-configuration-management)
2. [Create Azure Resources using ARM Templates](#2-create-azure-resources-using-arm-templates)
3. [Create Azure Resources Using Azure CLI](#3-create-azure-resources-using-azure-cli)
4. [Explore Azure Automation with DevOps](#4-explore-azure-automation-with-devops)
5. [Implement Desired State Configuration (DSC)](#5-implement-desired-state-configuration-dsc)
6. [Implement Bicep](#6-implement-bicep)
7. [Additional Exam Topics](#7-additional-exam-topics)
8. [Past Exam Scenario Bank](#8-past-exam-scenario-bank)
9. [Quick Reference Checklist](#9-quick-reference-checklist)

---

## 1. Explore Infrastructure as Code and Configuration Management

### What is Infrastructure as Code (IaC)?

IaC is the process of managing and provisioning infrastructure through **machine-readable definition files** rather than manual processes.

**Key principle:** Infrastructure is described in code files that can be versioned, tested, and reused.

### Imperative vs Declarative

| Approach | Description | Example Tools |
|---|---|---|
| **Imperative** | You specify *how* to achieve the desired state (step-by-step commands) | Azure CLI, Azure PowerShell |
| **Declarative** | You specify *what* the desired state should be; the tool figures out how | ARM Templates, Bicep, Terraform |

> **Exam tip:** ARM templates and Bicep are **declarative**. Azure CLI commands are **imperative**. The exam frequently tests this distinction.

### Idempotency

**Idempotency** means running the same operation multiple times produces the same result — no unintended side effects from repeated runs.

- Declarative tools (ARM, Bicep) are inherently idempotent
- Imperative scripts must be written carefully to be idempotent

### Configuration Management vs Provisioning

| Concept | Description | Tools |
|---|---|---|
| **Provisioning** | Creating infrastructure (VMs, networks, storage) | ARM, Bicep, Terraform |
| **Configuration Management** | Ensuring software/settings on provisioned VMs are in desired state | DSC, Ansible, Chef, Puppet |

### Configuration Drift

**Configuration drift** = when the actual state of infrastructure diverges from the desired state due to manual changes, patches, or other untracked modifications.

**Solutions:**
- Azure Automation State Configuration (DSC)
- Azure Policy (detect/deny non-compliant resources)
- Azure Automanage Machine Configuration

**Series exam pattern — "prevent project configuration from changing over time":**

| Solution proposed | Meets the goal? | Why |
|---|---|---|
| Azure Automation State Configuration (DSC) | **Yes** | Continuously enforces desired state on VMs |
| Azure Policy | **Yes** | Detects and denies non-compliant resource configurations |
| Add a code coverage step to build pipelines | **No** | Code coverage measures test coverage — unrelated to config governance |

> **Exam tip:** Code coverage, automated security testing, and CI triggers are **build/test concepts** — they do NOT prevent configuration drift. Only IaC enforcement tools (DSC, Azure Policy) address configuration governance.

### Mutable vs Immutable Infrastructure

| Type | Description | Example |
|---|---|---|
| **Mutable** | Infrastructure is updated in place | Patching a running VM |
| **Immutable** | Infrastructure is replaced entirely on change | Container deployments, VM image replacement |

> **Exam tip:** Immutable infrastructure reduces configuration drift since you never modify running instances.

---

## 2. Create Azure Resources using ARM Templates

### ARM Template Structure

ARM (Azure Resource Manager) templates are JSON files with the following sections:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "variables": {},
  "functions": [],
  "resources": [],
  "outputs": {}
}
```

| Section | Required | Description |
|---|---|---|
| `$schema` | Yes | JSON schema URL for the template |
| `contentVersion` | Yes | Template version for tracking |
| `parameters` | No | Values passed in at deployment time |
| `variables` | No | Reusable values derived from parameters |
| `functions` | No | Custom user-defined functions |
| `resources` | Yes | Azure resources to deploy |
| `outputs` | No | Values returned after deployment |

### Parameters

```json
"parameters": {
  "storageAccountName": {
    "type": "string",
    "defaultValue": "mystorage",
    "minLength": 3,
    "maxLength": 24,
    "metadata": {
      "description": "Name of the storage account"
    }
  }
}
```

**Parameter types:** `string`, `int`, `bool`, `object`, `array`, `secureString`, `secureObject`

> **Exam tip:** Use `secureString` for passwords and secrets — values are NOT logged in deployment history.

### Variables

```json
"variables": {
  "storageAccountName": "[concat(parameters('prefix'), 'storage')]"
}
```

Variables cannot use `reference()` function — only `parameters()` and other variables.

### Key Template Functions

| Function | Purpose | Example |
|---|---|---|
| `concat()` | String/array concatenation | `concat('prefix', '-', 'suffix')` |
| `resourceId()` | Get resource ID | `resourceId('Microsoft.Storage/storageAccounts', 'myStorage')` |
| `reference()` | Get runtime property of a resource | `reference(resourceId(...)).primaryEndpoints.blob` |
| `parameters()` | Access parameter value | `parameters('location')` |
| `variables()` | Access variable value | `variables('storageAccountName')` |
| `uniqueString()` | Generate deterministic hash | `uniqueString(resourceGroup().id)` |
| `resourceGroup()` | Get current RG info | `resourceGroup().location` |
| `subscription()` | Get subscription info | `subscription().subscriptionId` |

> **Exam tip:** `reference()` cannot be used in `variables` section. It can only be used in `resources` and `outputs`.

### Resource Dependencies

Use `dependsOn` to control deployment order:

```json
"resources": [
  {
    "type": "Microsoft.Network/virtualNetworks",
    "name": "myVNet",
    ...
  },
  {
    "type": "Microsoft.Network/networkInterfaces",
    "name": "myNIC",
    "dependsOn": ["[resourceId('Microsoft.Network/virtualNetworks', 'myVNet')]"],
    ...
  }
]
```

Alternatively, use implicit dependencies via `reference()` — ARM detects the dependency automatically.

### Deployment Modes

| Mode | Behavior | Default? |
|---|---|---|
| **Incremental** | Adds/updates resources in template; leaves existing resources NOT in template untouched | YES (default) |
| **Complete** | Adds/updates resources in template; **DELETES** existing resources NOT in template | No |

> **Exam trap:** Complete mode is dangerous — it deletes resources not in the template. Always run **what-if** before complete mode.
> **Exam tip (HOTSPOT tested):** "Prevent modification/deletion of existing resources in the resource group" → set **Deployment mode = Incremental**. Complete mode would delete those resources. This is the setting to change in the ARM template deployment task in the release pipeline.

**Important rules:**
- Default mode is **Incremental**
- Only root-level templates support Complete mode
- **Linked and nested templates must use Incremental mode**
- In Incremental mode, if a resource already exists, ALL properties are reapplied (not just changed ones)

**Specify deployment mode:**
```bash
az deployment group create \
  --resource-group myRG \
  --template-file main.json \
  --mode Complete
```

### Linked Templates

Linked templates allow you to split large templates into smaller, reusable components. A **main template** references one or more linked templates via URI.

```json
{
  "type": "Microsoft.Resources/deployments",
  "name": "linkedTemplate",
  "properties": {
    "mode": "Incremental",
    "templateLink": {
      "uri": "https://mystorageaccount.blob.core.windows.net/templates/storage.json"
    },
    "parameters": {
      "storageAccountName": { "value": "[parameters('storageAccountName')]" }
    }
  }
}
```

**`templateLink` vs `template` vs Key Vault reference — exam distinction:**

| Property | Used when | Has `uri`? |
|---|---|---|
| **`templateLink`** | External template referenced by URI (linked template) | ✅ Yes |
| **`template`** | Inline template embedded directly in the resource | ❌ No URI |

> **Exam tip (HOTSPOT tested):** When an ARM deployment resource has `"uri"` pointing to an external file → the property is `"templateLink"`. Resource type for nested/linked template deployments is always `"Microsoft.Resources/deployments"`.

**Dynamic Key Vault resource ID using `resourceId()`:**
```json
"id": "[resourceId(parameters('vaultSubscription'),
       parameters('vaultResourceGroupName'),
       'Microsoft.KeyVault/vaults',
       parameters('vaultName'))]"
```
This generates the Key Vault resource ID dynamically at deployment time using subscription, resource group, type, and name parameters — supports cross-subscription Key Vault references.

**Cross-resource-group deployments with linked templates:**
- A linked template can deploy resources into a **different resource group** than the main template using the `resourceGroup` property on the deployment resource.
- This is the recommended pattern when you need to deploy to **multiple resource groups** in a single release pipeline run.
- Example scenario: main template → linked template A (deploys 4 VMs to RG1) + linked template B (deploys 2 SQL databases to RG2).

> **Exam tip:** Linked templates must be accessible via **URI** (publicly accessible or using SAS token). They **cannot** be referenced from local file system at deploy time.
>
> **Exam tip (DRAG DROP — GitHub ARM template deployment sequence):** To deploy an ARM template located on GitHub as part of a build/release process, the three-step sequence is:
> 1. **Create a release pipeline** — the deployment mechanism
> 2. **Add an Azure Resource Group Deployment task** — references the ARM template directly via GitHub URL
> 3. **Set the template parameters** — configure the task with the template location and parameter values
>
> "Create a package" is not needed (GitHub-hosted templates are referenced by URL, not packaged). "Create a job agent" is not a required step.
>
> **Exam tip (tested — series):** "Deploy to two resource groups (VMs in one, SQL databases in other) using ARM templates" — the exam tests multiple proposed solutions:
>
> | Proposed solution | Meets the goal? | Why |
> |---|---|---|
> | Main template with **two linked templates**, each scoped to its resource group | **Yes** | Linked templates support cross-resource-group deployment natively in one orchestrated run |
> | **Two standalone templates**, each deploying to its own resource group | **No** | Standalone templates are independent deployments — no single orchestrated pipeline run; cannot coordinate cross-resource-group deployment in one step |

### Nested Templates

Inline templates within the parent template:

```json
{
  "type": "Microsoft.Resources/deployments",
  "name": "nestedTemplate",
  "properties": {
    "mode": "Incremental",
    "template": {
      "$schema": "...",
      "contentVersion": "1.0.0.0",
      "resources": [...]
    }
  }
}
```

### Key Vault Integration with ARM Templates

Reference Key Vault secrets in ARM template parameters:

```json
"adminPassword": {
  "reference": {
    "keyVault": {
      "id": "/subscriptions/.../resourceGroups/.../providers/Microsoft.KeyVault/vaults/myVault"
    },
    "secretName": "adminPassword"
  }
}
```

> **Exam tip:** This is done in the **parameters file** (`.parameters.json`), NOT in the template itself. Key Vault must have `enabledForTemplateDeployment: true`.

### What-If Operation

Preview changes before deployment:

```bash
az deployment group what-if \
  --resource-group myRG \
  --template-file main.json
```

Output symbols:
- `+` = will be created
- `~` = will be modified
- `-` = will be deleted
- `=` = no change

---

## 3. Create Azure Resources Using Azure CLI

### Azure CLI Basics

Azure CLI is an **imperative**, cross-platform command-line tool for managing Azure resources.

**Authentication:**
```bash
az login                          # Interactive browser login
az login --service-principal      # Service principal login
az account set --subscription <id>  # Set active subscription
```

### Core CLI Pattern

```bash
az <resource-type> <action> --<param> <value>
```

Examples:
```bash
# Create resource group
az group create --name myRG --location eastus

# Create storage account
az storage account create \
  --name mystorageacct \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS

# Deploy ARM template
az deployment group create \
  --resource-group myRG \
  --template-file main.json \
  --parameters @params.json
```

### Output Formats

```bash
az vm list --output table    # Human-readable table
az vm list --output json     # Full JSON
az vm list --output tsv      # Tab-separated (good for scripting)
az vm list --output yaml     # YAML format
```

### Querying with JMESPath

```bash
az vm list --query "[].{Name:name, RG:resourceGroup}" --output table
az storage account show --name myacct --query "primaryEndpoints.blob"
```

### Azure CLI vs Azure PowerShell

| Feature | Azure CLI | Azure PowerShell |
|---|---|---|
| Platform | Cross-platform (Linux, macOS, Windows) | Windows (also cross-platform via PS Core) |
| Approach | Imperative bash-style | Imperative cmdlet-style |
| Scripting | Bash, batch scripts | PowerShell scripts |
| Output | Text, JSON, table, TSv | PowerShell objects |

### Deploying Templates with Azure CLI

```bash
# Validate template (syntax only)
az deployment group validate \
  --resource-group myRG \
  --template-file main.json

# What-if preview
az deployment group what-if \
  --resource-group myRG \
  --template-file main.json

# Deploy
az deployment group create \
  --resource-group myRG \
  --template-file main.json \
  --parameters storageAccountName=myaccount location=eastus
```

### Subscription-level vs Resource Group-level Deployments

| Scope | CLI command |
|---|---|
| Resource group | `az deployment group create` |
| Subscription | `az deployment sub create` |
| Management group | `az deployment mg create` |
| Tenant | `az deployment tenant create` |

---

## 4. Explore Azure Automation with DevOps

### What is Azure Automation?

Azure Automation provides a cloud-based automation and configuration management platform supporting:
- **Process Automation** — runbooks to automate repetitive tasks
- **Configuration Management** — DSC to enforce desired state
- **Update Management** — OS patch management
- **Shared capabilities** — credentials, certificates, connections, schedules

### Runbooks

A **runbook** is an automated script (procedure) in Azure Automation.

**Runbook types:**

| Type | Language | Best For |
|---|---|---|
| **PowerShell** | PowerShell | Most Windows/Azure tasks |
| **PowerShell Workflow** | PowerShell Workflow | Long-running, parallel tasks with checkpoints |
| **Python** | Python 2/3 | Cross-platform, data processing |
| **Graphical** | Visual (drag-and-drop) | Simple workflows without scripting |
| **Graphical PowerShell Workflow** | Visual workflow | Complex parallelism visually |

> **Exam tip:** Only **PowerShell Workflow** and **Graphical PowerShell Workflow** runbooks support **checkpoints** and **parallel execution blocks**.

### Checkpoints in PowerShell Workflow Runbooks

```powershell
Checkpoint-Workflow
```

Saves the state of the workflow so if it's suspended (e.g., Azure maintenance), it resumes from the last checkpoint — not from the beginning.

> **Exam tip:** Regular PowerShell runbooks do NOT support checkpoints. Only PowerShell **Workflow** runbooks do.

### Runbook Execution

**Ways to trigger a runbook:**
1. **Manually** — from Azure portal or PowerShell
2. **Schedule** — run at specific times
3. **Webhook** — HTTP POST triggers execution
4. **Azure Alerts** — respond to monitoring alerts
5. **Event Grid** — event-driven triggers

### Webhooks

A **webhook** provides a URL that external systems can POST to in order to trigger a runbook.

- URL contains a security token (cannot be retrieved after creation — save it)
- Can pass parameters via the request body
- Expires after a configurable time (default: 1 year max)

```powershell
# Access webhook data in runbook
$WebhookData = Get-AutomationVariable -Name "WebhookData"
$RequestBody = $WebhookData.RequestBody | ConvertFrom-Json
```

> **Exam tip:** The webhook URL is only shown **once** at creation. If lost, you must create a new webhook.

### Hybrid Runbook Worker

By default, runbooks run in Azure sandbox. A **Hybrid Runbook Worker** runs runbooks on on-premises machines or other clouds.

Use cases:
- Access local databases
- Manage on-premises servers
- Run scripts requiring local network access

**Setup:** Install the Hybrid Worker agent on the target machine and register it with the Automation account.

### Azure Automation Shared Resources

| Resource | Purpose |
|---|---|
| **Credentials** | Store username/password securely |
| **Certificates** | Store certificates for authentication |
| **Connections** | Store connection info (e.g., Azure subscription) |
| **Variables** | Store shared values (encrypted or plain) |
| **Schedules** | Define when runbooks run |
| **Modules** | PowerShell modules available to runbooks |

### Update Management

- Assesses update compliance for Windows and Linux VMs
- Schedules deployment of required updates
- Works across Azure VMs, on-premises, and other clouds
- Uses Log Analytics workspace for data

---

## 5. Implement Desired State Configuration (DSC)

### What is DSC?

**Desired State Configuration (DSC)** is a PowerShell-based management platform that enables you to define the desired state of your systems and ensure they remain in that state.

**Components:**
- **Configuration** — PowerShell script defining desired state
- **Resources** — modules that implement state (e.g., File, Registry, Service, WindowsFeature)
- **LCM (Local Configuration Manager)** — engine on each node that applies/monitors configurations

### DSC Configuration Example

```powershell
Configuration WebServerConfig {
  Import-DscResource -ModuleName PSDesiredStateConfiguration

  Node "WebServer01" {
    WindowsFeature IIS {
      Ensure = "Present"
      Name   = "Web-Server"
    }

    File WebRoot {
      Ensure          = "Present"
      Type            = "Directory"
      DestinationPath = "C:\inetpub\wwwroot"
    }
  }
}

# Compile to MOF file
WebServerConfig -OutputPath "C:\DSC\Configs"
```

Compiling a configuration creates a **MOF (Managed Object Format)** file — the machine-readable form of the configuration.

### Push vs Pull Mode

| Mode | Description | Use Case |
|---|---|---|
| **Push** | Administrator pushes configuration directly to node using `Start-DscConfiguration` | Small environments, immediate application |
| **Pull** | Nodes periodically check a pull server for their assigned configuration | Large-scale, automated environments |

> **Exam tip:** Azure Automation State Configuration acts as a **pull server**. Nodes check in at a configured frequency (default: 15 minutes).

### LCM (Local Configuration Manager) Modes

The LCM controls how DSC configurations are applied:

| ConfigurationMode | Behavior |
|---|---|
| `ApplyOnly` | Apply once; do NOT monitor or auto-correct drift |
| `ApplyAndMonitor` | Apply and monitor; report drift but do NOT auto-correct |
| `ApplyAndAutocorrect` | Apply and auto-correct any drift detected |

> **Exam tip:** To **automatically fix** configuration drift, set ConfigurationMode to `ApplyAndAutocorrect`.

### Azure Automation State Configuration

Azure Automation State Configuration is a cloud-based DSC pull server service.

**Key capabilities:**
- Import and compile DSC configurations
- Register Azure VMs, on-premises machines, and other cloud VMs as managed nodes
- Assign node configurations to nodes
- View compliance dashboard in Azure portal
- Send compliance data to Azure Monitor Logs

**Onboarding a VM to Azure Automation DSC — 5-Step Sequence (Exam Favorite DRAG DROP):**

| Step | Exam Action Wording | Details |
|------|---------------------|---------|
| 1 | **Upload a configuration to Azure Automation State Configuration** | Import the DSC `.ps1` script into the Automation Account |
| 2 | **Compile a configuration into a node configuration** | Compile the script into a `.mof` file (machine-readable desired state) |
| 3 | **Onboard the virtual machines to Azure Automation State Configuration** | Register VMs as managed nodes (`Register-AzureRmAutomationDscNode`) |
| 4 | **Assign the node configuration** | Map the compiled MOF to the registered nodes |
| 5 | **Check the compliance status of the node** | Verify nodes report as Compliant in the Azure portal |

**Eliminated distractors:** "Create a management group" (organizes subscriptions — unrelated to DSC) and "Assign tags to the virtual machines" (tagging is not a DSC workflow step).

> **Exam tip:** The compilation step (configuration → node configuration) is a **separate step** from uploading. You must compile before assigning to nodes. The technically correct sequence is: **Upload → Compile → Onboard → Assign → Check compliance**. Note: the exam question explicitly states "more than one order is correct" — any sequence containing all five correct steps receives full credit.

### Linux DSC

DSC is also available for Linux via the **nx** DSC resource module:
- `nxFile` — manage files/directories
- `nxService` — manage services (systemd/init)
- `nxPackage` — manage packages (apt, yum)
- `nxUser` — manage user accounts

### DSC vs Azure Policy vs ARM Templates

| Tool | Use Case |
|---|---|
| **DSC** | OS-level and software configuration on VMs |
| **Azure Policy** | Enforce/audit Azure resource compliance |
| **ARM/Bicep** | Provision Azure resources |

---

## 6. Implement Bicep

### What is Bicep?

Bicep is a **domain-specific language (DSL)** for declaratively deploying Azure resources. It transpiles to ARM JSON templates.

**Advantages over ARM JSON:**
- Simpler, cleaner syntax (no JSON boilerplate)
- Better modularity with modules
- Type safety with decorators
- First-class tooling in VS Code

> **Exam tip:** Bicep has **feature parity** with ARM templates — anything you can do in ARM JSON, you can do in Bicep.

### Bicep File Structure

```bicep
// Parameters
param location string = resourceGroup().location
param storageAccountName string

// Variables
var storageSkuName = 'Standard_LRS'

// Resources
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSkuName
  }
  kind: 'StorageV2'        // ← resource kind (e.g. StorageV2, BlobStorage)
  properties: {            // ← resource-specific settings
    supportsHttpsTrafficOnly: true
  }
}

// Outputs
output storageAccountId string = storageAccount.id
```

### Bicep Top-Level Keywords (Exam Favorite — HOTSPOT)

Understanding when to use each keyword is critical for HOTSPOT questions:

| Keyword | Purpose | Example |
|---|---|---|
| `param` | Declares an input parameter (value supplied at deploy time) | `param storageAccount string` |
| `var` | Declares a variable (computed within the template) | `var name = '${param}${uniqueString(...)}` |
| `resource` | Declares an Azure resource to deploy | `resource sa 'Microsoft.Storage/...' = { ... }` |
| `kind` | A **property inside a resource block** — specifies the resource kind/type variant | `kind: 'StorageV2'` |
| `properties` | A **property inside a resource block** — contains resource-specific configuration settings | `properties: { supportsHttpsTrafficOnly: true }` |
| `type` | Used in user-defined types or object type declarations | `type myType = { name: string }` |
| `module` | Calls another Bicep file as a reusable module | `module vm './vm.bicep' = { ... }` |

> **Exam tip (tested — HOTSPOT):** In a Bicep storage account resource block, after `sku: { name: 'Standard_GRS' }`:
> - The next top-level property for the resource variant is `kind:` (e.g., `'StorageV2'`)
> - The block containing resource-specific settings (like `supportsHttpsTrafficOnly`) uses `properties:`
> - `param`, `var`, and `type` are top-level file declarations, NOT valid inside a resource body.

### Parameters with Decorators

```bicep
@minLength(3)
@maxLength(24)
@description('Name of the storage account')
param storageAccountName string

@secure()
param adminPassword string

@allowed(['eastus', 'westus', 'northeurope'])
param location string = 'eastus'
```

**Common decorators:**

| Decorator | Purpose |
|---|---|
| `@description()` | Document the parameter |
| `@secure()` | Treat as sensitive (not logged) |
| `@minLength()` / `@maxLength()` | String/array length constraints |
| `@minValue()` / `@maxValue()` | Integer constraints |
| `@allowed()` | Restrict to allowed values |

### Variables and Expressions

```bicep
var suffix = uniqueString(resourceGroup().id)
var storageAccountName = 'storage${suffix}'
var isProduction = (environment == 'prod')
```

### Resource References (Symbolic Names)

```bicep
resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: 'myVNet'
  location: location
  properties: { ... }
}

resource subnet 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = {
  parent: vnet       // Implicit dependency — no dependsOn needed
  name: 'default'
  properties: { ... }
}
```

> **Exam tip:** Using `parent` property creates an **implicit dependency** — Bicep automatically determines deployment order.

### Modules

Modules allow you to split Bicep code into reusable files:

```bicep
// main.bicep
module storage './modules/storage.bicep' = {
  name: 'storageDeployment'
  params: {
    location: location
    storageAccountName: storageAccountName
  }
}

// Access module output
output storageEndpoint string = storage.outputs.blobEndpoint
```

> **Exam tip:** Modules enable **encapsulation** — each module is a self-contained template. They are the Bicep equivalent of linked templates.

### Conditions

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = if (deployStorage) {
  name: storageAccountName
  ...
}
```

### Loops

```bicep
// Deploy multiple resources
resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for i in range(0, 3): {
  name: 'storage${i}'
  location: location
  ...
}]

// Loop over array parameter
param environments array = ['dev', 'staging', 'prod']
resource rgs 'Microsoft.Resources/resourceGroups@2022-09-01' = [for env in environments: {
  name: 'rg-${env}'
  location: location
}]
```

### Bicep Deployment Commands

```bash
# Deploy Bicep file
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters storageAccountName=myaccount

# What-if preview
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep

# Decompile ARM JSON to Bicep
az bicep decompile --file main.json

# Build Bicep to ARM JSON (transpile)
az bicep build --file main.bicep
```

### Deploy Bicep from Azure Pipelines

```yaml
# Azure Pipelines — deploy Bicep
- task: AzureCLI@2
  displayName: 'Deploy Bicep'
  inputs:
    azureSubscription: 'MyServiceConnection'
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az deployment group create \
        --resource-group $(resourceGroup) \
        --template-file infrastructure/main.bicep \
        --parameters environmentName=$(environment)
```

Alternatively, use the `AzureResourceManagerTemplateDeployment@3` task which supports Bicep directly.

### Deploy Bicep from GitHub Actions

```yaml
- name: Deploy Bicep
  uses: azure/arm-deploy@v1
  with:
    subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    resourceGroupName: myRG
    template: ./infrastructure/main.bicep
    parameters: environmentName=dev
```

### Bicep vs ARM JSON Comparison

| Feature | ARM JSON | Bicep |
|---|---|---|
| Syntax | Verbose JSON | Clean DSL |
| Modules | Linked/nested templates | Native `module` keyword |
| Loops | `copy` element | `for` expression |
| Conditions | `condition` property | `if` expression |
| Dependencies | Explicit `dependsOn` | Implicit via symbolic names |
| Transpilation | Native | Transpiles to ARM JSON |
| Decompile | N/A | `az bicep decompile` |
| Parity | Full | Full (same capabilities) |

### Parameter Files

```bicep
// main.bicepparam (Bicep parameter file)
using 'main.bicep'

param storageAccountName = 'myStorage'
param location = 'eastus'
```

Or traditional JSON format:
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": { "value": "myStorage" }
  }
}
```

---

## 7. Additional Exam Topics

### Terraform on Azure

While not an Azure-native tool, Terraform is a common exam topic:

- **Provider:** `hashicorp/azurerm`
- **State file:** `terraform.tfstate` — tracks current infrastructure state
- **Core commands:** `terraform init` → `terraform plan` → `terraform apply` → `terraform destroy`
- **Remote state:** Store state in Azure Blob Storage for team environments
- **Workspaces:** Manage multiple environments with same config

> **Exam tip:** Terraform state file stores sensitive data. Store in a **secure remote backend** (Azure Storage with encryption, access control).

**Terraform Azure Development Toolchain — Yeoman + Terratest:**

| Framework | Purpose | Role in Terraform Azure workflow |
|---|---|---|
| **Yeoman** | Scaffolding tool | `yo az-terra-module` generator creates the initial Terraform module file structure for Azure (main.tf, variables.tf, outputs.tf, test folder) |
| **Terratest** | Go-based testing framework | Writes automated tests for Terraform configurations; deploys real infrastructure, validates it, then destroys it |

> **Exam tip:** When asked which frameworks are needed to deploy an Azure resource group with Terraform, the answer is **Yeoman** (scaffold the module structure) + **Terratest** (test the Terraform code). Vault = secrets management (not a deployment framework); Node.js/Tiller = unrelated.

### ARM Template Deployment Scopes

| Scope | Template Schema | CLI Command |
|---|---|---|
| Resource Group | `deploymentTemplate.json` | `az deployment group create` |
| Subscription | `subscriptionDeploymentTemplate.json` | `az deployment sub create` |
| Management Group | `managementGroupDeploymentTemplate.json` | `az deployment mg create` |
| Tenant | `tenantDeploymentTemplate.json` | `az deployment tenant create` |

### ARM Template Export

- Export from Azure portal: Resource Group → Export template
- Exported templates may need cleanup — they include all current property values
- Use as starting point, not production-ready templates

### Azure Resource Manager Deployment History

- Each deployment to a resource group is recorded in deployment history
- Max **800 deployments** per resource group — oldest are auto-deleted
- Use `az deployment group list` to view history
- Use `az deployment group show` to inspect a specific deployment

### Bicep Linter

The Bicep linter runs automatically in VS Code and during `az bicep build`:
- Detects best practice violations
- Warns about unused parameters, variables, outputs
- Configurable via `bicepconfig.json`

### Azure Deployment Stacks

A newer feature (exam may reference):
- Group multiple ARM/Bicep deployments as a **single manageable unit**
- Apply **deny assignments** to prevent modification of stack resources
- Clean up by deleting the stack (optionally deletes managed resources)

```bash
az stack group create \
  --name myStack \
  --resource-group myRG \
  --template-file main.bicep \
  --deny-settings-mode none
```

### DSC vs Guest Configuration (Automanage)

| Feature | DSC (Azure Automation) | Azure Automanage Machine Configuration |
|---|---|---|
| Agent | LCM (Windows/Linux) | Guest Configuration extension |
| Policies | DSC configurations | Azure Policy |
| Scope | Azure + on-premises + other clouds | Azure VMs primarily |
| Reporting | Automation Account dashboard | Azure Policy compliance |

> **Exam tip:** Azure Automanage Machine Configuration (formerly Guest Configuration) is the **newer/recommended** approach for audit-and-enforce scenarios at scale.

### ARM Template Best Practices

| Practice | Recommendation |
|---|---|
| Parameter naming | Use camelCase |
| Secrets | Always use `secureString` type or Key Vault reference |
| Deployment mode | Default to Incremental; use Complete only when needed |
| What-if | Always run before Complete mode deployments |
| Linked templates | Store in version-controlled, accessible location |
| Template validation | Use `az deployment group validate` before deploying |

### Key Vault Integration Patterns in IaC

**Option 1: Static reference in parameter file** (Key Vault ID + secret name — resolved at deploy time)
**Option 2: Dynamic reference using `reference()` in template** (advanced, less common)
**Option 3: Bicep `getSecret()` function:**

```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-02-01' existing = {
  name: keyVaultName
}

module vm './vm.bicep' = {
  name: 'vmDeployment'
  params: {
    adminPassword: kv.getSecret('adminPassword')
  }
}
```

> **Exam tip:** `getSecret()` in Bicep is equivalent to the static Key Vault reference in ARM JSON parameter files — it retrieves the secret at deployment time.

### Ansible vs DSC vs Chef/Puppet

| Tool | Type | Language | Agentless? |
|---|---|---|---|
| **Ansible** | Configuration Mgmt | YAML (Playbooks) | Yes (SSH) |
| **DSC** | Configuration Mgmt | PowerShell | No (LCM agent) |
| **Chef** | Configuration Mgmt | Ruby (Recipes) | No (Chef Client) |
| **Puppet** | Configuration Mgmt | Puppet DSL (Manifests) | No (Puppet Agent) |
| **Terraform** | Provisioning | HCL | Yes (API-based) |

---

## 8. Past Exam Scenario Bank

**Q1.** Your team manages 50 VMs with IIS installed. After deploying a new configuration, some VMs are found with IIS disabled. The DSC LCM is configured with `ApplyOnly`. What change fixes the issue?

**A:** Change `ConfigurationMode` from `ApplyOnly` to `ApplyAndAutocorrect`. ApplyOnly applies the configuration once but does not monitor or correct drift. ApplyAndAutocorrect continuously checks and re-applies the desired state.

---

**Q2.** You deploy an ARM template to a resource group in Complete mode. The resource group previously had 3 storage accounts, but the template only defines 1. How many storage accounts remain after deployment?

**A:** 1. Complete mode deletes any resources in the resource group that are NOT defined in the template. The 2 storage accounts not in the template are deleted.

---

**Q3.** You want to reference a Key Vault secret in your ARM template without hardcoding the secret value. Where do you specify the Key Vault reference?

**A:** In the **parameters file** (`.parameters.json`), not the template itself. The Key Vault must have `enabledForTemplateDeployment` set to `true`. The reference uses the Key Vault resource ID and the secret name.

---

**Q4.** A runbook needs to continue from where it left off if the Azure job is interrupted. Which runbook type should you use?

**A:** **PowerShell Workflow** runbook. It supports `Checkpoint-Workflow` to save state. Regular PowerShell runbooks restart from the beginning if interrupted.

---

**Q5.** You need to trigger an Azure Automation runbook from an external third-party system via HTTP. What should you use?

**A:** A **Webhook**. A webhook provides a URL that external systems can HTTP POST to in order to trigger the runbook. The webhook URL must be saved at creation time — it cannot be retrieved later.

---

**Q6.** Which ARM template deployment mode is the default when you do not specify a mode?

**A:** **Incremental**. Resources in the template are added/updated; resources already in the resource group but not in the template are left unchanged.

---

**Q7.** You have a Bicep module that deploys a virtual network. How do you reference the output of this module in the parent template?

**A:** Use the module's symbolic name followed by `.outputs.<outputName>`:
```bicep
module vnet './modules/vnet.bicep' = { ... }
output vnetId string = vnet.outputs.vnetId
```

---

**Q8.** Your team uses Azure Automation State Configuration. After registering a VM, the compliance dashboard shows the node as "Not Compliant." Which step did you likely miss?

**A:** The DSC configuration was imported but **not compiled** into a node configuration (MOF). You must compile the configuration before assigning it to a node. Configuration → Compile → Assign to node → Monitor compliance.

---

**Q9.** You need to preview what changes an ARM template deployment will make without actually deploying. What command do you use?

**A:**
```bash
az deployment group what-if --resource-group myRG --template-file main.json
```
The what-if operation shows resources that will be created (`+`), modified (`~`), deleted (`-`), or unchanged (`=`).

---

**Q10.** You are authoring a Bicep file. You want to ensure an `adminPassword` parameter value is never logged or shown in deployment outputs. What decorator do you use?

**A:** `@secure()`. Parameters decorated with `@secure()` are treated as sensitive — Azure Resource Manager does not log or display their values in deployment history.

---

**Q11.** A linked template must be deployed with which mode?

**A:** **Incremental**. Only the **root-level template** supports Complete mode. Linked and nested templates must always use Incremental mode.

---

**Q12.** You need to run automation runbooks against on-premises servers that are behind a firewall. What Azure Automation feature enables this?

**A:** **Hybrid Runbook Worker**. Installing the Hybrid Worker agent on the on-premises machine allows runbooks to execute locally, with access to local resources and network.

---

**Q13.** Which of the following correctly describes the difference between imperative and declarative IaC?

**A:**
- **Imperative** (e.g., Azure CLI): You specify the steps/commands to achieve a state. Order matters. Not inherently idempotent.
- **Declarative** (e.g., ARM/Bicep): You specify the desired end state. The tool determines how to achieve it. Inherently idempotent.

---

**Q14.** You export an ARM template from the Azure portal. A colleague says the template is ready to use in production pipelines. Is this correct?

**A:** No. Exported templates capture the current state of resources and often include unnecessary properties, hardcoded values, and missing dependencies. They should be used as a **starting point** and cleaned up before use in production.

---

**Q15.** In Bicep, how do you create an implicit dependency between two resources without using `dependsOn`?

**A:** Use the **symbolic name** of one resource in the definition of another (either via `parent` property or by referencing the resource's properties). Bicep detects these references and automatically infers the deployment order.

---

**Q16.** Your organization wants to prevent any changes to resources deployed as part of a specific deployment group. Which feature should you use?

**Q18. (HOTSPOT)** You need to create a storage account using a Bicep file. Complete the file:
```
param storageAccount string
var storageAccountNameToUse = '${storageAccount}${uniqueString(resourceGroup().id)}'
resource invoiceStorage 'Microsoft.Storage/storageAccounts@2022-05-01' = {
  name: storageAccountNameToUse
  location: 'eastus'
  sku: { name: 'Standard_GRS' }
  [BOX1]: 'StorageV2'
  [BOX2]: {
    supportsHttpsTrafficOnly: true
  }
}
```
Options for each box: kind / param / properties / type / var

**A: BOX1 = `kind`; BOX2 = `properties`.**
`kind` is the resource body property that specifies the storage account variant (StorageV2, BlobStorage, etc.). `properties` is the resource body block for resource-specific configuration settings. `param`, `var`, and `type` are top-level Bicep file declarations — they cannot appear inside a resource body.

---

**A:** **Azure Deployment Stacks** with deny assignments (`--deny-settings-mode`). This prevents modifications to stack-managed resources outside of the stack's deployment process.

---

**Q17.** You plan to create a release pipeline that deploys Azure resources using ARM templates. The pipeline will create: 2 resource groups, 4 VMs in one resource group, and 2 SQL databases in the other. Solution: create a main template with two linked templates, each deploying resources to its respective resource group. Does this meet the goal?

A. Yes  B. No

**A: A — Yes.**
A main ARM template with two linked templates is the correct pattern for deploying to multiple resource groups. Each linked template is scoped to its own resource group using the `resourceGroup` property on the deployment resource. Linked templates support cross-resource-group deployments, allowing one pipeline run to create both resource groups and deploy all resources in a single orchestrated deployment.

---

**Q23.** You plan to create a release pipeline that will deploy Azure resources using ARM templates. The pipeline will create: 2 resource groups, 4 VMs in one resource group, and 2 SQL databases in the other. Solution: Create two standalone templates, each of which will deploy the resources in its respective group. Does this meet the goal?

A. Yes  B. No

**A: B — No.**
Two standalone templates are two independent deployments — they are not orchestrated as a single unit. The correct pattern for deploying to multiple resource groups in one coordinated pipeline run is a **main template with two linked templates**, each scoped to its respective resource group. Standalone templates cannot natively coordinate cross-resource-group deployments in one step.

---

**Q19. (HOTSPOT)** You have an Azure Data Factory (Contoso Data Factory) in resource group Contoso Dev, and a release pipeline (Pipeline1) that deploys it to ContosoRG. The ARM template deployment task is configured. Answer:
- The setting to change to prevent modification of existing databases/web apps in ContosoRG: [Action / Template location / **Deployment mode** / Deployment scope]
- Pipeline1 retrieves the ARM template from: [CI build output / **linked artifact** / default Git branch of Data Factory]

**A:**
- **Deployment mode** — change to **Incremental** to leave existing resources (databases, web apps) in ContosoRG untouched. Complete mode would delete them.
- **Linked artifact** — the release pipeline retrieves the ARM template from the linked build artifact (CI build output linked to the release pipeline).

---

**Q20. (HOTSPOT)** You have an ARM template that deploys resources and references secrets in Azure Key Vault. You need to dynamically generate the Key Vault resource ID during deployment. Complete the template:

```json
"type": [dropdown],
"properties": {
    "mode": "Incremental",
    [dropdown]: {
        "uri": "[uri(parameters('_artifactsLocation'), ...)]"
    }
}
```

Options for type: Microsoft.KeyVault/vaults / Microsoft.Resources/deployments / Microsoft.Subscription/subscriptions
Options for second dropdown: deployment / template / templateLink

**A:**
- **type: `"Microsoft.Resources/deployments"`** — nested/linked template deployments always use this resource type.
- **second dropdown: `"templateLink"`** — used when the template is external (referenced by URI). `"template"` is for inline templates. The `uri` property confirms this is an external linked template.

---

**Q21.** You plan to use Terraform to deploy an Azure resource group. You need to install the required frameworks to support the planned deployment. Which two frameworks should you install?

A. Vault  B. Terratest  C. Node.js  D. Yeoman  E. Tiller

**A:** B and D — **Terratest** + **Yeoman**.
- **Yeoman** (`yo az-terra-module`): scaffolds the Terraform module structure for Azure (main.tf, variables.tf, outputs.tf, test folder).
- **Terratest**: Go-based testing framework that deploys real infrastructure from Terraform code, validates it, then destroys it.
- Vault = secrets management (unrelated to Terraform deployment scaffolding); Node.js/Tiller = unrelated.

---

**Q25.** DRAG DROP — As part of your application build process, you need to deploy a group of resources to Azure by using an ARM template located on GitHub. Which three actions should you perform in sequence?
> Actions: Create a package / Add an Azure Resource Group Deployment task / Create a job agent / Create a release pipeline / Set the template parameters

**A (in order):**
1. Create a release pipeline
2. Add an Azure Resource Group Deployment task
3. Set the template parameters

The Azure Resource Group Deployment task can reference an ARM template directly via GitHub URL — no packaging needed. "Create a job agent" is not a required step in this workflow.

---

**Q24.** DRAG DROP — You need to use Azure Automation State Configuration to manage the ongoing consistency of virtual machine configurations. Which five actions should you perform in sequence?
> Available: Onboard the VMs to Azure Automation State Configuration / Check the compliance status of the node / Create a management group / Assign the node configuration / Compile a configuration into a node configuration / Upload a configuration to Azure Automation State Configuration / Assign tags to the virtual machines

**A (in order):**
1. Upload a configuration to Azure Automation State Configuration
2. Compile a configuration into a node configuration
3. Onboard the virtual machines to Azure Automation State Configuration
4. Assign the node configuration
5. Check the compliance status of the node

"Create a management group" and "Assign tags to the virtual machines" are not part of the DSC workflow.

---

**Q26.** You add virtual machines as managed nodes in Azure Automation State Configuration. You need to configure the computers in Group7. What should you do?

A. Run the Register-AzureRmAutomationDscNode Azure PowerShell cmdlet  B. Modify the ConfigurationMode property of the LCM  C. Install PowerShell Core  D. Modify the RefreshMode property of the LCM

**A: A — Run Register-AzureRmAutomationDscNode.**
`Register-AzureRmAutomationDscNode` is the cmdlet that **registers an Azure VM as a DSC node** in an Azure Automation account, connecting it to State Configuration so it receives and applies DSC configurations. This is the required registration step. **ConfigurationMode** controls how config is applied (ApplyOnly / ApplyAndMonitor / ApplyAndAutoCorrect) — not registration. **PowerShell Core** is irrelevant; DSC uses Windows PowerShell. **RefreshMode** controls push vs pull delivery mode — not registration.

> **Exam tip — DSC series:** This question and Q104 (same scenario) form a series:
> - Register nodes → `Register-AzureRmAutomationDscNode`
> - Fix nodes not auto-correcting drift → change `ConfigurationMode` to `ApplyAndAutoCorrect`

---

**Q27. (DRAG DROP)** You need to configure Azure Automation for the computers in Pool7. Which three actions should you perform in sequence?

> Actions: Run New-AzureRmResourceGroupDeployment / Create an ARM template file (.json) / Run Import-AzureRmAutomationDscConfiguration / Run Start-AzureRmAutomationDscCompilationJob / Create a DSC configuration file (.ps1)

**A (in order):**
1. **Create a DSC configuration file (.ps1)** — write the PowerShell DSC script defining the desired state
2. **Run Import-AzureRmAutomationDscConfiguration** — imports the .ps1 configuration into the Azure Automation account
3. **Run Start-AzureRmAutomationDscCompilationJob** — compiles the configuration into a node configuration (MOF file) ready to be assigned to nodes

`New-AzureRmResourceGroupDeployment` deploys ARM templates — unrelated to DSC. ARM template (.json) is for infrastructure provisioning, not DSC configuration scripts.

> **Exam tip — DSC setup sequence:** .ps1 → Import → Compile → (Assign node config) → (Check compliance). Always: write script first, import second, compile third.

---

**Q22.** You manage a project in Azure DevOps. You need to prevent the configuration of the project from changing over time. Solution: Add a code coverage step to the build pipelines. Does this meet the goal?

A. Yes  B. No

**A: B — No.**
Code coverage measures what percentage of the code is exercised by tests — it is a testing quality metric. It has no effect on configuration governance or drift. To prevent configuration from changing over time, use tools that enforce desired state: **Azure Automation DSC** (`ApplyAndAutocorrect` mode), **Azure Policy**, or **Azure Blueprints**. Adding a build step cannot stop manual or drift-based changes to infrastructure configuration.

---

## 9. Quick Reference Checklist

### IaC Approach Selection
- [ ] Provisioning Azure resources → ARM Templates or Bicep
- [ ] OS/software configuration on VMs → DSC or Ansible
- [ ] Enforce resource compliance → Azure Policy
- [ ] Automate operational tasks → Azure Automation Runbooks
- [ ] Cross-cloud / third-party → Terraform

### ARM Template Key Facts
- [ ] 6 sections: `$schema`, `contentVersion`, `parameters`, `variables`, `functions`, `resources`, `outputs`
- [ ] Default deployment mode = **Incremental**
- [ ] Complete mode = deletes resources NOT in template
- [ ] Linked/nested templates must use **Incremental** mode
- [ ] `reference()` cannot be used in `variables` section
- [ ] Key Vault reference goes in **parameters file**, requires `enabledForTemplateDeployment: true`
- [ ] Max 800 deployments per resource group in history

### Bicep Key Facts
- [ ] `@secure()` → sensitive parameter, not logged
- [ ] `parent:` → implicit dependency, replaces `dependsOn`
- [ ] `module` keyword → reusable components (like linked templates)
- [ ] `kv.getSecret()` → retrieve Key Vault secret at deploy time
- [ ] `az bicep decompile` → ARM JSON → Bicep
- [ ] `az bicep build` → Bicep → ARM JSON

### Azure Automation Key Facts
- [ ] Runbook types: PowerShell, PowerShell Workflow, Python, Graphical, Graphical Workflow
- [ ] Only **PowerShell Workflow** supports `Checkpoint-Workflow`
- [ ] Webhook URL only shown **once** at creation
- [ ] **Hybrid Runbook Worker** = run runbooks on-premises
- [ ] Shared resources: Credentials, Certificates, Connections, Variables, Schedules, Modules

### DSC Key Facts
- [ ] Push mode → `Start-DscConfiguration` (admin pushes to node)
- [ ] Pull mode → node pulls from pull server (Azure Automation acts as pull server)
- [ ] LCM ConfigurationMode: `ApplyOnly` | `ApplyAndMonitor` | `ApplyAndAutocorrect`
- [ ] `ApplyAndAutocorrect` → automatically fixes configuration drift
- [ ] Must **compile** configuration before assigning to nodes
- [ ] Configuration → MOF (Managed Object Format) file

### Deployment Mode Decision Table
| Scenario | Mode |
|---|---|
| Add new resources, keep existing | Incremental |
| Replace entire resource group state | Complete |
| Linked/nested templates | Incremental (always) |
| Safe default | Incremental |
| After running what-if and confirming | Complete (if required) |
