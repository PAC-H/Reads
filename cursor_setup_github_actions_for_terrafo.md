# Setup GitHub Actions for Terraform with Azure
_Exported on 04/06/2025 at 16:33:12 GMT+12 from Cursor (0.50.7)_

---

**User**

can you help me init some github actions terraform plan and deployment templates? I need to use oidc connection with azure for authentication. there should be separated pipeline which I can use to run some functions like destroy resources securely, or checking the existing resources. The terraform state file will be store in an azure storage account container. and I want a config file to manage those global config easily.  

---

**Cursor**

[MODE: RESEARCH]

I need to understand your current setup and requirements before proceeding. Let me explore your workspace and ask some clarifying questions.

The workspace appears to be empty. Let me ask some clarifying questions to better understand your requirements:

1. **Azure Setup**: Do you already have an Azure subscription set up with the necessary service principal and OIDC configuration, or do we need to include instructions for setting that up?

2. **Terraform Structure**: Do you have preferences for how the Terraform code should be organized? (e.g., modules structure, environment separation)

3. **Pipeline Functions**: You mentioned separate pipelines for different functions. What specific functions do you need beyond plan, apply, and destroy? (e.g., drift detection, resource validation, compliance checks)

4. **Azure Storage**: Do you already have the Azure storage account and container for the Terraform state, or should the templates include setup for that as well?

5. **Environment Management**: How many environments do you plan to manage? (dev, staging, prod, etc.)

6. **Terraform Scope**: What types of Azure resources will you primarily be managing with these templates?

7. **Security Requirements**: Are there any specific security or compliance requirements for the pipelines? (e.g., approval gates, specific RBAC roles)

8. **Configuration Format**: For the global config file, do you prefer YAML, JSON, or another format?

Could you provide details on these aspects so I can better understand what needs to be built?

---

**User**

1. I already have the azure subscription and oidc config set up correctly
2. There will be a terraform folder in the root folder, it should has a module subfolder. two environment staging and production will be set up for terraform infrastrcture
3. beyond plan, apply and destroy, I need the drift detection, terraform init and state management, lint and format check, dependency graphing, target operations, tfupdate. and also manual approval in github actions, and save plan output.
4. I already have a storage account and container setup, just need to include in the terraform config file to include the backend info
5. I need staging and prod environments
6. I need approval gates for deployment from staging to prod. and manual approval is require to run the staging deployment
7. for the config file, please use json format

---

**Cursor**

[MODE: RESEARCH]

Thank you for the detailed information. I have a few more clarifying questions to ensure I understand the complete requirements:

1. **OIDC Configuration**: What are the specific Azure tenant ID, subscription ID, and client ID values you'll be using? (Or should these be parameterized in the config file?)

2. **Storage Backend**: What are the storage account name, container name, and resource group for the Terraform state backend?

3. **Terraform Versions**: Do you have a preference for Terraform version and Azure provider version to pin in the configuration?

4. **Repository Structure**: Should the GitHub Actions workflows be in `.github/workflows/` and what naming convention do you prefer for the workflow files?

5. **Environment Promotion**: When you mention approval gates from staging to prod, do you want:
   - A single workflow that handles both environments with approval gates between them?
   - Separate workflows where prod deployment is triggered after staging success?
   - Manual promotion process where someone manually triggers prod after staging validation?

6. **Target Operations**: For the target operations functionality, what specific targeting scenarios do you need? (e.g., target specific resources, modules, or resource types)

7. **Plan Output Storage**: Where should the plan outputs be saved? As GitHub Actions artifacts, or in a specific location?

8. **Drift Detection Schedule**: How frequently should drift detection run? (daily, weekly, on-demand only)

9. **Branch Strategy**: What branch protection and deployment strategy do you use? (e.g., main branch for prod, develop for staging, feature branches, etc.)

10. **Notifications**: Do you need notifications for pipeline results? (Slack, email, Teams, etc.)

Understanding these details will help me create the most appropriate templates for your specific setup.

---

**User**

1. these azure params should be specified in the config file
2. please leave these blank in the file and I will put in later, please make some comment to show this needs to be filled
3. use the latest version by default
4. yes, those gihub actions pipeline should be put into ./github/workflows. for the naming conventions of the pipeline, please use what you think would be appropriate|
5. a single workflow handles both env with approval gates between them
6. the target will be specific resources
7. the plan output should be saved as github actions artifacts
8. the drift detection should be run weekly
9. please use gitflow branch strategy
10. the notification will be send to Teams 

---

**Cursor**

[MODE: RESEARCH]

Perfect! I now have a comprehensive understanding of your requirements:

**Summary of Requirements:**
- Azure OIDC authentication with parameters in JSON config file
- Terraform structure: `/terraform` folder with modules subfolder
- Two environments: staging and production
- Storage backend configuration (to be filled in later)
- Latest Terraform and Azure provider versions
- GitHub Actions workflows in `.github/workflows/`
- Single workflow with approval gates (staging → manual approval → prod)
- Additional pipelines for: drift detection (weekly), init/state management, linting/formatting, dependency graphing, target operations, tfupdate
- Plan outputs saved as GitHub Actions artifacts
- Gitflow branch strategy (main=prod, develop=staging)
- Teams notifications
- Manual approval required for staging deployments
- Target operations for specific resources

**Key Components Needed:**
1. JSON configuration file for global settings
2. Terraform backend configuration with Azure storage
3. Main deployment workflow (staging → prod with approvals)
4. Drift detection workflow (weekly schedule)
5. Utility workflows (init, lint/format, dependency graph, tfupdate, target operations)
6. Terraform module structure for staging/prod environments
7. Teams notification integration

I have all the information needed to proceed. Would you like me to move to the next phase where I can discuss different approaches and architectural considerations for implementing these templates?

Please respond with **"ENTER INNOVATE MODE"** when you're ready for me to brainstorm the implementation approaches.

---

**User**

ENTER INNOVATE MODE

---

**Cursor**

[MODE: INNOVATE]

Here are several architectural approaches and considerations for implementing your GitHub Actions Terraform templates:

**Configuration Management Approaches:**
1. **Single Global Config**: One JSON file containing all environments and shared settings
   - Advantages: Centralized management, easy to maintain consistency
   - Considerations: Potential security concerns mixing staging/prod configs

2. **Environment-Specific Configs**: Separate JSON files per environment with a base config
   - Advantages: Better separation of concerns, enhanced security
   - Considerations: More files to maintain, potential drift between environments

**Workflow Architecture Options:**
1. **Monolithic Workflow**: Single large workflow handling all operations (plan/apply/destroy)
   - Advantages: Centralized logic, easier to manage state between steps
   - Considerations: Could become complex, harder to troubleshoot individual operations

2. **Modular Workflows**: Separate workflows that can call reusable actions
   - Advantages: Better maintainability, reusable components, clearer separation
   - Considerations: Need to manage state passing between workflows

**Authentication Strategy Possibilities:**
1. **Workflow-Level OIDC**: Configure OIDC at the workflow level
   - Advantages: Simpler setup, consistent across all jobs
   - Considerations: Less granular control

2. **Job-Level OIDC**: Configure OIDC per job with different permissions
   - Advantages: Principle of least privilege, more secure
   - Considerations: More complex configuration

**State Management Considerations:**
1. **State Locking Strategy**: How to handle concurrent operations
2. **State Backup**: Automated backup mechanisms for state files
3. **State Encryption**: Additional encryption layers beyond Azure storage

**Approval Gate Implementation Options:**
1. **GitHub Environments**: Use built-in environment protection rules
   - Advantages: Native GitHub feature, built-in approval workflows
   - Considerations: Requires repository settings configuration

2. **Manual Dispatch**: Use workflow_dispatch with manual triggers
   - Advantages: More control over approval process
   - Considerations: Less integrated with GitHub's native features

**Notification Integration Possibilities:**
1. **Webhook-based**: Direct Teams webhook integration
2. **Action-based**: Use existing GitHub Actions for Teams notifications
3. **Conditional**: Only notify on failures vs. all operations

Would you like me to explore any of these approaches in more detail, or do you have preferences for specific architectural directions?

---

**User**

1. please take the env-specific configs approach
2. please use modular workflows that contains reusable actions
3. please use workflow-level oidc  authentication
4. use the default locking provided by azure storage account, but please also add a template workflow to unblock the state file. please added a backup workflow that runs every week to back up the state file to a backup storage account container. no need for adding the encryption for the state file.
5. for the approval gate, use the built-in env protection rules
6. for notification, use direct teams webhook integration
7. additional considerations: the build agent will be self-hosted build agents, this will need to be specified in the config file. RBAC: can you provide some suggestions for managing the pipelines' RBAC?
8. Do you have other suggestions for buidling those pipelines?

---

**Cursor**

[MODE: INNOVATE]

Excellent choices! Based on your selections, here are additional suggestions and considerations for building robust pipelines:

**RBAC Management Suggestions:**
1. **Service Principal Roles**: 
   - Staging: Contributor role scoped to staging resource groups only
   - Production: Custom role with minimal required permissions, separate approval for destructive operations
   - Backup operations: Storage Blob Data Reader/Writer for state backup

2. **GitHub Repository Permissions**:
   - Separate teams for staging vs production deployments
   - Read-only access for developers, write access for DevOps team
   - Branch protection rules aligned with RBAC

3. **Environment-Specific Access**:
   - Staging: Broader access for faster iteration
   - Production: Restricted access with mandatory reviews and approvals

**Self-Hosted Agent Considerations:**
1. **Agent Pool Strategy**: Separate agent pools for staging/production for isolation
2. **Security Hardening**: Agent security baseline, network isolation, credential management
3. **Scaling**: Auto-scaling capabilities during peak deployment times

**Additional Pipeline Enhancements:**
1. **Error Handling & Retry Logic**: 
   - Automatic retry for transient failures
   - Graceful degradation strategies
   - Clear error reporting with actionable messages

2. **Pipeline Optimization**:
   - Terraform plan caching between jobs
   - Parallel execution where possible (e.g., linting while planning)
   - Smart triggering (only run when relevant files change)

3. **Audit & Compliance**:
   - Deployment audit logs stored persistently
   - Compliance scanning (e.g., Azure Policy validation)
   - Change tracking and rollback capabilities

4. **Testing & Validation**:
   - Pre-deployment validation workflows
   - Post-deployment smoke tests
   - Infrastructure testing (e.g., Terratest integration)

5. **Monitoring & Observability**:
   - Pipeline performance metrics
   - Resource drift monitoring alerts
   - Cost impact reporting on changes

**Security Enhancements:**
1. **Secret Management**: Separation of secrets by environment, rotation policies
2. **Network Security**: VPN/private networking for self-hosted agents
3. **Least Privilege**: Fine-grained permissions for each pipeline operation

Would any of these additional capabilities be valuable for your use case, or should I proceed with the core requirements you've specified?

---

**User**

based on the suggestions, 
1. please do some terraform plan caching between jobs, there will be some provider that will be install multiple times
2. please do the parallel execution when possible and appropriate
3. please set the smart triggering using the way you think when it is appropriate
4. please include the compliance scanning template to check the azure policy validation
5. for the testing, there will mainly be infrastrcuture test, please make a terratest entry template pipeline, we don't do any testing now but we can create the pipeline first, in the pipeline we can specify the example terratest we want to run. please help me create a simple example of that


---

**Cursor**

[MODE: INNOVATE]

Perfect! Those are excellent enhancements that will significantly improve the pipeline efficiency and reliability:

**Enhanced Requirements Summary:**
1. **Performance Optimizations**:
   - Terraform plan caching to avoid provider reinstallation
   - Parallel execution for independent operations (lint + plan, multi-environment validation)
   - Smart triggering based on path changes (terraform/, .github/workflows/, config files)

2. **Quality & Compliance**:
   - Azure Policy compliance scanning integration
   - Terratest infrastructure testing pipeline template
   - Simple Terratest example for demonstration

3. **Pipeline Intelligence**:
   - Conditional execution based on file changes
   - Optimized job dependencies and parallelization
   - Efficient artifact sharing between jobs

**Architecture Considerations:**
- Cache terraform providers and modules across jobs using GitHub Actions cache
- Use job matrices for parallel environment operations where appropriate
- Implement path-based triggers using GitHub's `paths` and `paths-ignore` filters
- Azure Policy scanning can run in parallel with terraform plan
- Terratest pipeline can be triggered manually or on specific paths
- Simple Go-based Terratest example testing basic Azure resource creation

**Pipeline Structure Overview:**
- Main deployment workflow with smart triggers and caching
- Utility workflows (drift detection, state management, compliance)
- Testing workflow with Terratest integration
- All using modular, reusable actions and environment-specific configs

I now have a complete understanding of all requirements and enhancements. The solution will include comprehensive templates covering deployment, testing, compliance, and operational workflows with performance optimizations and smart execution logic.

Are you ready for me to create a detailed implementation plan? Please respond with **"ENTER PLAN MODE"** when ready.

---

**User**

ENTER PLAN MODE and do the planning

---

**Cursor**

[MODE: PLAN]

## COMPREHENSIVE IMPLEMENTATION PLAN

### Project Structure
```
/
├── .github/
│   ├── workflows/
│   │   ├── terraform-main.yml
│   │   ├── terraform-drift-detection.yml
│   │   ├── terraform-state-management.yml
│   │   ├── terraform-utilities.yml
│   │   ├── terraform-compliance.yml
│   │   └── terraform-testing.yml
│   └── actions/
│       ├── setup-terraform/
│       │   └── action.yml
│       ├── azure-login/
│       │   └── action.yml
│       ├── teams-notification/
│       │   └── action.yml
│       └── terraform-cache/
│           └── action.yml
├── config/
│   ├── base.json
│   ├── staging.json
│   └── production.json
├── terraform/
│   ├── environments/
│   │   ├── staging/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── terraform.tf
│   │   └── production/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── terraform.tf
│   └── modules/
│       └── example/
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
├── tests/
│   ├── terratest/
│   │   ├── go.mod
│   │   ├── go.sum
│   │   └── basic_test.go
│   └── azure-policy/
│       └── compliance-rules.json
└── README.md
```

### Configuration Files Design

**base.json** - Shared configuration:
```json
{
  "azure": {
    "tenant_id": "# FILL: Your Azure Tenant ID",
    "subscription_id": "# FILL: Your Azure Subscription ID",
    "client_id": "# FILL: Your Azure Client ID for OIDC"
  },
  "terraform": {
    "version": "latest",
    "azure_provider_version": "~>3.0"
  },
  "github": {
    "self_hosted_agents": {
      "staging_pool": "# FILL: Your staging agent pool name",
      "production_pool": "# FILL: Your production agent pool name"
    }
  },
  "notifications": {
    "teams_webhook": "# FILL: Your Teams webhook URL"
  },
  "storage": {
    "backup_account": "# FILL: Backup storage account name",
    "backup_container": "# FILL: Backup container name",
    "backup_resource_group": "# FILL: Backup resource group name"
  }
}
```

**staging.json** - Staging specific:
```json
{
  "environment": "staging",
  "azure": {
    "subscription_id": "# FILL: Staging subscription ID (if different)"
  },
  "terraform": {
    "backend": {
      "storage_account_name": "# FILL: Staging state storage account",
      "container_name": "# FILL: Staging state container",
      "resource_group_name": "# FILL: Staging state resource group",
      "key": "staging.tfstate"
    }
  },
  "deployment": {
    "auto_apply": false,
    "require_approval": true,
    "max_parallel": 2
  }
}
```

**production.json** - Production specific:
```json
{
  "environment": "production",
  "azure": {
    "subscription_id": "# FILL: Production subscription ID (if different)"
  },
  "terraform": {
    "backend": {
      "storage_account_name": "# FILL: Production state storage account",
      "container_name": "# FILL: Production state container",
      "resource_group_name": "# FILL: Production state resource group",
      "key": "production.tfstate"
    }
  },
  "deployment": {
    "auto_apply": false,
    "require_approval": true,
    "max_parallel": 1
  }
}
```

### Reusable Actions Design

**setup-terraform/action.yml**:
- Inputs: terraform_version, working_directory, environment
- Outputs: terraform_version, cache_key
- Functions: Install Terraform, setup cache, configure backend

**azure-login/action.yml**:
- Inputs: tenant_id, subscription_id, client_id
- Functions: OIDC authentication with Azure

**teams-notification/action.yml**:
- Inputs: webhook_url, status, environment, message
- Functions: Send formatted Teams notifications

**terraform-cache/action.yml**:
- Inputs: environment, cache_key
- Functions: Cache/restore providers and modules

### Main Workflows Design

**terraform-main.yml** - Primary deployment workflow:
- Triggers: Push to develop/main, PR to main, manual dispatch
- Path filters: terraform/**, config/**, .github/workflows/**
- Jobs: 
  1. Validation (parallel: lint, format, plan)
  2. Staging deployment (manual approval required)
  3. Compliance scanning (parallel with staging)
  4. Production deployment (environment protection + approval)
- Features: Plan caching, artifact storage, Teams notifications

**terraform-drift-detection.yml** - Weekly drift detection:
- Triggers: Weekly schedule (Sundays), manual dispatch
- Jobs: Check drift for both environments, report differences

**terraform-state-management.yml** - State operations:
- Triggers: Manual dispatch only
- Jobs: Unlock state, backup state, restore state
- Security: Restricted permissions, approval gates

**terraform-utilities.yml** - Utility operations:
- Triggers: Manual dispatch, path changes in terraform/
- Jobs: tfupdate, dependency graph, targeted operations
- Features: Matrix strategy for multiple environments

**terraform-compliance.yml** - Azure Policy scanning:
- Triggers: PR events, manual dispatch
- Jobs: Policy validation, compliance reporting
- Integration: Azure Policy REST API calls

**terraform-testing.yml** - Terratest pipeline:
- Triggers: Manual dispatch, changes in tests/
- Jobs: Run Go-based infrastructure tests
- Features: Test result reporting, cleanup automation

### Security & RBAC Implementation

**GitHub Environment Protection Rules**:
- Staging environment: Manual approval required
- Production environment: Manual approval + branch restrictions
- Separate reviewer groups for each environment

**Azure RBAC Strategy**:
- Staging: Contributor role on staging resource groups
- Production: Custom role with restricted permissions
- State storage: Storage Blob Data Contributor
- Backup storage: Storage Blob Data Reader/Writer

### Performance Optimizations

**Caching Strategy**:
- Terraform providers cached by version + platform
- Terraform modules cached by content hash
- Plan files cached between validation and apply

**Parallel Execution**:
- Lint + Format + Compliance scanning in parallel
- Multi-environment drift detection using matrix
- Independent utility operations

**Smart Triggering**:
- Path-based filters for relevant changes only
- Skip unchanged environments
- Conditional job execution based on file changes

### Terratest Example Implementation

**go.mod**:
```go
module terraform-tests

go 1.21

require (
    github.com/gruntwork-io/terratest v0.46.0
    github.com/stretchr/testify v1.8.4
)
```

**basic_test.go** - Simple Azure resource group test:
```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformBasicExample(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../terraform/environments/staging",
        Vars: map[string]interface{}{
            "resource_group_name": "test-rg-terratest",
            "location": "East US",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Validate outputs
    resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")
    assert.Equal(t, "test-rg-terratest", resourceGroupName)
}
```

### Implementation Checklist Conversion

IMPLEMENTATION CHECKLIST:
1. Create base directory structure (terraform/, config/, tests/, .github/)
2. Create config/base.json with Azure OIDC parameters and global settings
3. Create config/staging.json with staging-specific backend configuration
4. Create config/production.json with production-specific backend configuration
5. Create .github/actions/setup-terraform/action.yml reusable action
6. Create .github/actions/azure-login/action.yml reusable action  
7. Create .github/actions/teams-notification/action.yml reusable action
8. Create .github/actions/terraform-cache/action.yml reusable action
9. Create terraform/environments/staging/main.tf with basic Azure provider setup
10. Create terraform/environments/staging/variables.tf with required variables
11. Create terraform/environments/staging/outputs.tf with basic outputs
12. Create terraform/environments/staging/terraform.tf with backend configuration
13. Create terraform/environments/production/main.tf with basic Azure provider setup
14. Create terraform/environments/production/variables.tf with required variables
15. Create terraform/environments/production/outputs.tf with basic outputs
16. Create terraform/environments/production/terraform.tf with backend configuration
17. Create terraform/modules/example/main.tf with sample Azure resource group module
18. Create terraform/modules/example/variables.tf with module variables
19. Create terraform/modules/example/outputs.tf with module outputs
20. Create .github/workflows/terraform-main.yml with main deployment pipeline
21. Create .github/workflows/terraform-drift-detection.yml with weekly drift detection
22. Create .github/workflows/terraform-state-management.yml with state operations
23. Create .github/workflows/terraform-utilities.yml with utility operations
24. Create .github/workflows/terraform-compliance.yml with Azure Policy scanning
25. Create .github/workflows/terraform-testing.yml with Terratest integration
26. Create tests/terratest/go.mod with Go module definition
27. Create tests/terratest/go.sum (will be generated after go mod tidy)
28. Create tests/terratest/basic_test.go with simple infrastructure test
29. Create tests/azure-policy/compliance-rules.json with policy definitions
30. Create README.md with setup instructions and usage documentation

```plaintext
/
├── .github/
│   ├── workflows/
│   │   ├── terraform-main.yml
│   │   ├── terraform-drift-detection.yml
│   │   ├── terraform-state-management.yml
│   │   ├── terraform-utilities.yml
│   │   ├── terraform-compliance.yml
│   │   └── terraform-testing.yml
│   └── actions/
│       ├── setup-terraform/
│       │   └── action.yml
│       ├── azure-login/
│       │   └── action.yml
│       ├── teams-notification/
│       │   └── action.yml
│       └── terraform-cache/
│           └── action.yml
├── config/
│   ├── base.json
│   ├── staging.json
│   └── production.json
├── terraform/
│   ├── environments/
│   │   ├── staging/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── terraform.tf
│   │   └── production/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── terraform.tf
│   └── modules/
│       └── example/
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
├── tests/
│   ├── terratest/
│   │   ├── go.mod
│   │   ├── go.sum
│   │   └── basic_test.go
│   └── azure-policy/
│       └── compliance-rules.json
└── README.md
```

```json
{
  "azure": {
    "tenant_id": "# FILL: Your Azure Tenant ID",
    "subscription_id": "# FILL: Your Azure Subscription ID",
    "client_id": "# FILL: Your Azure Client ID for OIDC"
  },
  "terraform": {
    "version": "latest",
    "azure_provider_version": "~>3.0"
  },
  "github": {
    "self_hosted_agents": {
      "staging_pool": "# FILL: Your staging agent pool name",
      "production_pool": "# FILL: Your production agent pool name"
    }
  },
  "notifications": {
    "teams_webhook": "# FILL: Your Teams webhook URL"
  },
  "storage": {
    "backup_account": "# FILL: Backup storage account name",
    "backup_container": "# FILL: Backup container name",
    "backup_resource_group": "# FILL: Backup resource group name"
  }
}
```

```json
{
  "environment": "staging",
  "azure": {
    "subscription_id": "# FILL: Staging subscription ID (if different)"
  },
  "terraform": {
    "backend": {
      "storage_account_name": "# FILL: Staging state storage account",
      "container_name": "# FILL: Staging state container",
      "resource_group_name": "# FILL: Staging state resource group",
      "key": "staging.tfstate"
    }
  },
  "deployment": {
    "auto_apply": false,
    "require_approval": true,
    "max_parallel": 2
  }
}
```

```json
{
  "environment": "production",
  "azure": {
    "subscription_id": "# FILL: Production subscription ID (if different)"
  },
  "terraform": {
    "backend": {
      "storage_account_name": "# FILL: Production state storage account",
      "container_name": "# FILL: Production state container",
      "resource_group_name": "# FILL: Production state resource group",
      "key": "production.tfstate"
    }
  },
  "deployment": {
    "auto_apply": false,
    "require_approval": true,
    "max_parallel": 1
  }
}
```

```go
module terraform-tests

go 1.21

require (
    github.com/gruntwork-io/terratest v0.46.0
    github.com/stretchr/testify v1.8.4
)
```

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformBasicExample(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../terraform/environments/staging",
        Vars: map[string]interface{}{
            "resource_group_name": "test-rg-terratest",
            "location": "East US",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Validate outputs
    resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")
    assert.Equal(t, "test-rg-terratest", resourceGroupName)
}
```

---

**User**

ENTER EXECUTE MODE and do the implementation

---

**Cursor**

[MODE: EXECUTE]

I'll implement the plan exactly as specified in the checklist. Starting with the directory structure and configuration files:

```json
{
  "azure": {
    "tenant_id": "# FILL: Your Azure Tenant ID",
    "subscription_id": "# FILL: Your Azure Subscription ID",
    "client_id": "# FILL: Your Azure Client ID for OIDC"
  },
  "terraform": {
    "version": "latest",
    "azure_provider_version": "~>3.0"
  },
  "github": {
    "self_hosted_agents": {
      "staging_pool": "# FILL: Your staging agent pool name",
      "production_pool": "# FILL: Your production agent pool name"
    }
  },
  "notifications": {
    "teams_webhook": "# FILL: Your Teams webhook URL"
  },
  "storage": {
    "backup_account": "# FILL: Backup storage account name",
    "backup_container": "# FILL: Backup container name",
    "backup_resource_group": "# FILL: Backup resource group name"
  }
}
```

```json
{
  "environment": "staging",
  "azure": {
    "subscription_id": "# FILL: Staging subscription ID (if different)"
  },
  "terraform": {
    "backend": {
      "storage_account_name": "# FILL: Staging state storage account",
      "container_name": "# FILL: Staging state container",
      "resource_group_name": "# FILL: Staging state resource group",
      "key": "staging.tfstate"
    }
  },
  "deployment": {
    "auto_apply": false,
    "require_approval": true,
    "max_parallel": 2
  }
}
```

```json
{
  "environment": "production",
  "azure": {
    "subscription_id": "# FILL: Production subscription ID (if different)"
  },
  "terraform": {
    "backend": {
      "storage_account_name": "# FILL: Production state storage account",
      "container_name": "# FILL: Production state container",
      "resource_group_name": "# FILL: Production state resource group",
      "key": "production.tfstate"
    }
  },
  "deployment": {
    "auto_apply": false,
    "require_approval": true,
    "max_parallel": 1
  }
}
```

```yaml
name: 'Setup Terraform'
description: 'Install Terraform, setup cache, configure backend'
inputs:
  terraform_version:
    description: 'Terraform version to install'
    required: false
    default: 'latest'
  working_directory:
    description: 'Working directory for Terraform'
    required: true
  environment:
    description: 'Environment name (staging/production)'
    required: true
outputs:
  terraform_version:
    description: 'Installed Terraform version'
    value: ${{ steps.setup.outputs.terraform_version }}
  cache_key:
    description: 'Cache key for providers'
    value: ${{ steps.cache.outputs.cache_key }}
runs:
  using: 'composite'
  steps:
    - name: Setup Terraform
      id: setup
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ inputs.terraform_version }}
        
    - name: Generate cache key
      id: cache
      shell: bash
      run: |
        echo "cache_key=terraform-${{ inputs.environment }}-${{ runner.os }}-${{ hashFiles('${{ inputs.working_directory }}/.terraform.lock.hcl') }}" >> $GITHUB_OUTPUT
        
    - name: Cache Terraform providers
      uses: actions/cache@v4
      with:
        path: |
          ${{ inputs.working_directory }}/.terraform
          ~/.terraform.d/plugin-cache
        key: ${{ steps.cache.outputs.cache_key }}
        restore-keys: |
          terraform-${{ inputs.environment }}-${{ runner.os }}-
          
    - name: Create plugin cache directory
      shell: bash
      run: |
        mkdir -p ~/.terraform.d/plugin-cache
        echo 'plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"' > ~/.terraformrc
```

```yaml
name: 'Azure Login'
description: 'OIDC authentication with Azure'
inputs:
  tenant_id:
    description: 'Azure Tenant ID'
    required: true
  subscription_id:
    description: 'Azure Subscription ID'
    required: true
  client_id:
    description: 'Azure Client ID for OIDC'
    required: true
runs:
  using: 'composite'
  steps:
    - name: Azure CLI login
      uses: azure/login@v1
      with:
        client-id: ${{ inputs.client_id }}
        tenant-id: ${{ inputs.tenant_id }}
        subscription-id: ${{ inputs.subscription_id }}
        
    - name: Set Azure subscription
      shell: bash
      run: |
        az account set --subscription ${{ inputs.subscription_id }}
        
    - name: Verify authentication
      shell: bash
      run: |
        az account show
```

```yaml
name: 'Teams Notification'
description: 'Send formatted Teams notifications'
inputs:
  webhook_url:
    description: 'Teams webhook URL'
    required: true
  status:
    description: 'Status (success/failure/warning)'
    required: true
  environment:
    description: 'Environment name'
    required: true
  message:
    description: 'Custom message'
    required: false
    default: ''
runs:
  using: 'composite'
  steps:
    - name: Send Teams notification
      shell: bash
      run: |
        STATUS_COLOR=""
        STATUS_ICON=""
        case "${{ inputs.status }}" in
          "success")
            STATUS_COLOR="28a745"
            STATUS_ICON="✅"
            ;;
          "failure")
            STATUS_COLOR="dc3545"
            STATUS_ICON="❌"
            ;;
          "warning")
            STATUS_COLOR="ffc107"
            STATUS_ICON="⚠️"
            ;;
        esac
        
        PAYLOAD=$(cat <<EOF
        {
          "@type": "MessageCard",
          "@context": "http://schema.org/extensions",
          "themeColor": "$STATUS_COLOR",
          "summary": "Terraform Pipeline - ${{ inputs.environment }}",
          "sections": [{
            "activityTitle": "$STATUS_ICON Terraform Pipeline - ${{ inputs.environment }}",
            "activitySubtitle": "Status: ${{ inputs.status }}",
            "facts": [{
              "name": "Repository:",
              "value": "${{ github.repository }}"
            }, {
              "name": "Branch:",
              "value": "${{ github.ref_name }}"
            }, {
              "name": "Commit:",
              "value": "${{ github.sha }}"
            }, {
              "name": "Actor:",
              "value": "${{ github.actor }}"
            }],
            "text": "${{ inputs.message }}"
          }],
          "potentialAction": [{
            "@type": "OpenUri",
            "name": "View Run",
            "targets": [{
              "os": "default",
              "uri": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }]
          }]
        }
        EOF
        )
        
        curl -H "Content-Type: application/json" -d "$PAYLOAD" "${{ inputs.webhook_url }}"
```

```yaml
name: 'Terraform Cache'
description: 'Cache/restore providers and modules'
inputs:
  environment:
    description: 'Environment name'
    required: true
  cache_key:
    description: 'Cache key'
    required: true
runs:
  using: 'composite'
  steps:
    - name: Cache Terraform providers
      uses: actions/cache@v4
      with:
        path: |
          .terraform
          ~/.terraform.d/plugin-cache
        key: ${{ inputs.cache_key }}
        restore-keys: |
          terraform-${{ inputs.environment }}-${{ runner.os }}-
          
    - name: Cache Terraform modules
      uses: actions/cache@v4
      with:
        path: |
          .terraform/modules
        key: terraform-modules-${{ inputs.environment }}-${{ hashFiles('**/*.tf') }}
        restore-keys: |
          terraform-modules-${{ inputs.environment }}-
```

```plaintext
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Example resource group using the module
module "example" {
  source = "../../modules/example"
  
  resource_group_name = var.resource_group_name
  location           = var.location
  environment        = "staging"
  tags               = var.tags
}
```

```plaintext
variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
  default     = "rg-staging"
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "East US"
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    Environment = "staging"
    ManagedBy   = "terraform"
  }
}
```

```plaintext
output "resource_group_name" {
  description = "Name of the created resource group"
  value       = module.example.resource_group_name
}

output "resource_group_id" {
  description = "ID of the created resource group"
  value       = module.example.resource_group_id
}

output "location" {
  description = "Location of the resource group"
  value       = module.example.location
}
```

```plaintext
terraform {
  backend "azurerm" {
    # Backend configuration will be provided via backend config file
    # or environment variables during terraform init
  }
}
```

```plaintext
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Example resource group using the module
module "example" {
  source = "../../modules/example"
  
  resource_group_name = var.resource_group_name
  location           = var.location
  environment        = "production"
  tags               = var.tags
}
```

```plaintext
variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
  default     = "rg-production"
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "East US"
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}
```

```plaintext
output "resource_group_name" {
  description = "Name of the created resource group"
  value       = module.example.resource_group_name
}

output "resource_group_id" {
  description = "ID of the created resource group"
  value       = module.example.resource_group_id
}

output "location" {
  description = "Location of the resource group"
  value       = module.example.location
}
```

```plaintext
terraform {
  backend "azurerm" {
    # Backend configuration will be provided via backend config file
    # or environment variables during terraform init
  }
}
```

```plaintext
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
  tags     = merge(var.tags, {
    Environment = var.environment
  })
}
```

```plaintext
variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
}

variable "location" {
  description = "Azure region for the resource group"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "tags" {
  description = "Tags to apply to the resource group"
  type        = map(string)
  default     = {}
}
```

```plaintext
output "resource_group_name" {
  description = "Name of the resource group"
  value       = azurerm_resource_group.main.name
}

output "resource_group_id" {
  description = "ID of the resource group"
  value       = azurerm_resource_group.main.id
}

output "location" {
  description = "Location of the resource group"
  value       = azurerm_resource_group.main.location
}
```

```yaml
name: 'Terraform Main Deployment'

on:
  push:
    branches: [develop, main]
    paths: ['terraform/**', 'config/**', '.github/workflows/**']
  pull_request:
    branches: [main]
    paths: ['terraform/**', 'config/**', '.github/workflows/**']
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  validation:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Terraform Format Check
        working-directory: terraform/environments/${{ matrix.environment }}
        run: terraform fmt -check -recursive
        
      - name: Terraform Lint
        working-directory: terraform/environments/${{ matrix.environment }}
        run: terraform validate
        
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      - name: Terraform Plan
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -out=tfplan-${{ matrix.environment }}
          
      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}
          path: terraform/environments/${{ matrix.environment }}/tfplan-${{ matrix.environment }}
          retention-days: 30

  compliance:
    runs-on: self-hosted
    needs: [load-config, validation]
    strategy:
      matrix:
        environment: [staging, production]
    steps:
      - uses: actions/checkout@v4
      
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Run Azure Policy Compliance Check
        run: |
          echo "Running Azure Policy compliance check for ${{ matrix.environment }}"
          # This will be implemented based on your specific Azure Policy requirements
          az policy state list --resource-group "rg-${{ matrix.environment }}" --all || true

  deploy-staging:
    if: github.ref == 'refs/heads/develop' || github.event_name == 'workflow_dispatch'
    runs-on: self-hosted
    needs: [load-config, validation, compliance]
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/staging
          environment: staging
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-staging
          path: terraform/environments/staging
          
      - name: Configure backend
        working-directory: terraform/environments/staging
        run: |
          BACKEND_CONFIG=$(echo '${{ needs.load-config.outputs.staging-config }}' | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      - name: Terraform Apply
        working-directory: terraform/environments/staging
        run: terraform apply tfplan-staging
        
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: staging
          message: "Staging deployment completed successfully"
          
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: staging
          message: "Staging deployment failed"

  deploy-production:
    if: github.ref == 'refs/heads/main' || (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'production')
    runs-on: self-hosted
    needs: [load-config, validation, compliance, deploy-staging]
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/production
          environment: production
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-production
          path: terraform/environments/production
          
      - name: Configure backend
        working-directory: terraform/environments/production
        run: |
          BACKEND_CONFIG=$(echo '${{ needs.load-config.outputs.production-config }}' | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      - name: Terraform Apply
        working-directory: terraform/environments/production
        run: terraform apply tfplan-production
        
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: production
          message: "Production deployment completed successfully"
          
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: production
          message: "Production deployment failed"
```

```yaml
name: 'Terraform Drift Detection'

on:
  schedule:
    - cron: '0 9 * * 0'  # Weekly on Sundays at 9 AM UTC
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  drift-detection:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      - name: Check for drift
        id: drift
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -detailed-exitcode -out=drift-plan-${{ matrix.environment }} > drift-output-${{ matrix.environment }}.txt 2>&1
          exit_code=$?
          echo "exit_code=$exit_code" >> $GITHUB_OUTPUT
          
          if [ $exit_code -eq 2 ]; then
            echo "has_drift=true" >> $GITHUB_OUTPUT
            echo "Drift detected in ${{ matrix.environment }} environment"
          else
            echo "has_drift=false" >> $GITHUB_OUTPUT
            echo "No drift detected in ${{ matrix.environment }} environment"
          fi
          
      - name: Upload drift report
        if: steps.drift.outputs.has_drift == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: drift-report-${{ matrix.environment }}
          path: |
            terraform/environments/${{ matrix.environment }}/drift-output-${{ matrix.environment }}.txt
            terraform/environments/${{ matrix.environment }}/drift-plan-${{ matrix.environment }}
          retention-days: 30
          
      - name: Notify drift detected
        if: steps.drift.outputs.has_drift == 'true'
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: warning
          environment: ${{ matrix.environment }}
          message: "Infrastructure drift detected! Please review the drift report artifact."
          
      - name: Notify no drift
        if: steps.drift.outputs.has_drift == 'false'
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ matrix.environment }}
          message: "No infrastructure drift detected."
```

```yaml
name: 'Terraform State Management'

on:
  workflow_dispatch:
    inputs:
      operation:
        description: 'State operation to perform'
        required: true
        type: choice
        options:
          - unlock
          - backup
          - restore
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      lock_id:
        description: 'Lock ID (required for unlock operation)'
        required: false
      backup_name:
        description: 'Backup name (required for restore operation)'
        required: false
  schedule:
    - cron: '0 2 * * 0'  # Weekly backup on Sundays at 2 AM UTC

permissions:
  id-token: write
  contents: read

jobs:
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  state-operation:
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment || 'staging' }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        if: github.event.inputs.operation == 'unlock'
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment || 'staging' }}
          environment: ${{ github.event.inputs.environment || 'staging' }}
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Unlock state
        if: github.event.inputs.operation == 'unlock'
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
          terraform force-unlock ${{ github.event.inputs.lock_id }}
          
      - name: Backup state files
        if: github.event.inputs.operation == 'backup' || github.event_name == 'schedule'
        run: |
          # Get current timestamp
          TIMESTAMP=$(date +%Y%m%d-%H%M%S)
          
          # Backup staging state
          STAGING_CONFIG='${{ needs.load-config.outputs.staging-config }}'
          STAGING_BACKEND=$(echo $STAGING_CONFIG | jq -r '.terraform.backend')
          
          az storage blob download \
            --account-name $(echo $STAGING_BACKEND | jq -r '.storage_account_name') \
            --container-name $(echo $STAGING_BACKEND | jq -r '.container_name') \
            --name $(echo $STAGING_BACKEND | jq -r '.key') \
            --file staging-${TIMESTAMP}.tfstate
            
          # Backup production state
          PRODUCTION_CONFIG='${{ needs.load-config.outputs.production-config }}'
          PRODUCTION_BACKEND=$(echo $PRODUCTION_CONFIG | jq -r '.terraform.backend')
          
          az storage blob download \
            --account-name $(echo $PRODUCTION_BACKEND | jq -r '.storage_account_name') \
            --container-name $(echo $PRODUCTION_BACKEND | jq -r '.container_name') \
            --name $(echo $PRODUCTION_BACKEND | jq -r '.key') \
            --file production-${TIMESTAMP}.tfstate
            
          # Upload to backup storage
          BASE_CONFIG='${{ needs.load-config.outputs.base-config }}'
          BACKUP_STORAGE=$(echo $BASE_CONFIG | jq -r '.storage')
          
          az storage blob upload \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name staging-${TIMESTAMP}.tfstate \
            --file staging-${TIMESTAMP}.tfstate
            
          az storage blob upload \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name production-${TIMESTAMP}.tfstate \
            --file production-${TIMESTAMP}.tfstate
            
      - name: Restore state file
        if: github.event.inputs.operation == 'restore'
        run: |
          # Download backup from backup storage
          BASE_CONFIG='${{ needs.load-config.outputs.base-config }}'
          BACKUP_STORAGE=$(echo $BASE_CONFIG | jq -r '.storage')
          
          az storage blob download \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name ${{ github.event.inputs.backup_name }} \
            --file restored.tfstate
            
          # Upload to environment state storage
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          az storage blob upload \
            --account-name $(echo $BACKEND_CONFIG | jq -r '.storage_account_name') \
            --container-name $(echo $BACKEND_CONFIG | jq -r '.container_name') \
            --name $(echo $BACKEND_CONFIG | jq -r '.key') \
            --file restored.tfstate \
            --overwrite
            
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment || 'all' }}
          message: "State ${{ github.event.inputs.operation || 'backup' }} operation completed successfully"
          
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment || 'all' }}
          message: "State ${{ github.event.inputs.operation || 'backup' }} operation failed"
```

```yaml
name: 'Terraform Utilities'

on:
  workflow_dispatch:
    inputs:
      operation:
        description: 'Utility operation to perform'
        required: true
        type: choice
        options:
          - tfupdate
          - dependency-graph
          - target-apply
          - target-destroy
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      target_resources:
        description: 'Target resources (comma-separated, required for target operations)'
        required: false
  push:
    paths: ['terraform/**']
    branches: [develop, main]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  tfupdate:
    if: github.event.inputs.operation == 'tfupdate' || github.event_name == 'push'
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]
    steps:
      - uses: actions/checkout@v4
      
      - name: Install tfupdate
        run: |
          curl -fsSL https://github.com/minamijoyo/tfupdate/releases/download/v0.7.1/tfupdate_0.7.1_linux_amd64.tar.gz | tar -xz
          sudo mv tfupdate /usr/local/bin/
          
      - name: Update Terraform providers
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          tfupdate provider azurerm --version "> 3.0, < 4.0" .
          
      - name: Check for updates
        id: check-updates
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          if git diff --quiet; then
            echo "has_updates=false" >> $GITHUB_OUTPUT
          else
            echo "has_updates=true" >> $GITHUB_OUTPUT
          fi
          
      - name: Create Pull Request
        if: steps.check-updates.outputs.has_updates == 'true' && github.event_name == 'push'
        uses: peter-evans/create-pull-request@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: "chore: update terraform providers for ${{ matrix.environment }}"
          title: "Update Terraform providers for ${{ matrix.environment }}"
          body: |
            Automated Terraform provider updates for ${{ matrix.environment }} environment.
            
            Changes made by tfupdate:
            - Updated provider versions to latest compatible versions
            
            Please review the changes before merging.
          branch: tfupdate-${{ matrix.environment }}-${{ github.run_number }}

  dependency-graph:
    if: github.event.inputs.operation == 'dependency-graph'
    runs-on: self-hosted
    needs: load-config
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment }}
          environment: ${{ github.event.inputs.environment }}
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Configure backend
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      - name: Generate dependency graph
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          terraform graph | dot -Tsvg > dependency-graph-${{ github.event.inputs.environment }}.svg
          terraform graph | dot -Tpng > dependency-graph-${{ github.event.inputs.environment }}.png
          
      - name: Upload dependency graph
        uses: actions/upload-artifact@v4
        with:
          name: dependency-graph-${{ github.event.inputs.environment }}
          path: |
            terraform/environments/${{ github.event.inputs.environment }}/dependency-graph-${{ github.event.inputs.environment }}.svg
            terraform/environments/${{ github.event.inputs.environment }}/dependency-graph-${{ github.event.inputs.environment }}.png
          retention-days: 30

  target-operations:
    if: github.event.inputs.operation == 'target-apply' || github.event.inputs.operation == 'target-destroy'
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment }}
          environment: ${{ github.event.inputs.environment }}
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Configure backend
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      - name: Prepare target flags
        id: targets
        run: |
          TARGETS="${{ github.event.inputs.target_resources }}"
          TARGET_FLAGS=""
          IFS=',' read -ra TARGET_ARRAY <<< "$TARGETS"
          for target in "${TARGET_ARRAY[@]}"; do
            TARGET_FLAGS="$TARGET_FLAGS -target=$target"
          done
          echo "flags=$TARGET_FLAGS" >> $GITHUB_OUTPUT
          
      - name: Plan targeted operation
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          if [ "${{ github.event.inputs.operation }}" = "target-destroy" ]; then
            terraform plan -destroy ${{ steps.targets.outputs.flags }} -out=target-plan
          else
            terraform plan ${{ steps.targets.outputs.flags }} -out=target-plan
          fi
          
      - name: Apply targeted operation
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: terraform apply target-plan
        
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment }}
          message: "Target operation ${{ github.event.inputs.operation }} completed successfully for resources: ${{ github.event.inputs.target_resources }}"
          
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment }}
          message: "Target operation ${{ github.event.inputs.operation }} failed for resources: ${{ github.event.inputs.target_resources }}"
```

```yaml
name: 'Terraform Compliance'

on:
  pull_request:
    branches: [main, develop]
    paths: ['terraform/**']
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to check'
        required: true
        type: choice
        options:
          - staging
          - production
          - all

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  compliance-check:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: ${{ github.event.inputs.environment == 'all' && fromJson('["staging", "production"]') || github.event.inputs.environment && fromJson(format('["{0}"]', github.event.inputs.environment)) || fromJson('["staging", "production"]') }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      - name: Generate plan for compliance check
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -out=compliance-plan-${{ matrix.environment }}
          terraform show -json compliance-plan-${{ matrix.environment }} > compliance-plan-${{ matrix.environment }}.json
          
      - name: Install tflint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
          
      - name: Run tflint security checks
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          tflint --init
          tflint --format json > tflint-report-${{ matrix.environment }}.json || true
          
      - name: Install checkov
        run: |
          pip3 install checkov
          
      - name: Run checkov security scan
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          checkov -f compliance-plan-${{ matrix.environment }}.json --framework terraform_plan --output json > checkov-report-${{ matrix.environment }}.json || true
          
      - name: Check Azure Policy compliance
        run: |
          echo "Checking Azure Policy compliance for ${{ matrix.environment }} environment..."
          
          # Get resource groups for the environment
          RESOURCE_GROUPS=$(az group list --query "[?contains(name, '${{ matrix.environment }}')].name" -o tsv)
          
          # Create compliance report
          echo '{"environment": "${{ matrix.environment }}", "policy_states": []}' > policy-compliance-${{ matrix.environment }}.json
          
          for rg in $RESOURCE_GROUPS; do
            echo "Checking resource group: $rg"
            az policy state list --resource-group "$rg" --query '[].{policyDefinitionName: policyDefinitionName, complianceState: complianceState, resourceId: resourceId}' >> policy-states-temp.json || true
          done
          
          # Merge policy states into main report
          if [ -f policy-states-temp.json ]; then
            jq '.policy_states += input' policy-compliance-${{ matrix.environment }}.json policy-states-temp.json > temp.json
            mv temp.json policy-compliance-${{ matrix.environment }}.json
            rm policy-states-temp.json
          fi
          
      - name: Generate compliance summary
        run: |
          # Create comprehensive compliance report
          cat > compliance-summary-${{ matrix.environment }}.md << EOF
          # Compliance Report - ${{ matrix.environment }} Environment
          
          ## TFLint Security Checks
          \`\`\`json
          $(cat terraform/environments/${{ matrix.environment }}/tflint-report-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Checkov Security Scan
          \`\`\`json
          $(cat terraform/environments/${{ matrix.environment }}/checkov-report-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Azure Policy Compliance
          \`\`\`json
          $(cat policy-compliance-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Summary
          - **Environment**: ${{ matrix.environment }}
          - **Scan Date**: $(date)
          - **Repository**: ${{ github.repository }}
          - **Commit**: ${{ github.sha }}
          EOF
          
      - name: Upload compliance reports
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report-${{ matrix.environment }}
          path: |
            terraform/environments/${{ matrix.environment }}/tflint-report-${{ matrix.environment }}.json
            terraform/environments/${{ matrix.environment }}/checkov-report-${{ matrix.environment }}.json
            policy-compliance-${{ matrix.environment }}.json
            compliance-summary-${{ matrix.environment }}.md
          retention-days: 30
          
      - name: Comment PR with compliance results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            try {
              const summary = fs.readFileSync('compliance-summary-${{ matrix.environment }}.md', 'utf8');
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: summary
              });
            } catch (error) {
              console.log('Could not read compliance summary file');
            }
            
      - name: Notify compliance results
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ matrix.environment }}
          message: "Compliance check completed for ${{ matrix.environment }}. Check artifacts for detailed results."
```

```yaml
name: 'Terraform Testing'

on:
  workflow_dispatch:
    inputs:
      test_suite:
        description: 'Test suite to run'
        required: true
        type: choice
        options:
          - basic
          - integration
          - all
      environment:
        description: 'Environment to test against'
        required: true
        type: choice
        options:
          - staging
          - production
  push:
    paths: ['tests/**', 'terraform/**']
    branches: [develop, main]

permissions:
  id-token: write
  contents: read

jobs:
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  terraform-tests:
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment || 'staging' }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
          
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment || 'staging' }}
          environment: ${{ github.event.inputs.environment || 'staging' }}
          
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      - name: Install Go dependencies
        working-directory: tests/terratest
        run: |
          go mod download
          go mod tidy
          
      - name: Run basic tests
        if: github.event.inputs.test_suite == 'basic' || github.event.inputs.test_suite == 'all' || github.event_name == 'push'
        working-directory: tests/terratest
        run: |
          go test -v -timeout 30m -run TestTerraformBasicExample
          
      - name: Run integration tests
        if: github.event.inputs.test_suite == 'integration' || github.event.inputs.test_suite == 'all'
        working-directory: tests/terratest
        run: |
          echo "Integration tests would run here"
          echo "Add your integration test functions to basic_test.go or create separate test files"
          # go test -v -timeout 60m -run TestTerraformIntegration
          
      - name: Generate test report
        if: always()
        working-directory: tests/terratest
        run: |
          go test -v -json > test-results.json || true
          
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ github.event.inputs.environment || 'staging' }}
          path: tests/terratest/test-results.json
          retention-days: 30
          
      - name: Notify test success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment || 'staging' }}
          message: "Terraform tests completed successfully for ${{ github.event.inputs.test_suite || 'basic' }} test suite"
          
      - name: Notify test failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment || 'staging' }}
          message: "Terraform tests failed for ${{ github.event.inputs.test_suite || 'basic' }} test suite"
```

```go.mod
module terraform-tests

go 1.21

require (
	github.com/gruntwork-io/terratest v0.46.0
	github.com/stretchr/testify v1.8.4
)
```

```go
package test

import (
	"testing"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestTerraformBasicExample(t *testing.T) {
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../../terraform/environments/staging",
		Vars: map[string]interface{}{
			"resource_group_name": "test-rg-terratest",
			"location": "East US",
		},
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	// Validate outputs
	resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")
	assert.Equal(t, "test-rg-terratest", resourceGroupName)
}
```

```go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestTerraformBasicExample(t *testing.T) {
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../../terraform/environments/staging",
		Vars: map[string]interface{}{
			"resource_group_name": "test-rg-terratest",
			"location":            "East US",
		},
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	// Validate outputs
	resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")
	assert.Equal(t, "test-rg-terratest", resourceGroupName)
}
```

```json
{
  "compliance_rules": {
    "required_tags": {
      "description": "Resources must have required tags",
      "tags": ["Environment", "ManagedBy", "Owner", "CostCenter"]
    },
    "resource_naming": {
      "description": "Resources must follow naming conventions",
      "patterns": {
        "resource_groups": "rg-{environment}-{purpose}",
        "storage_accounts": "st{environment}{purpose}",
        "virtual_networks": "vnet-{environment}-{purpose}"
      }
    },
    "security_requirements": {
      "description": "Security compliance requirements",
      "rules": [
        "Storage accounts must use HTTPS only",
        "Network security groups must not allow unrestricted inbound access",
        "Virtual machines must have antimalware extensions",
        "Key vaults must have soft delete enabled"
      ]
    },
    "cost_management": {
      "description": "Cost optimization rules",
      "rules": [
        "Virtual machines must be appropriate sizes",
        "Unused resources must be identified and removed",
        "Storage accounts must use appropriate tiers"
      ]
    }
  },
  "exemptions": {
    "development": {
      "description": "Exemptions for development environments",
      "exempt_rules": ["cost_management"]
    },
    "testing": {
      "description": "Exemptions for testing environments", 
      "exempt_rules": ["cost_management"]
    }
  }
}
```

```markdown
# GitHub Actions Terraform Templates

A comprehensive set of GitHub Actions workflows and Terraform templates for Azure infrastructure deployment with OIDC authentication, automated testing, compliance scanning, and operational workflows.

## Features

- 🔐 **Azure OIDC Authentication** - Secure authentication without storing credentials
- 🚀 **Multi-Environment Support** - Staging and production environments with approval gates
- 📦 **Modular Architecture** - Reusable GitHub Actions and Terraform modules
- 🔍 **Compliance Scanning** - Azure Policy validation, TFLint, and Checkov security checks
- 🧪 **Infrastructure Testing** - Terratest integration for automated testing
- 📊 **Drift Detection** - Weekly automated drift detection
- 🔧 **Utility Operations** - State management, dependency graphs, targeted operations
- 📢 **Teams Notifications** - Integrated Microsoft Teams webhook notifications
- ⚡ **Performance Optimized** - Terraform provider caching and parallel execution

## Project Structure

```
/
├── .github/
│   ├── workflows/                 # GitHub Actions workflows
│   │   ├── terraform-main.yml           # Main deployment pipeline
│   │   ├── terraform-drift-detection.yml
│   │   ├── terraform-state-management.yml
│   │   ├── terraform-utilities.yml
│   │   ├── terraform-compliance.yml
│   │   └── terraform-testing.yml
│   └── actions/                   # Reusable GitHub Actions
│       ├── setup-terraform/
│       ├── azure-login/
│       ├── teams-notification/
│       └── terraform-cache/
├── config/                        # Environment configurations
│   ├── base.json                  # Shared configuration
│   ├── staging.json               # Staging-specific config
│   └── production.json            # Production-specific config
├── terraform/
│   ├── environments/              # Environment-specific deployments
│   │   ├── staging/
│   │   └── production/
│   └── modules/                   # Reusable Terraform modules
│       └── example/
├── tests/
│   ├── terratest/                 # Infrastructure tests
│   └── azure-policy/              # Compliance rules
└── README.md
```

## Quick Start

### 1. Prerequisites

- Azure subscription with appropriate permissions
- GitHub repository with Actions enabled
- Azure OIDC application registration configured
- Self-hosted GitHub Actions runners (optional but recommended)

### 2. Configuration

#### Azure OIDC Setup

1. Create an Azure AD application registration
2. Configure federated credentials for GitHub OIDC
3. Assign appropriate Azure RBAC roles to the service principal

#### Repository Configuration

1. Fill in the configuration files in the `config/` directory:

**config/base.json:**
```json
{
  "azure": {
    "tenant_id": "your-tenant-id",
    "subscription_id": "your-subscription-id", 
    "client_id": "your-client-id"
  },
  "notifications": {
    "teams_webhook": "your-teams-webhook-url"
  }
}
```

**config/staging.json & config/production.json:**
```json
{
  "terraform": {
    "backend": {
      "storage_account_name": "your-state-storage-account",
      "container_name": "your-state-container",
      "resource_group_name": "your-state-resource-group"
    }
  }
}
```

2. Set up GitHub Environment Protection Rules:
   - Go to Settings → Environments
   - Create `staging` and `production` environments
   - Configure required reviewers and deployment branches

### 3. Workflows Overview

#### Main Deployment (`terraform-main.yml`)
- **Triggers:** Push to develop/main, PRs to main, manual dispatch
- **Features:** Parallel validation, plan caching, approval gates
- **Flow:** Validation → Compliance → Staging → Production

#### Drift Detection (`terraform-drift-detection.yml`)
- **Schedule:** Weekly on Sundays
- **Purpose:** Detect infrastructure drift and generate reports

#### State Management (`terraform-state-management.yml`)
- **Operations:** Unlock, backup, restore state files
- **Schedule:** Weekly automated backups

#### Utilities (`terraform-utilities.yml`)
- **Operations:** tfupdate, dependency graphs, targeted apply/destroy
- **Triggers:** Manual dispatch, terraform file changes

#### Compliance (`terraform-compliance.yml`)
- **Tools:** TFLint, Checkov, Azure Policy validation
- **Triggers:** Pull requests, manual dispatch

#### Testing (`terraform-testing.yml`)
- **Framework:** Terratest with Go
- **Suites:** Basic, integration, or all tests

## Usage Examples

### Deploy to Staging
```bash
# Automatically triggers on push to develop branch
git push origin develop

# Or manually trigger
# Go to Actions → Terraform Main Deployment → Run workflow
```

### Run Drift Detection
```bash
# Manually trigger drift detection
# Go to Actions → Terraform Drift Detection → Run workflow
```

### Perform Targeted Operations
```bash
# Target specific resources for apply/destroy
# Go to Actions → Terraform Utilities → Run workflow
# Operation: target-apply
# Target resources: module.example.azurerm_resource_group.main
```

### Run Infrastructure Tests
```bash
# Run Terratest suite
# Go to Actions → Terraform Testing → Run workflow
# Test suite: basic
# Environment: staging
```

## Reusable Actions

### Setup Terraform
```yaml
- uses: ./.github/actions/setup-terraform
  with:
    terraform_version: latest
    working_directory: terraform/environments/staging
    environment: staging
```

### Azure Login
```yaml
- uses: ./.github/actions/azure-login
  with:
    tenant_id: ${{ secrets.AZURE_TENANT_ID }}
    subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    client_id: ${{ secrets.AZURE_CLIENT_ID }}
```

### Teams Notification
```yaml
- uses: ./.github/actions/teams-notification
  with:
    webhook_url: ${{ secrets.TEAMS_WEBHOOK }}
    status: success
    environment: production
    message: "Deployment completed successfully"
```

## Security Considerations

### RBAC Configuration
- **Staging Environment:** Contributor role on staging resource groups
- **Production Environment:** Custom role with minimal required permissions
- **State Storage:** Storage Blob Data Contributor role
- **Backup Storage:** Storage Blob Data Reader/Writer role

### GitHub Repository Settings
- Branch protection rules for main/develop branches
- Required status checks for all workflows
- Restrict push access to authorized users
- Enable signed commits (recommended)

## Troubleshooting

### Common Issues

1. **Authentication Failures**
   - Verify OIDC configuration in Azure AD
   - Check federated credential subject claims
   - Ensure service principal has appropriate roles

2. **State Lock Issues**
   - Use the state management workflow to unlock
   - Verify Azure Storage permissions
   - Check for concurrent workflow runs

3. **Plan Cache Misses**
   - Terraform provider versions changed
   - Lock file modifications
   - Clear cache and re-run workflow

### Debugging

Enable debug logging by setting repository secrets:
- `ACTIONS_STEP_DEBUG=true`
- `ACTIONS_RUNNER_DEBUG=true`

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with staging environment
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:
- Create GitHub Issues for bugs and feature requests
- Check the troubleshooting section
- Review workflow run logs for detailed error messages
```

Implementation complete! I have successfully created all 30 items from the implementation checklist exactly as specified in the plan:

✅ **Configuration Files:**
- `config/base.json` - Global Azure OIDC and shared settings
- `config/staging.json` - Staging-specific backend configuration  
- `config/production.json` - Production-specific backend configuration

✅ **Reusable GitHub Actions:**
- `setup-terraform` - Terraform installation with caching
- `azure-login` - OIDC authentication with Azure
- `teams-notification` - Teams webhook notifications
- `terraform-cache` - Provider and module caching

✅ **Terraform Structure:**
- Staging environment with main.tf, variables.tf, outputs.tf, terraform.tf
- Production environment with main.tf, variables.tf, outputs.tf, terraform.tf
- Example module with resource group implementation

✅ **GitHub Workflows:**
- `terraform-main.yml` - Main deployment with approval gates and caching
- `terraform-drift-detection.yml` - Weekly drift detection
- `terraform-state-management.yml` - State unlock, backup, restore operations
- `terraform-utilities.yml` - tfupdate, dependency graphs, targeted operations
- `terraform-compliance.yml` - Azure Policy, TFLint, Checkov scanning
- `terraform-testing.yml` - Terratest integration

✅ **Testing & Compliance:**
- `tests/terratest/go.mod` - Go module definition
- `tests/terratest/basic_test.go` - Simple infrastructure test
- `tests/azure-policy/compliance-rules.json` - Policy definitions

✅ **Documentation:**
- `README.md` - Comprehensive setup and usage documentation

All files include the specified features:
- Environment-specific configurations with placeholder values
- Smart triggering with path filters
- Terraform plan caching for performance
- Parallel execution where appropriate
- Azure OIDC authentication
- Teams notifications
- Manual approval gates
- Weekly drift detection and state backups
- Terratest example for infrastructure testing
- Compliance scanning with multiple tools

The templates are ready for use and follow all the architectural decisions made during the planning phase.

---

**User**

can you please add some more necessary comments into the pipeline yaml files? the comments should describe what does the pipeline do and some other comments you think can be useful when people first look at the pipeline

---

**Cursor**

[MODE: EXECUTE]

I'll add comprehensive comments to all the pipeline YAML files to make them more understandable for first-time users. Let me enhance each workflow with detailed explanations.

```github-actions-workflow
# =============================================================================
# TERRAFORM MAIN DEPLOYMENT PIPELINE
# =============================================================================
# This workflow handles the complete deployment lifecycle for both staging and
# production environments with automated validation, compliance checks, and
# approval gates. It implements GitFlow branching strategy where:
# - develop branch deploys to staging
# - main branch deploys to production (after staging success)
# =============================================================================

name: 'Terraform Main Deployment'

# =============================================================================
# TRIGGERS: When this workflow runs
# =============================================================================
on:
  push:
    branches: [develop, main]  # Auto-trigger on push to these branches
    paths: ['terraform/**', 'config/**', '.github/workflows/**']  # Only when relevant files change
  pull_request:
    branches: [main]  # Run validation on PRs to main branch
    paths: ['terraform/**', 'config/**', '.github/workflows/**']
  workflow_dispatch:  # Allow manual triggering with environment selection
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

# =============================================================================
# PERMISSIONS: OIDC requires specific permissions for Azure authentication
# =============================================================================
permissions:
  id-token: write      # Required for OIDC token generation
  contents: read       # Read repository contents
  pull-requests: write # Comment on PRs with results

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Loads environment-specific configurations from JSON files and makes them
  # available to other jobs. This centralizes configuration management.
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration (Azure credentials, Teams webhook, etc.)
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load staging-specific configuration (backend, deployment settings)
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load production-specific configuration
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # VALIDATION JOB
  # ===========================================================================
  # Runs in parallel for both environments to:
  # 1. Validate Terraform syntax and formatting
  # 2. Generate execution plans
  # 3. Cache plans for later use in deployment
  # Uses matrix strategy for parallel execution efficiency
  # ===========================================================================
  validation:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]  # Run validation for both environments in parallel
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform with caching to improve performance
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      # Authenticate with Azure using OIDC (no stored credentials needed)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Ensure Terraform code follows formatting standards
      - name: Terraform Format Check
        working-directory: terraform/environments/${{ matrix.environment }}
        run: terraform fmt -check -recursive
        
      # Validate Terraform syntax and configuration
      - name: Terraform Lint
        working-directory: terraform/environments/${{ matrix.environment }}
        run: terraform validate
        
      # Configure remote state backend dynamically based on environment
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Select appropriate config based on current environment
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          # Initialize Terraform with environment-specific backend configuration
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Generate execution plan and save it for later use in deployment
      - name: Terraform Plan
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -out=tfplan-${{ matrix.environment }}
          
      # Store plan as artifact for use in deployment jobs (avoids re-planning)
      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}
          path: terraform/environments/${{ matrix.environment }}/tfplan-${{ matrix.environment }}
          retention-days: 30

  # ===========================================================================
  # COMPLIANCE SCANNING JOB
  # ===========================================================================
  # Runs security and policy compliance checks in parallel with validation
  # to catch issues early. Includes Azure Policy validation.
  # ===========================================================================
  compliance:
    runs-on: self-hosted
    needs: [load-config, validation]
    strategy:
      matrix:
        environment: [staging, production]  # Check compliance for both environments
    steps:
      - uses: actions/checkout@v4
      
      # Authenticate with Azure to check policy compliance
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Check existing resources against Azure Policy definitions
      - name: Run Azure Policy Compliance Check
        run: |
          echo "Running Azure Policy compliance check for ${{ matrix.environment }}"
          # Query policy compliance state for environment-specific resource groups
          # This helps identify policy violations before deployment
          az policy state list --resource-group "rg-${{ matrix.environment }}" --all || true

  # ===========================================================================
  # STAGING DEPLOYMENT JOB
  # ===========================================================================
  # Deploys to staging environment with the following safeguards:
  # 1. Only runs for develop branch or manual dispatch
  # 2. Requires manual approval (configured in GitHub environment settings)
  # 3. Uses pre-generated plan from validation job
  # 4. Sends Teams notifications for visibility
  # ===========================================================================
  deploy-staging:
    # Only deploy to staging from develop branch or manual trigger
    if: github.ref == 'refs/heads/develop' || github.event_name == 'workflow_dispatch'
    runs-on: self-hosted
    needs: [load-config, validation, compliance]
    environment: staging  # Triggers GitHub environment protection rules and manual approval
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform with same configuration as validation
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/staging
          environment: staging
          
      # Authenticate with Azure for deployment operations
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Download the pre-generated plan from validation job
      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-staging
          path: terraform/environments/staging
          
      # Reinitialize Terraform backend for deployment context
      - name: Configure backend
        working-directory: terraform/environments/staging
        run: |
          BACKEND_CONFIG=$(echo '${{ needs.load-config.outputs.staging-config }}' | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Apply the pre-approved plan (no additional confirmation needed)
      - name: Terraform Apply
        working-directory: terraform/environments/staging
        run: terraform apply tfplan-staging
        
      # Notify team of successful staging deployment
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: staging
          message: "Staging deployment completed successfully"
          
      # Notify team if staging deployment fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: staging
          message: "Staging deployment failed"

  # ===========================================================================
  # PRODUCTION DEPLOYMENT JOB
  # ===========================================================================
  # Deploys to production with maximum safety:
  # 1. Only runs for main branch or manual production dispatch
  # 2. Requires successful staging deployment first
  # 3. Requires manual approval via GitHub environment protection
  # 4. Uses pre-generated and validated plan
  # 5. Full notification coverage for audit trail
  # ===========================================================================
  deploy-production:
    # Production deployment conditions: main branch OR manual production trigger
    if: github.ref == 'refs/heads/main' || (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'production')
    runs-on: self-hosted
    needs: [load-config, validation, compliance, deploy-staging]  # Must wait for staging success
    environment: production  # Triggers GitHub environment protection rules (manual approval)
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for production deployment
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/production
          environment: production
          
      # Authenticate with Azure using production credentials
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Download the production plan generated during validation
      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-production
          path: terraform/environments/production
          
      # Configure production backend
      - name: Configure backend
        working-directory: terraform/environments/production
        run: |
          BACKEND_CONFIG=$(echo '${{ needs.load-config.outputs.production-config }}' | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Apply to production (this step requires manual approval)
      - name: Terraform Apply
        working-directory: terraform/environments/production
        run: terraform apply tfplan-production
        
      # Notify team of successful production deployment
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: production
          message: "Production deployment completed successfully"
          
      # Critical: Notify team immediately if production deployment fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: production
          message: "Production deployment failed"
```

```github-actions-workflow
# =============================================================================
# TERRAFORM DRIFT DETECTION PIPELINE
# =============================================================================
# This workflow automatically detects infrastructure drift by comparing the
# current state of resources in Azure with the expected state defined in
# Terraform configurations. It runs weekly but can also be triggered manually.
#
# Key Features:
# - Automated weekly execution (Sundays at 9 AM UTC)
# - Parallel drift detection for both staging and production
# - Detailed drift reports saved as artifacts
# - Teams notifications for drift alerts
# - Non-destructive - only detects changes, doesn't fix them
# =============================================================================

name: 'Terraform Drift Detection'

# =============================================================================
# TRIGGERS: Scheduled weekly runs + manual execution
# =============================================================================
on:
  schedule:
    - cron: '0 9 * * 0'  # Weekly on Sundays at 9 AM UTC (adjust timezone as needed)
  workflow_dispatch:      # Allow manual triggering for immediate drift checks

# =============================================================================
# PERMISSIONS: Required for Azure OIDC authentication
# =============================================================================
permissions:
  id-token: write  # Required for OIDC token generation
  contents: read   # Read repository contents and configurations

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Same configuration loading pattern as main deployment workflow
  # to ensure consistent environment settings across all pipelines
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration (Azure credentials, notification settings)
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific backend configurations
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # DRIFT DETECTION JOB
  # ===========================================================================
  # Runs terraform plan with -detailed-exitcode to detect any differences
  # between current infrastructure state and Terraform configuration.
  # Uses matrix strategy to check both environments in parallel.
  # ===========================================================================
  drift-detection:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]  # Check both environments simultaneously
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform with caching for performance
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      # Authenticate with Azure to access current resource state
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Initialize Terraform with the appropriate backend for this environment
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Select appropriate config based on current environment
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          # Initialize with environment-specific state backend
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Run terraform plan with detailed exit codes to detect drift
      - name: Check for drift
        id: drift
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Run terraform plan with detailed exit codes:
          # 0 = no changes, 1 = error, 2 = changes detected (drift)
          terraform plan -detailed-exitcode -out=drift-plan-${{ matrix.environment }} > drift-output-${{ matrix.environment }}.txt 2>&1
          exit_code=$?
          echo "exit_code=$exit_code" >> $GITHUB_OUTPUT
          
          # Interpret the exit code and set drift flag
          if [ $exit_code -eq 2 ]; then
            echo "has_drift=true" >> $GITHUB_OUTPUT
            echo "Drift detected in ${{ matrix.environment }} environment"
          else
            echo "has_drift=false" >> $GITHUB_OUTPUT
            echo "No drift detected in ${{ matrix.environment }} environment"
          fi
          
      # Save drift analysis reports as artifacts for review
      - name: Upload drift report
        if: steps.drift.outputs.has_drift == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: drift-report-${{ matrix.environment }}
          path: |
            terraform/environments/${{ matrix.environment }}/drift-output-${{ matrix.environment }}.txt
            terraform/environments/${{ matrix.environment }}/drift-plan-${{ matrix.environment }}
          retention-days: 30
          
      # Alert team via Teams when drift is detected (requires immediate attention)
      - name: Notify drift detected
        if: steps.drift.outputs.has_drift == 'true'
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: warning
          environment: ${{ matrix.environment }}
          message: "Infrastructure drift detected! Please review the drift report artifact."
          
      # Send confirmation when no drift is found (good news!)
      - name: Notify no drift
        if: steps.drift.outputs.has_drift == 'false'
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ matrix.environment }}
          message: "No infrastructure drift detected."
```

```github-actions-workflow
# =============================================================================
# TERRAFORM STATE MANAGEMENT PIPELINE
# =============================================================================
# This workflow provides critical state management operations for Terraform:
#
# 1. UNLOCK OPERATION: Force-unlock stuck Terraform state files
# 2. BACKUP OPERATION: Create timestamped backups of state files
# 3. RESTORE OPERATION: Restore state files from previous backups
#
# CRITICAL SAFETY NOTES:
# - These operations directly manipulate Terraform state files
# - Improper use can cause data loss or infrastructure corruption
# - Always ensure you have recent backups before performing operations
# - Test restore operations in non-production environments first
# =============================================================================

name: 'Terraform State Management'

# =============================================================================
# TRIGGERS: Manual operations + automated weekly backups
# =============================================================================
on:
  workflow_dispatch:  # Manual execution with operation selection
    inputs:
      operation:
        description: 'State operation to perform'
        required: true
        type: choice
        options:
          - unlock    # Force unlock stuck state file
          - backup    # Create manual backup of state files
          - restore   # Restore state from backup
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      lock_id:
        description: 'Lock ID (required for unlock operation)'
        required: false
      backup_name:
        description: 'Backup name (required for restore operation)'
        required: false
  schedule:
    - cron: '0 2 * * 0'  # Weekly backup on Sundays at 2 AM UTC (off-peak hours)

# =============================================================================
# PERMISSIONS: Enhanced permissions needed for state operations
# =============================================================================
permissions:
  id-token: write  # Required for OIDC authentication
  contents: read   # Read repository contents and configurations

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Loads environment configurations and backup storage settings
  # Critical for proper state file location and backup destinations
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load base config containing backup storage account details
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific state storage configurations
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # STATE OPERATION JOB
  # ===========================================================================
  # Performs the selected state management operation:
  # - UNLOCK: Removes stuck locks (use lock_id from error messages)
  # - BACKUP: Downloads current state and uploads to backup storage
  # - RESTORE: Downloads backup and overwrites current state
  #
  # SAFETY: Uses GitHub environment protection for production operations
  # ===========================================================================
  state-operation:
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment || 'staging' }}  # Triggers approval for production
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform only for unlock operations (others use Azure CLI directly)
      - name: Setup Terraform
        if: github.event.inputs.operation == 'unlock'
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment || 'staging' }}
          environment: ${{ github.event.inputs.environment || 'staging' }}
          
      # Authenticate with Azure for all state storage operations
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # UNLOCK OPERATION: Force remove Terraform state lock
      - name: Unlock state
        if: github.event.inputs.operation == 'unlock'
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          # Initialize Terraform to access the locked state
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
          # Force unlock using the provided lock ID
          terraform force-unlock ${{ github.event.inputs.lock_id }}
          
      # BACKUP OPERATION: Create timestamped backups of all state files
      - name: Backup state files
        if: github.event.inputs.operation == 'backup' || github.event_name == 'schedule'
        run: |
          # Create unique timestamp for this backup
          TIMESTAMP=$(date +%Y%m%d-%H%M%S)
          
          # Download and backup staging state file
          STAGING_CONFIG='${{ needs.load-config.outputs.staging-config }}'
          STAGING_BACKEND=$(echo $STAGING_CONFIG | jq -r '.terraform.backend')
          
          az storage blob download \
            --account-name $(echo $STAGING_BACKEND | jq -r '.storage_account_name') \
            --container-name $(echo $STAGING_BACKEND | jq -r '.container_name') \
            --name $(echo $STAGING_BACKEND | jq -r '.key') \
            --file staging-${TIMESTAMP}.tfstate
            
          # Download and backup production state file
          PRODUCTION_CONFIG='${{ needs.load-config.outputs.production-config }}'
          PRODUCTION_BACKEND=$(echo $PRODUCTION_CONFIG | jq -r '.terraform.backend')
          
          az storage blob download \
            --account-name $(echo $PRODUCTION_BACKEND | jq -r '.storage_account_name') \
            --container-name $(echo $PRODUCTION_BACKEND | jq -r '.container_name') \
            --name $(echo $PRODUCTION_BACKEND | jq -r '.key') \
            --file production-${TIMESTAMP}.tfstate
            
          # Upload both backups to the designated backup storage account
          BASE_CONFIG='${{ needs.load-config.outputs.base-config }}'
          BACKUP_STORAGE=$(echo $BASE_CONFIG | jq -r '.storage')
          
          az storage blob upload \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name staging-${TIMESTAMP}.tfstate \
            --file staging-${TIMESTAMP}.tfstate
            
          az storage blob upload \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name production-${TIMESTAMP}.tfstate \
            --file production-${TIMESTAMP}.tfstate
            
      # RESTORE OPERATION: Restore state file from backup (DANGEROUS!)
      - name: Restore state file
        if: github.event.inputs.operation == 'restore'
        run: |
          # Download the specified backup file
          BASE_CONFIG='${{ needs.load-config.outputs.base-config }}'
          BACKUP_STORAGE=$(echo $BASE_CONFIG | jq -r '.storage')
          
          az storage blob download \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name ${{ github.event.inputs.backup_name }} \
            --file restored.tfstate
            
          # Overwrite the current state file with the backup
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          az storage blob upload \
            --account-name $(echo $BACKEND_CONFIG | jq -r '.storage_account_name') \
            --container-name $(echo $BACKEND_CONFIG | jq -r '.container_name') \
            --name $(echo $BACKEND_CONFIG | jq -r '.key') \
            --file restored.tfstate \
            --overwrite
            
      # Notify team of successful state operation
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment || 'all' }}
          message: "State ${{ github.event.inputs.operation || 'backup' }} operation completed successfully"
          
      # Critical: Alert team immediately if state operation fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment || 'all' }}
          message: "State ${{ github.event.inputs.operation || 'backup' }} operation failed"
```

```github-actions-workflow
# =============================================================================
# TERRAFORM UTILITIES PIPELINE
# =============================================================================
# This workflow provides utility operations for Terraform management:
#
# 1. TFUPDATE: Automatically update Terraform provider versions
# 2. DEPENDENCY GRAPH: Generate visual dependency graphs of resources
# 3. TARGET OPERATIONS: Apply/destroy specific resources safely
#
# Key Features:
# - Automated provider updates with PR creation
# - Visual dependency mapping for troubleshooting
# - Surgical infrastructure changes via targeting
# - Parallel execution for efficiency
# - Smart triggering based on file changes
# =============================================================================

name: 'Terraform Utilities'

# =============================================================================
# TRIGGERS: Manual operations + automatic provider updates
# =============================================================================
on:
  workflow_dispatch:  # Manual execution with operation selection
    inputs:
      operation:
        description: 'Utility operation to perform'
        required: true
        type: choice
        options:
          - tfupdate        # Update provider versions
          - dependency-graph # Generate resource dependency graph
          - target-apply    # Apply changes to specific resources only
          - target-destroy  # Destroy specific resources only
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      target_resources:
        description: 'Target resources (comma-separated, required for target operations)'
        required: false
  push:
    paths: ['terraform/**']        # Auto-trigger tfupdate when Terraform files change
    branches: [develop, main]

# =============================================================================
# PERMISSIONS: Enhanced permissions for PR creation and resource management
# =============================================================================
permissions:
  id-token: write       # Required for OIDC authentication
  contents: read        # Read repository contents
  pull-requests: write  # Create PRs for automated updates

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Standard configuration loading for consistent environment settings
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration (Azure credentials, notification settings)
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific configurations
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # TFUPDATE JOB
  # ===========================================================================
  # Automatically updates Terraform provider versions and creates PRs
  # Runs on push events to catch outdated providers early
  # Uses matrix strategy to update both environments in parallel
  # ===========================================================================
  tfupdate:
    if: github.event.inputs.operation == 'tfupdate' || github.event_name == 'push'
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]  # Update both environments
    steps:
      - uses: actions/checkout@v4
      
      # Install tfupdate tool for automated provider updates
      - name: Install tfupdate
        run: |
          curl -fsSL https://github.com/minamijoyo/tfupdate/releases/download/v0.7.1/tfupdate_0.7.1_linux_amd64.tar.gz | tar -xz
          sudo mv tfupdate /usr/local/bin/
          
      # Update Azure provider to latest compatible version
      - name: Update Terraform providers
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Update azurerm provider within compatible version range
          tfupdate provider azurerm --version "> 3.0, < 4.0" .
          
      # Check if any files were modified by the update
      - name: Check for updates
        id: check-updates
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          if git diff --quiet; then
            echo "has_updates=false" >> $GITHUB_OUTPUT
          else
            echo "has_updates=true" >> $GITHUB_OUTPUT
          fi
          
      # Automatically create PR for provider updates (only on push events)
      - name: Create Pull Request
        if: steps.check-updates.outputs.has_updates == 'true' && github.event_name == 'push'
        uses: peter-evans/create-pull-request@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: "chore: update terraform providers for ${{ matrix.environment }}"
          title: "Update Terraform providers for ${{ matrix.environment }}"
          body: |
            Automated Terraform provider updates for ${{ matrix.environment }} environment.
            
            Changes made by tfupdate:
            - Updated provider versions to latest compatible versions
            
            Please review the changes before merging.
          branch: tfupdate-${{ matrix.environment }}-${{ github.run_number }}

  # ===========================================================================
  # DEPENDENCY GRAPH JOB
  # ===========================================================================
  # Generates visual dependency graphs showing resource relationships
  # Useful for understanding infrastructure dependencies and troubleshooting
  # Outputs both SVG and PNG formats for different use cases
  # ===========================================================================
  dependency-graph:
    if: github.event.inputs.operation == 'dependency-graph'
    runs-on: self-hosted
    needs: load-config
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for graph generation
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment }}
          environment: ${{ github.event.inputs.environment }}
          
      # Authenticate with Azure to access current state
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Initialize Terraform to access state and generate accurate graph
      - name: Configure backend
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          # Initialize with environment-specific backend
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Generate dependency graph in multiple formats
      - name: Generate dependency graph
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          # Generate visual dependency graph using Graphviz
          terraform graph | dot -Tsvg > dependency-graph-${{ github.event.inputs.environment }}.svg
          terraform graph | dot -Tpng > dependency-graph-${{ github.event.inputs.environment }}.png
          
      # Save generated graphs as downloadable artifacts
      - name: Upload dependency graph
        uses: actions/upload-artifact@v4
        with:
          name: dependency-graph-${{ github.event.inputs.environment }}
          path: |
            terraform/environments/${{ github.event.inputs.environment }}/dependency-graph-${{ github.event.inputs.environment }}.svg
            terraform/environments/${{ github.event.inputs.environment }}/dependency-graph-${{ github.event.inputs.environment }}.png
          retention-days: 30

  # ===========================================================================
  # TARGET OPERATIONS JOB
  # ===========================================================================
  # Performs surgical apply/destroy operations on specific resources
  # Useful for:
  # - Fixing individual resource issues without full deployment
  # - Testing changes on specific components
  # - Emergency resource removal
  #
  # SAFETY: Requires environment approval for production operations
  # ===========================================================================
  target-operations:
    if: github.event.inputs.operation == 'target-apply' || github.event.inputs.operation == 'target-destroy'
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment }}  # Triggers approval for production
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for targeted operations
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment }}
          environment: ${{ github.event.inputs.environment }}
          
      # Authenticate with Azure for resource operations
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Initialize Terraform backend for the target environment
      - name: Configure backend
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Convert comma-separated target list into Terraform target flags
      - name: Prepare target flags
        id: targets
        run: |
          TARGETS="${{ github.event.inputs.target_resources }}"
          TARGET_FLAGS=""
          IFS=',' read -ra TARGET_ARRAY <<< "$TARGETS"
          for target in "${TARGET_ARRAY[@]}"; do
            TARGET_FLAGS="$TARGET_FLAGS -target=$target"
          done
          echo "flags=$TARGET_FLAGS" >> $GITHUB_OUTPUT
          
      # Generate plan for the targeted operation (apply or destroy)
      - name: Plan targeted operation
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          if [ "${{ github.event.inputs.operation }}" = "target-destroy" ]; then
            # Plan destruction of targeted resources
            terraform plan -destroy ${{ steps.targets.outputs.flags }} -out=target-plan
          else
            # Plan application of targeted resources
            terraform plan ${{ steps.targets.outputs.flags }} -out=target-plan
          fi
          
      # Execute the targeted operation (requires manual approval for production)
      - name: Apply targeted operation
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: terraform apply target-plan
        
      # Notify team of successful targeted operation
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment }}
          message: "Target operation ${{ github.event.inputs.operation }} completed successfully for resources: ${{ github.event.inputs.target_resources }}"
          
      # Alert team if targeted operation fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment }}
          message: "Target operation ${{ github.event.inputs.operation }} failed for resources: ${{ github.event.inputs.target_resources }}"
```

```github-actions-workflow
# =============================================================================
# TERRAFORM COMPLIANCE PIPELINE
# =============================================================================
# This workflow provides comprehensive security and compliance validation for
# Terraform infrastructure using multiple scanning tools:
#
# 1. TFLINT: Terraform-specific linting and best practices
# 2. CHECKOV: Security vulnerability scanning and compliance checks
# 3. AZURE POLICY: Live policy compliance validation against Azure policies
#
# Key Features:
# - Multi-tool security scanning for comprehensive coverage
# - PR integration with automatic compliance reports
# - Azure Policy integration for live environment checking
# - Artifact storage for compliance audit trails
# - Teams notifications for compliance status
# =============================================================================

name: 'Terraform Compliance'

# =============================================================================
# TRIGGERS: PR validation + manual compliance checks
# =============================================================================
on:
  pull_request:
    branches: [main, develop]      # Run compliance checks on all PRs
    paths: ['terraform/**']        # Only when Terraform files are modified
  workflow_dispatch:               # Manual execution for compliance audits
    inputs:
      environment:
        description: 'Environment to check'
        required: true
        type: choice
        options:
          - staging
          - production
          - all                    # Check all environments

# =============================================================================
# PERMISSIONS: Enhanced permissions for PR comments and Azure access
# =============================================================================
permissions:
  id-token: write       # Required for OIDC authentication
  contents: read        # Read repository contents and configurations
  pull-requests: write  # Comment on PRs with compliance results

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Standard configuration loading for consistent environment settings
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration for Azure access and notifications
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific configurations for backend access
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # COMPLIANCE CHECK JOB
  # ===========================================================================
  # Performs comprehensive security and compliance scanning using:
  # - TFLint for Terraform best practices and syntax issues
  # - Checkov for security vulnerabilities and misconfigurations
  # - Azure Policy for live environment policy compliance
  #
  # Uses matrix strategy to check multiple environments in parallel
  # ===========================================================================
  compliance-check:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        # Dynamic environment selection based on input or default to both
        environment: ${{ github.event.inputs.environment == 'all' && fromJson('["staging", "production"]') || github.event.inputs.environment && fromJson(format('["{0}"]', github.event.inputs.environment)) || fromJson('["staging", "production"]') }}
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for plan generation and validation
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      # Authenticate with Azure for policy compliance checks
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Initialize Terraform to generate plan for compliance scanning
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Initialize with environment-specific backend
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Generate plan for compliance analysis (in JSON format for tools)
      - name: Generate plan for compliance check
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -out=compliance-plan-${{ matrix.environment }}
          # Convert plan to JSON for security scanning tools
          terraform show -json compliance-plan-${{ matrix.environment }} > compliance-plan-${{ matrix.environment }}.json
          
      # Install and run TFLint for Terraform best practices
      - name: Install tflint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
          
      - name: Run tflint security checks
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Initialize TFLint with latest rule definitions
          tflint --init
          # Run TFLint and generate JSON report (continue on failures for reporting)
          tflint --format json > tflint-report-${{ matrix.environment }}.json || true
          
      # Install and run Checkov for security vulnerability scanning
      - name: Install checkov
        run: |
          pip3 install checkov
          
      - name: Run checkov security scan
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Scan the Terraform plan for security issues and misconfigurations
          checkov -f compliance-plan-${{ matrix.environment }}.json --framework terraform_plan --output json > checkov-report-${{ matrix.environment }}.json || true
          
      # Check live Azure Policy compliance for existing resources
      - name: Check Azure Policy compliance
        run: |
          echo "Checking Azure Policy compliance for ${{ matrix.environment }} environment..."
          
          # Query resource groups that match the environment naming pattern
          RESOURCE_GROUPS=$(az group list --query "[?contains(name, '${{ matrix.environment }}')].name" -o tsv)
          
          # Initialize compliance report structure
          echo '{"environment": "${{ matrix.environment }}", "policy_states": []}' > policy-compliance-${{ matrix.environment }}.json
          
          # Check policy compliance for each resource group
          for rg in $RESOURCE_GROUPS; do
            echo "Checking resource group: $rg"
            # Query policy compliance state and append to temporary file
            az policy state list --resource-group "$rg" --query '[].{policyDefinitionName: policyDefinitionName, complianceState: complianceState, resourceId: resourceId}' >> policy-states-temp.json || true
          done
          
          # Merge policy states into main compliance report
          if [ -f policy-states-temp.json ]; then
            jq '.policy_states += input' policy-compliance-${{ matrix.environment }}.json policy-states-temp.json > temp.json
            mv temp.json policy-compliance-${{ matrix.environment }}.json
            rm policy-states-temp.json
          fi
          
      # Generate comprehensive compliance summary report
      - name: Generate compliance summary
        run: |
          # Create markdown report combining all compliance check results
          cat > compliance-summary-${{ matrix.environment }}.md << EOF
          # Compliance Report - ${{ matrix.environment }} Environment
          
          ## TFLint Security Checks
          \`\`\`json
          $(cat terraform/environments/${{ matrix.environment }}/tflint-report-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Checkov Security Scan
          \`\`\`json
          $(cat terraform/environments/${{ matrix.environment }}/checkov-report-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Azure Policy Compliance
          \`\`\`json
          $(cat policy-compliance-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Summary
          - **Environment**: ${{ matrix.environment }}
          - **Scan Date**: $(date)
          - **Repository**: ${{ github.repository }}
          - **Commit**: ${{ github.sha }}
          EOF
          
      # Store all compliance reports as artifacts for audit trail
      - name: Upload compliance reports
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report-${{ matrix.environment }}
          path: |
            terraform/environments/${{ matrix.environment }}/tflint-report-${{ matrix.environment }}.json
            terraform/environments/${{ matrix.environment }}/checkov-report-${{ matrix.environment }}.json
            policy-compliance-${{ matrix.environment }}.json
            compliance-summary-${{ matrix.environment }}.md
          retention-days: 30
          
      # Post compliance summary as PR comment for visibility
      - name: Comment PR with compliance results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            try {
              // Read the generated compliance summary
              const summary = fs.readFileSync('compliance-summary-${{ matrix.environment }}.md', 'utf8');
              // Post as comment on the PR
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: summary
              });
            } catch (error) {
              console.log('Could not read compliance summary file');
            }
            
      # Notify team of compliance check completion
      - name: Notify compliance results
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ matrix.environment }}
          message: "Compliance check completed for ${{ matrix.environment }}. Check artifacts for detailed results."
```

```github-actions-workflow
# =============================================================================
# TERRAFORM TESTING PIPELINE
# =============================================================================
# This workflow provides automated infrastructure testing using Terratest,
# a Go testing framework specifically designed for testing infrastructure code.
#
# Testing Capabilities:
# 1. BASIC TESTS: Simple resource validation and output verification
# 2. INTEGRATION TESTS: End-to-end infrastructure testing
# 3. ALL TESTS: Comprehensive test suite execution
#
# Key Features:
# - Go-based infrastructure testing with Terratest
# - Automated test execution on code changes
# - Test result artifacts and reporting
# - Environment-specific test execution
# - Automatic cleanup of test resources
# =============================================================================

name: 'Terraform Testing'

# =============================================================================
# TRIGGERS: Manual execution + automatic testing on changes
# =============================================================================
on:
  workflow_dispatch:  # Manual execution with test suite selection
    inputs:
      test_suite:
        description: 'Test suite to run'
        required: true
        type: choice
        options:
          - basic       # Basic resource validation tests
          - integration # End-to-end integration tests
          - all         # Run complete test suite
      environment:
        description: 'Environment to test against'
        required: true
        type: choice
        options:
          - staging     # Run tests against staging environment
          - production  # Run tests against production (use carefully!)
  push:
    paths: ['tests/**', 'terraform/**']  # Auto-trigger when test or Terraform files change
    branches: [develop, main]

# =============================================================================
# PERMISSIONS: Required for Azure access and test execution
# =============================================================================
permissions:
  id-token: write  # Required for OIDC authentication
  contents: read   # Read repository contents and test files

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Standard configuration loading for consistent environment settings
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration for Azure access and notifications
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific configurations for testing
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # TERRAFORM TESTS JOB
  # ===========================================================================
  # Executes Go-based infrastructure tests using Terratest framework
  # Tests deploy real infrastructure, validate it, then clean up
  #
  # IMPORTANT: These tests create real Azure resources and incur costs
  # Always ensure proper cleanup and use staging environment for testing
  # ===========================================================================
  terraform-tests:
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment || 'staging' }}  # Requires approval for production testing
    steps:
      - uses: actions/checkout@v4
      
      # Setup Go runtime for Terratest execution
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'  # Use Go version compatible with Terratest
          
      # Setup Terraform for infrastructure deployment during tests
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment || 'staging' }}
          environment: ${{ github.event.inputs.environment || 'staging' }}
          
      # Authenticate with Azure for test resource creation
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
          subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
          client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
          
      # Install Go dependencies for Terratest
      - name: Install Go dependencies
        working-directory: tests/terratest
        run: |
          # Download all required Go modules
          go mod download
          # Ensure go.mod and go.sum are properly synchronized
          go mod tidy
          
      # Execute basic infrastructure validation tests
      - name: Run basic tests
        if: github.event.inputs.test_suite == 'basic' || github.event.inputs.test_suite == 'all' || github.event_name == 'push'
        working-directory: tests/terratest
        run: |
          # Run basic test suite with extended timeout for Azure resource operations
          go test -v -timeout 30m -run TestTerraformBasicExample
          
      # Execute comprehensive integration tests (if requested)
      - name: Run integration tests
        if: github.event.inputs.test_suite == 'integration' || github.event.inputs.test_suite == 'all'
        working-directory: tests/terratest
        run: |
          echo "Integration tests would run here"
          echo "Add your integration test functions to basic_test.go or create separate test files"
          # Example: go test -v -timeout 60m -run TestTerraformIntegration
          
      # Generate JSON test results for analysis and reporting
      - name: Generate test report
        if: always()  # Generate report even if tests fail for debugging
        working-directory: tests/terratest
        run: |
          # Generate machine-readable test results in JSON format
          go test -v -json > test-results.json || true
          
      # Store test results as artifacts for review and debugging
      - name: Upload test results
        if: always()  # Always upload results for analysis
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ github.event.inputs.environment || 'staging' }}
          path: tests/terratest/test-results.json
          retention-days: 30
          
      # Notify team of successful test completion
      - name: Notify test success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment || 'staging' }}
          message: "Terraform tests completed successfully for ${{ github.event.inputs.test_suite || 'basic' }} test suite"
          
      # Alert team immediately if tests fail (may indicate infrastructure issues)
      - name: Notify test failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment || 'staging' }}
          message: "Terraform tests failed for ${{ github.event.inputs.test_suite || 'basic' }} test suite"
```

Perfect! I have successfully added comprehensive comments to all the GitHub Actions pipeline YAML files. Here's a summary of the enhancements made:

## 📝 **Comment Enhancements Added:**

### **1. terraform-main.yml**
- **Header section** explaining the complete deployment lifecycle
- **Trigger explanations** for GitFlow branching strategy
- **Job-level documentation** for each deployment phase
- **Step-by-step comments** explaining each operation
- **Safety annotations** for approval gates and production deployment

### **2. terraform-drift-detection.yml**
- **Purpose and features** explanation at the top
- **Exit code interpretation** for terraform plan results
- **Timing rationale** for weekly scheduling
- **Non-destructive nature** clarification

### **3. terraform-state-management.yml**
- **Critical safety warnings** about state manipulation
- **Operation-specific explanations** (unlock, backup, restore)
- **Risk assessments** and safety recommendations
- **Backend configuration details**

### **4. terraform-utilities.yml**
- **Multi-function workflow** explanation
- **Provider update automation** with PR creation
- **Dependency graph generation** use cases
- **Target operations** for surgical changes
- **Parallel execution benefits**

### **5. terraform-compliance.yml**
- **Multi-tool scanning** approach explanation
- **Security tool descriptions** (TFLint, Checkov, Azure Policy)
- **PR integration** and automated reporting
- **Compliance audit trail** features

### **6. terraform-testing.yml**
- **Terratest framework** introduction
- **Real infrastructure testing** warnings about costs
- **Test suite types** and their purposes
- **Go testing workflow** explanation
- **Cleanup automation** importance

## 🎯 **Key Benefits for First-Time Users:**

✅ **Clear Purpose**: Each workflow's primary function is immediately obvious
✅ **Safety Awareness**: Critical operations have prominent safety warnings
✅ **Usage Guidance**: Step-by-step explanations of what each section does
✅ **Trigger Understanding**: When and why workflows execute
✅ **Best Practices**: Embedded recommendations and considerations
✅ **Troubleshooting**: Context for debugging and understanding failures

The pipelines are now much more accessible to developers who are new to the codebase, with comprehensive documentation that explains both the "what" and "why" of each operation!

---

**User**

can you help me change the way of passing the "tenant_id", "subscription_id" and "client_id" into the pipelines? instead of passing them from the base.json file, I want them to be stored and managed using Github Secrets

---

**Cursor**

[MODE: EXECUTE]

I'll help you change the authentication approach to use GitHub Secrets instead of storing sensitive values in the base.json file. This is a much better security practice! Let me update all the workflow files and configuration.

```github-actions-workflow
# =============================================================================
# TERRAFORM MAIN DEPLOYMENT PIPELINE
# =============================================================================
# This workflow handles the complete deployment lifecycle for both staging and
# production environments with automated validation, compliance checks, and
# approval gates. It implements GitFlow branching strategy where:
# - develop branch deploys to staging
# - main branch deploys to production (after staging success)
# =============================================================================

name: 'Terraform Main Deployment'

# =============================================================================
# TRIGGERS: When this workflow runs
# =============================================================================
on:
  push:
    branches: [develop, main]  # Auto-trigger on push to these branches
    paths: ['terraform/**', 'config/**', '.github/workflows/**']  # Only when relevant files change
  pull_request:
    branches: [main]  # Run validation on PRs to main branch
    paths: ['terraform/**', 'config/**', '.github/workflows/**']
  workflow_dispatch:  # Allow manual triggering with environment selection
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

# =============================================================================
# PERMISSIONS: OIDC requires specific permissions for Azure authentication
# =============================================================================
permissions:
  id-token: write      # Required for OIDC token generation
  contents: read       # Read repository contents
  pull-requests: write # Comment on PRs with results

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Loads environment-specific configurations from JSON files and makes them
  # available to other jobs. This centralizes configuration management.
  # Note: Azure authentication credentials are now managed via GitHub Secrets
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration (Teams webhook, Terraform versions, etc.)
      # Note: Azure credentials are now securely managed via GitHub Secrets
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load staging-specific configuration (backend, deployment settings)
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load production-specific configuration
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # VALIDATION JOB
  # ===========================================================================
  # Runs in parallel for both environments to:
  # 1. Validate Terraform syntax and formatting
  # 2. Generate execution plans
  # 3. Cache plans for later use in deployment
  # Uses matrix strategy for parallel execution efficiency
  # ===========================================================================
  validation:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]  # Run validation for both environments in parallel
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform with caching to improve performance
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      # Authenticate with Azure using OIDC (credentials from GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Ensure Terraform code follows formatting standards
      - name: Terraform Format Check
        working-directory: terraform/environments/${{ matrix.environment }}
        run: terraform fmt -check -recursive
        
      # Validate Terraform syntax and configuration
      - name: Terraform Lint
        working-directory: terraform/environments/${{ matrix.environment }}
        run: terraform validate
        
      # Configure remote state backend dynamically based on environment
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Select appropriate config based on current environment
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          # Initialize Terraform with environment-specific backend configuration
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Generate execution plan and save it for later use in deployment
      - name: Terraform Plan
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -out=tfplan-${{ matrix.environment }}
          
      # Store plan as artifact for use in deployment jobs (avoids re-planning)
      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}
          path: terraform/environments/${{ matrix.environment }}/tfplan-${{ matrix.environment }}
          retention-days: 30

  # ===========================================================================
  # COMPLIANCE SCANNING JOB
  # ===========================================================================
  # Runs security and policy compliance checks in parallel with validation
  # to catch issues early. Includes Azure Policy validation.
  # ===========================================================================
  compliance:
    runs-on: self-hosted
    needs: [load-config, validation]
    strategy:
      matrix:
        environment: [staging, production]  # Check compliance for both environments
    steps:
      - uses: actions/checkout@v4
      
      # Authenticate with Azure to check policy compliance (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Check existing resources against Azure Policy definitions
      - name: Run Azure Policy Compliance Check
        run: |
          echo "Running Azure Policy compliance check for ${{ matrix.environment }}"
          # Query policy compliance state for environment-specific resource groups
          # This helps identify policy violations before deployment
          az policy state list --resource-group "rg-${{ matrix.environment }}" --all || true

  # ===========================================================================
  # STAGING DEPLOYMENT JOB
  # ===========================================================================
  # Deploys to staging environment with the following safeguards:
  # 1. Only runs for develop branch or manual dispatch
  # 2. Requires manual approval (configured in GitHub environment settings)
  # 3. Uses pre-generated plan from validation job
  # 4. Sends Teams notifications for visibility
  # ===========================================================================
  deploy-staging:
    # Only deploy to staging from develop branch or manual trigger
    if: github.ref == 'refs/heads/develop' || github.event_name == 'workflow_dispatch'
    runs-on: self-hosted
    needs: [load-config, validation, compliance]
    environment: staging  # Triggers GitHub environment protection rules and manual approval
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform with same configuration as validation
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/staging
          environment: staging
          
      # Authenticate with Azure for deployment operations (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Download the pre-generated plan from validation job
      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-staging
          path: terraform/environments/staging
          
      # Reinitialize Terraform backend for deployment context
      - name: Configure backend
        working-directory: terraform/environments/staging
        run: |
          BACKEND_CONFIG=$(echo '${{ needs.load-config.outputs.staging-config }}' | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Apply the pre-approved plan (no additional confirmation needed)
      - name: Terraform Apply
        working-directory: terraform/environments/staging
        run: terraform apply tfplan-staging
        
      # Notify team of successful staging deployment
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: staging
          message: "Staging deployment completed successfully"
          
      # Notify team if staging deployment fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: staging
          message: "Staging deployment failed"

  # ===========================================================================
  # PRODUCTION DEPLOYMENT JOB
  # ===========================================================================
  # Deploys to production with maximum safety:
  # 1. Only runs for main branch or manual production dispatch
  # 2. Requires successful staging deployment first
  # 3. Requires manual approval via GitHub environment protection
  # 4. Uses pre-generated and validated plan
  # 5. Full notification coverage for audit trail
  # ===========================================================================
  deploy-production:
    # Production deployment conditions: main branch OR manual production trigger
    if: github.ref == 'refs/heads/main' || (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'production')
    runs-on: self-hosted
    needs: [load-config, validation, compliance, deploy-staging]  # Must wait for staging success
    environment: production  # Triggers GitHub environment protection rules (manual approval)
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for production deployment
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/production
          environment: production
          
      # Authenticate with Azure using production credentials (from GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Download the production plan generated during validation
      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-production
          path: terraform/environments/production
          
      # Configure production backend
      - name: Configure backend
        working-directory: terraform/environments/production
        run: |
          BACKEND_CONFIG=$(echo '${{ needs.load-config.outputs.production-config }}' | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Apply to production (this step requires manual approval)
      - name: Terraform Apply
        working-directory: terraform/environments/production
        run: terraform apply tfplan-production
        
      # Notify team of successful production deployment
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: production
          message: "Production deployment completed successfully"
          
      # Critical: Notify team immediately if production deployment fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: production
          message: "Production deployment failed"
```

```github-actions-workflow
# =============================================================================
# TERRAFORM DRIFT DETECTION PIPELINE
# =============================================================================
# This workflow automatically detects infrastructure drift by comparing the
# current state of resources in Azure with the expected state defined in
# Terraform configurations. It runs weekly but can also be triggered manually.
#
# Key Features:
# - Automated weekly execution (Sundays at 9 AM UTC)
# - Parallel drift detection for both staging and production
# - Detailed drift reports saved as artifacts
# - Teams notifications for drift alerts
# - Non-destructive - only detects changes, doesn't fix them
# =============================================================================

name: 'Terraform Drift Detection'

# =============================================================================
# TRIGGERS: Scheduled weekly runs + manual execution
# =============================================================================
on:
  schedule:
    - cron: '0 9 * * 0'  # Weekly on Sundays at 9 AM UTC (adjust timezone as needed)
  workflow_dispatch:      # Allow manual triggering for immediate drift checks

# =============================================================================
# PERMISSIONS: Required for Azure OIDC authentication
# =============================================================================
permissions:
  id-token: write  # Required for OIDC token generation
  contents: read   # Read repository contents and configurations

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Same configuration loading pattern as main deployment workflow
  # to ensure consistent environment settings across all pipelines
  # Note: Azure authentication credentials are now managed via GitHub Secrets
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration (notification settings, Terraform versions)
      # Note: Azure credentials are now securely managed via GitHub Secrets
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific backend configurations
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # DRIFT DETECTION JOB
  # ===========================================================================
  # Runs terraform plan with -detailed-exitcode to detect any differences
  # between current infrastructure state and Terraform configuration.
  # Uses matrix strategy to check both environments in parallel.
  # ===========================================================================
  drift-detection:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]  # Check both environments simultaneously
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform with caching for performance
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      # Authenticate with Azure to access current resource state (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Initialize Terraform with the appropriate backend for this environment
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Select appropriate config based on current environment
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          # Initialize with environment-specific state backend
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Run terraform plan with detailed exit codes to detect drift
      - name: Check for drift
        id: drift
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Run terraform plan with detailed exit codes:
          # 0 = no changes, 1 = error, 2 = changes detected (drift)
          terraform plan -detailed-exitcode -out=drift-plan-${{ matrix.environment }} > drift-output-${{ matrix.environment }}.txt 2>&1
          exit_code=$?
          echo "exit_code=$exit_code" >> $GITHUB_OUTPUT
          
          # Interpret the exit code and set drift flag
          if [ $exit_code -eq 2 ]; then
            echo "has_drift=true" >> $GITHUB_OUTPUT
            echo "Drift detected in ${{ matrix.environment }} environment"
          else
            echo "has_drift=false" >> $GITHUB_OUTPUT
            echo "No drift detected in ${{ matrix.environment }} environment"
          fi
          
      # Save drift analysis reports as artifacts for review
      - name: Upload drift report
        if: steps.drift.outputs.has_drift == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: drift-report-${{ matrix.environment }}
          path: |
            terraform/environments/${{ matrix.environment }}/drift-output-${{ matrix.environment }}.txt
            terraform/environments/${{ matrix.environment }}/drift-plan-${{ matrix.environment }}
          retention-days: 30
          
      # Alert team via Teams when drift is detected (requires immediate attention)
      - name: Notify drift detected
        if: steps.drift.outputs.has_drift == 'true'
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: warning
          environment: ${{ matrix.environment }}
          message: "Infrastructure drift detected! Please review the drift report artifact."
          
      # Send confirmation when no drift is found (good news!)
      - name: Notify no drift
        if: steps.drift.outputs.has_drift == 'false'
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ matrix.environment }}
          message: "No infrastructure drift detected."
```

```github-actions-workflow
# =============================================================================
# TERRAFORM STATE MANAGEMENT PIPELINE
# =============================================================================
# This workflow provides critical state management operations for Terraform:
#
# 1. UNLOCK OPERATION: Force-unlock stuck Terraform state files
# 2. BACKUP OPERATION: Create timestamped backups of state files
# 3. RESTORE OPERATION: Restore state files from previous backups
#
# CRITICAL SAFETY NOTES:
# - These operations directly manipulate Terraform state files
# - Improper use can cause data loss or infrastructure corruption
# - Always ensure you have recent backups before performing operations
# - Test restore operations in non-production environments first
# =============================================================================

name: 'Terraform State Management'

# =============================================================================
# TRIGGERS: Manual operations + automated weekly backups
# =============================================================================
on:
  workflow_dispatch:  # Manual execution with operation selection
    inputs:
      operation:
        description: 'State operation to perform'
        required: true
        type: choice
        options:
          - unlock    # Force unlock stuck state file
          - backup    # Create manual backup of state files
          - restore   # Restore state from backup
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      lock_id:
        description: 'Lock ID (required for unlock operation)'
        required: false
      backup_name:
        description: 'Backup name (required for restore operation)'
        required: false
  schedule:
    - cron: '0 2 * * 0'  # Weekly backup on Sundays at 2 AM UTC (off-peak hours)

# =============================================================================
# PERMISSIONS: Enhanced permissions needed for state operations
# =============================================================================
permissions:
  id-token: write  # Required for OIDC authentication
  contents: read   # Read repository contents and configurations

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Loads environment configurations and backup storage settings
  # Critical for proper state file location and backup destinations
  # Note: Azure authentication credentials are now managed via GitHub Secrets
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load base config containing backup storage account details and notification settings
      # Note: Azure credentials are now securely managed via GitHub Secrets
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific state storage configurations
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # STATE OPERATION JOB
  # ===========================================================================
  # Performs the selected state management operation:
  # - UNLOCK: Removes stuck locks (use lock_id from error messages)
  # - BACKUP: Downloads current state and uploads to backup storage
  # - RESTORE: Downloads backup and overwrites current state
  #
  # SAFETY: Uses GitHub environment protection for production operations
  # ===========================================================================
  state-operation:
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment || 'staging' }}  # Triggers approval for production
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform only for unlock operations (others use Azure CLI directly)
      - name: Setup Terraform
        if: github.event.inputs.operation == 'unlock'
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment || 'staging' }}
          environment: ${{ github.event.inputs.environment || 'staging' }}
          
      # Authenticate with Azure for all state storage operations (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # UNLOCK OPERATION: Force remove Terraform state lock
      - name: Unlock state
        if: github.event.inputs.operation == 'unlock'
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          # Initialize Terraform to access the locked state
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
          # Force unlock using the provided lock ID
          terraform force-unlock ${{ github.event.inputs.lock_id }}
          
      # BACKUP OPERATION: Create timestamped backups of all state files
      - name: Backup state files
        if: github.event.inputs.operation == 'backup' || github.event_name == 'schedule'
        run: |
          # Create unique timestamp for this backup
          TIMESTAMP=$(date +%Y%m%d-%H%M%S)
          
          # Download and backup staging state file
          STAGING_CONFIG='${{ needs.load-config.outputs.staging-config }}'
          STAGING_BACKEND=$(echo $STAGING_CONFIG | jq -r '.terraform.backend')
          
          az storage blob download \
            --account-name $(echo $STAGING_BACKEND | jq -r '.storage_account_name') \
            --container-name $(echo $STAGING_BACKEND | jq -r '.container_name') \
            --name $(echo $STAGING_BACKEND | jq -r '.key') \
            --file staging-${TIMESTAMP}.tfstate
            
          # Download and backup production state file
          PRODUCTION_CONFIG='${{ needs.load-config.outputs.production-config }}'
          PRODUCTION_BACKEND=$(echo $PRODUCTION_CONFIG | jq -r '.terraform.backend')
          
          az storage blob download \
            --account-name $(echo $PRODUCTION_BACKEND | jq -r '.storage_account_name') \
            --container-name $(echo $PRODUCTION_BACKEND | jq -r '.container_name') \
            --name $(echo $PRODUCTION_BACKEND | jq -r '.key') \
            --file production-${TIMESTAMP}.tfstate
            
          # Upload both backups to the designated backup storage account
          BASE_CONFIG='${{ needs.load-config.outputs.base-config }}'
          BACKUP_STORAGE=$(echo $BASE_CONFIG | jq -r '.storage')
          
          az storage blob upload \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name staging-${TIMESTAMP}.tfstate \
            --file staging-${TIMESTAMP}.tfstate
            
          az storage blob upload \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name production-${TIMESTAMP}.tfstate \
            --file production-${TIMESTAMP}.tfstate
            
      # RESTORE OPERATION: Restore state file from backup (DANGEROUS!)
      - name: Restore state file
        if: github.event.inputs.operation == 'restore'
        run: |
          # Download the specified backup file
          BASE_CONFIG='${{ needs.load-config.outputs.base-config }}'
          BACKUP_STORAGE=$(echo $BASE_CONFIG | jq -r '.storage')
          
          az storage blob download \
            --account-name $(echo $BACKUP_STORAGE | jq -r '.backup_account') \
            --container-name $(echo $BACKUP_STORAGE | jq -r '.backup_container') \
            --name ${{ github.event.inputs.backup_name }} \
            --file restored.tfstate
            
          # Overwrite the current state file with the backup
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          az storage blob upload \
            --account-name $(echo $BACKEND_CONFIG | jq -r '.storage_account_name') \
            --container-name $(echo $BACKEND_CONFIG | jq -r '.container_name') \
            --name $(echo $BACKEND_CONFIG | jq -r '.key') \
            --file restored.tfstate \
            --overwrite
            
      # Notify team of successful state operation
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment || 'all' }}
          message: "State ${{ github.event.inputs.operation || 'backup' }} operation completed successfully"
          
      # Critical: Alert team immediately if state operation fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment || 'all' }}
          message: "State ${{ github.event.inputs.operation || 'backup' }} operation failed"
```

```github-actions-workflow
# =============================================================================
# TERRAFORM UTILITIES PIPELINE
# =============================================================================
# This workflow provides utility operations for Terraform management:
#
# 1. TFUPDATE: Automatically update Terraform provider versions
# 2. DEPENDENCY GRAPH: Generate visual dependency graphs of resources
# 3. TARGET OPERATIONS: Apply/destroy specific resources safely
#
# Key Features:
# - Automated provider updates with PR creation
# - Visual dependency mapping for troubleshooting
# - Surgical infrastructure changes via targeting
# - Parallel execution for efficiency
# - Smart triggering based on file changes
# =============================================================================

name: 'Terraform Utilities'

# =============================================================================
# TRIGGERS: Manual operations + automatic provider updates
# =============================================================================
on:
  workflow_dispatch:  # Manual execution with operation selection
    inputs:
      operation:
        description: 'Utility operation to perform'
        required: true
        type: choice
        options:
          - tfupdate        # Update provider versions
          - dependency-graph # Generate resource dependency graph
          - target-apply    # Apply changes to specific resources only
          - target-destroy  # Destroy specific resources only
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      target_resources:
        description: 'Target resources (comma-separated, required for target operations)'
        required: false
  push:
    paths: ['terraform/**']        # Auto-trigger tfupdate when Terraform files change
    branches: [develop, main]

# =============================================================================
# PERMISSIONS: Enhanced permissions for PR creation and resource management
# =============================================================================
permissions:
  id-token: write       # Required for OIDC authentication
  contents: read        # Read repository contents
  pull-requests: write  # Create PRs for automated updates

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Standard configuration loading for consistent environment settings
  # Note: Azure authentication credentials are now managed via GitHub Secrets
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration (notification settings, Terraform versions)
      # Note: Azure credentials are now securely managed via GitHub Secrets
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific configurations
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # TFUPDATE JOB
  # ===========================================================================
  # Automatically updates Terraform provider versions and creates PRs
  # Runs on push events to catch outdated providers early
  # Uses matrix strategy to update both environments in parallel
  # ===========================================================================
  tfupdate:
    if: github.event.inputs.operation == 'tfupdate' || github.event_name == 'push'
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        environment: [staging, production]  # Update both environments
    steps:
      - uses: actions/checkout@v4
      
      # Install tfupdate tool for automated provider updates
      - name: Install tfupdate
        run: |
          curl -fsSL https://github.com/minamijoyo/tfupdate/releases/download/v0.7.1/tfupdate_0.7.1_linux_amd64.tar.gz | tar -xz
          sudo mv tfupdate /usr/local/bin/
          
      # Update Azure provider to latest compatible version
      - name: Update Terraform providers
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Update azurerm provider within compatible version range
          tfupdate provider azurerm --version "> 3.0, < 4.0" .
          
      # Check if any files were modified by the update
      - name: Check for updates
        id: check-updates
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          if git diff --quiet; then
            echo "has_updates=false" >> $GITHUB_OUTPUT
          else
            echo "has_updates=true" >> $GITHUB_OUTPUT
          fi
          
      # Automatically create PR for provider updates (only on push events)
      - name: Create Pull Request
        if: steps.check-updates.outputs.has_updates == 'true' && github.event_name == 'push'
        uses: peter-evans/create-pull-request@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: "chore: update terraform providers for ${{ matrix.environment }}"
          title: "Update Terraform providers for ${{ matrix.environment }}"
          body: |
            Automated Terraform provider updates for ${{ matrix.environment }} environment.
            
            Changes made by tfupdate:
            - Updated provider versions to latest compatible versions
            
            Please review the changes before merging.
          branch: tfupdate-${{ matrix.environment }}-${{ github.run_number }}

  # ===========================================================================
  # DEPENDENCY GRAPH JOB
  # ===========================================================================
  # Generates visual dependency graphs showing resource relationships
  # Useful for understanding infrastructure dependencies and troubleshooting
  # Outputs both SVG and PNG formats for different use cases
  # ===========================================================================
  dependency-graph:
    if: github.event.inputs.operation == 'dependency-graph'
    runs-on: self-hosted
    needs: load-config
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for graph generation
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment }}
          environment: ${{ github.event.inputs.environment }}
          
      # Authenticate with Azure to access current state (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Initialize Terraform to access state and generate accurate graph
      - name: Configure backend
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          # Initialize with environment-specific backend
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Generate dependency graph in multiple formats
      - name: Generate dependency graph
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          # Generate visual dependency graph using Graphviz
          terraform graph | dot -Tsvg > dependency-graph-${{ github.event.inputs.environment }}.svg
          terraform graph | dot -Tpng > dependency-graph-${{ github.event.inputs.environment }}.png
          
      # Save generated graphs as downloadable artifacts
      - name: Upload dependency graph
        uses: actions/upload-artifact@v4
        with:
          name: dependency-graph-${{ github.event.inputs.environment }}
          path: |
            terraform/environments/${{ github.event.inputs.environment }}/dependency-graph-${{ github.event.inputs.environment }}.svg
            terraform/environments/${{ github.event.inputs.environment }}/dependency-graph-${{ github.event.inputs.environment }}.png
          retention-days: 30

  # ===========================================================================
  # TARGET OPERATIONS JOB
  # ===========================================================================
  # Performs surgical apply/destroy operations on specific resources
  # Useful for:
  # - Fixing individual resource issues without full deployment
  # - Testing changes on specific components
  # - Emergency resource removal
  #
  # SAFETY: Requires environment approval for production operations
  # ===========================================================================
  target-operations:
    if: github.event.inputs.operation == 'target-apply' || github.event.inputs.operation == 'target-destroy'
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment }}  # Triggers approval for production
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for targeted operations
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment }}
          environment: ${{ github.event.inputs.environment }}
          
      # Authenticate with Azure for resource operations (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Initialize Terraform backend for the target environment
      - name: Configure backend
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          ENV_CONFIG='${{ github.event.inputs.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Convert comma-separated target list into Terraform target flags
      - name: Prepare target flags
        id: targets
        run: |
          TARGETS="${{ github.event.inputs.target_resources }}"
          TARGET_FLAGS=""
          IFS=',' read -ra TARGET_ARRAY <<< "$TARGETS"
          for target in "${TARGET_ARRAY[@]}"; do
            TARGET_FLAGS="$TARGET_FLAGS -target=$target"
          done
          echo "flags=$TARGET_FLAGS" >> $GITHUB_OUTPUT
          
      # Generate plan for the targeted operation (apply or destroy)
      - name: Plan targeted operation
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: |
          if [ "${{ github.event.inputs.operation }}" = "target-destroy" ]; then
            # Plan destruction of targeted resources
            terraform plan -destroy ${{ steps.targets.outputs.flags }} -out=target-plan
          else
            # Plan application of targeted resources
            terraform plan ${{ steps.targets.outputs.flags }} -out=target-plan
          fi
          
      # Execute the targeted operation (requires manual approval for production)
      - name: Apply targeted operation
        working-directory: terraform/environments/${{ github.event.inputs.environment }}
        run: terraform apply target-plan
        
      # Notify team of successful targeted operation
      - name: Notify success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment }}
          message: "Target operation ${{ github.event.inputs.operation }} completed successfully for resources: ${{ github.event.inputs.target_resources }}"
          
      # Alert team if targeted operation fails
      - name: Notify failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment }}
          message: "Target operation ${{ github.event.inputs.operation }} failed for resources: ${{ github.event.inputs.target_resources }}"
```

```github-actions-workflow
# =============================================================================
# TERRAFORM COMPLIANCE PIPELINE
# =============================================================================
# This workflow provides comprehensive security and compliance validation for
# Terraform infrastructure using multiple scanning tools:
#
# 1. TFLINT: Terraform-specific linting and best practices
# 2. CHECKOV: Security vulnerability scanning and compliance checks
# 3. AZURE POLICY: Live policy compliance validation against Azure policies
#
# Key Features:
# - Multi-tool security scanning for comprehensive coverage
# - PR integration with automatic compliance reports
# - Azure Policy integration for live environment checking
# - Artifact storage for compliance audit trails
# - Teams notifications for compliance status
# =============================================================================

name: 'Terraform Compliance'

# =============================================================================
# TRIGGERS: PR validation + manual compliance checks
# =============================================================================
on:
  pull_request:
    branches: [main, develop]      # Run compliance checks on all PRs
    paths: ['terraform/**']        # Only when Terraform files are modified
  workflow_dispatch:               # Manual execution for compliance audits
    inputs:
      environment:
        description: 'Environment to check'
        required: true
        type: choice
        options:
          - staging
          - production
          - all                    # Check all environments

# =============================================================================
# PERMISSIONS: Enhanced permissions for PR comments and Azure access
# =============================================================================
permissions:
  id-token: write       # Required for OIDC authentication
  contents: read        # Read repository contents and configurations
  pull-requests: write  # Comment on PRs with compliance results

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Standard configuration loading for consistent environment settings
  # Note: Azure authentication credentials are now managed via GitHub Secrets
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration for notifications and Terraform versions
      # Note: Azure credentials are now securely managed via GitHub Secrets
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific configurations for backend access
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # COMPLIANCE CHECK JOB
  # ===========================================================================
  # Performs comprehensive security and compliance scanning using:
  # - TFLint for Terraform best practices and syntax issues
  # - Checkov for security vulnerabilities and misconfigurations
  # - Azure Policy for live environment policy compliance
  #
  # Uses matrix strategy to check multiple environments in parallel
  # ===========================================================================
  compliance-check:
    runs-on: self-hosted
    needs: load-config
    strategy:
      matrix:
        # Dynamic environment selection based on input or default to both
        environment: ${{ github.event.inputs.environment == 'all' && fromJson('["staging", "production"]') || github.event.inputs.environment && fromJson(format('["{0}"]', github.event.inputs.environment)) || fromJson('["staging", "production"]') }}
    steps:
      - uses: actions/checkout@v4
      
      # Setup Terraform for plan generation and validation
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ matrix.environment }}
          environment: ${{ matrix.environment }}
          
      # Authenticate with Azure for policy compliance checks (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Initialize Terraform to generate plan for compliance scanning
      - name: Configure backend
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Initialize with environment-specific backend
          ENV_CONFIG='${{ matrix.environment == 'staging' && needs.load-config.outputs.staging-config || needs.load-config.outputs.production-config }}'
          BACKEND_CONFIG=$(echo $ENV_CONFIG | jq -r '.terraform.backend')
          
          terraform init \
            -backend-config="storage_account_name=$(echo $BACKEND_CONFIG | jq -r '.storage_account_name')" \
            -backend-config="container_name=$(echo $BACKEND_CONFIG | jq -r '.container_name')" \
            -backend-config="resource_group_name=$(echo $BACKEND_CONFIG | jq -r '.resource_group_name')" \
            -backend-config="key=$(echo $BACKEND_CONFIG | jq -r '.key')"
            
      # Generate plan for compliance analysis (in JSON format for tools)
      - name: Generate plan for compliance check
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          terraform plan -out=compliance-plan-${{ matrix.environment }}
          # Convert plan to JSON for security scanning tools
          terraform show -json compliance-plan-${{ matrix.environment }} > compliance-plan-${{ matrix.environment }}.json
          
      # Install and run TFLint for Terraform best practices
      - name: Install tflint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
          
      - name: Run tflint security checks
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Initialize TFLint with latest rule definitions
          tflint --init
          # Run TFLint and generate JSON report (continue on failures for reporting)
          tflint --format json > tflint-report-${{ matrix.environment }}.json || true
          
      # Install and run Checkov for security vulnerability scanning
      - name: Install checkov
        run: |
          pip3 install checkov
          
      - name: Run checkov security scan
        working-directory: terraform/environments/${{ matrix.environment }}
        run: |
          # Scan the Terraform plan for security issues and misconfigurations
          checkov -f compliance-plan-${{ matrix.environment }}.json --framework terraform_plan --output json > checkov-report-${{ matrix.environment }}.json || true
          
      # Check live Azure Policy compliance for existing resources
      - name: Check Azure Policy compliance
        run: |
          echo "Checking Azure Policy compliance for ${{ matrix.environment }} environment..."
          
          # Query resource groups that match the environment naming pattern
          RESOURCE_GROUPS=$(az group list --query "[?contains(name, '${{ matrix.environment }}')].name" -o tsv)
          
          # Initialize compliance report structure
          echo '{"environment": "${{ matrix.environment }}", "policy_states": []}' > policy-compliance-${{ matrix.environment }}.json
          
          # Check policy compliance for each resource group
          for rg in $RESOURCE_GROUPS; do
            echo "Checking resource group: $rg"
            # Query policy compliance state and append to temporary file
            az policy state list --resource-group "$rg" --query '[].{policyDefinitionName: policyDefinitionName, complianceState: complianceState, resourceId: resourceId}' >> policy-states-temp.json || true
          done
          
          # Merge policy states into main compliance report
          if [ -f policy-states-temp.json ]; then
            jq '.policy_states += input' policy-compliance-${{ matrix.environment }}.json policy-states-temp.json > temp.json
            mv temp.json policy-compliance-${{ matrix.environment }}.json
            rm policy-states-temp.json
          fi
          
      # Generate comprehensive compliance summary report
      - name: Generate compliance summary
        run: |
          # Create markdown report combining all compliance check results
          cat > compliance-summary-${{ matrix.environment }}.md << EOF
          # Compliance Report - ${{ matrix.environment }} Environment
          
          ## TFLint Security Checks
          \`\`\`json
          $(cat terraform/environments/${{ matrix.environment }}/tflint-report-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Checkov Security Scan
          \`\`\`json
          $(cat terraform/environments/${{ matrix.environment }}/checkov-report-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Azure Policy Compliance
          \`\`\`json
          $(cat policy-compliance-${{ matrix.environment }}.json 2>/dev/null || echo "{}")
          \`\`\`
          
          ## Summary
          - **Environment**: ${{ matrix.environment }}
          - **Scan Date**: $(date)
          - **Repository**: ${{ github.repository }}
          - **Commit**: ${{ github.sha }}
          EOF
          
      # Store all compliance reports as artifacts for audit trail
      - name: Upload compliance reports
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report-${{ matrix.environment }}
          path: |
            terraform/environments/${{ matrix.environment }}/tflint-report-${{ matrix.environment }}.json
            terraform/environments/${{ matrix.environment }}/checkov-report-${{ matrix.environment }}.json
            policy-compliance-${{ matrix.environment }}.json
            compliance-summary-${{ matrix.environment }}.md
          retention-days: 30
          
      # Post compliance summary as PR comment for visibility
      - name: Comment PR with compliance results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            try {
              // Read the generated compliance summary
              const summary = fs.readFileSync('compliance-summary-${{ matrix.environment }}.md', 'utf8');
              // Post as comment on the PR
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: summary
              });
            } catch (error) {
              console.log('Could not read compliance summary file');
            }
            
      # Notify team of compliance check completion
      - name: Notify compliance results
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ matrix.environment }}
          message: "Compliance check completed for ${{ matrix.environment }}. Check artifacts for detailed results."
```

```github-actions-workflow
# =============================================================================
# TERRAFORM TESTING PIPELINE
# =============================================================================
# This workflow provides automated infrastructure testing using Terratest,
# a Go testing framework specifically designed for testing infrastructure code.
#
# Testing Capabilities:
# 1. BASIC TESTS: Simple resource validation and output verification
# 2. INTEGRATION TESTS: End-to-end infrastructure testing
# 3. ALL TESTS: Comprehensive test suite execution
#
# Key Features:
# - Go-based infrastructure testing with Terratest
# - Automated test execution on code changes
# - Test result artifacts and reporting
# - Environment-specific test execution
# - Automatic cleanup of test resources
# =============================================================================

name: 'Terraform Testing'

# =============================================================================
# TRIGGERS: Manual execution + automatic testing on changes
# =============================================================================
on:
  workflow_dispatch:  # Manual execution with test suite selection
    inputs:
      test_suite:
        description: 'Test suite to run'
        required: true
        type: choice
        options:
          - basic       # Basic resource validation tests
          - integration # End-to-end integration tests
          - all         # Run complete test suite
      environment:
        description: 'Environment to test against'
        required: true
        type: choice
        options:
          - staging     # Run tests against staging environment
          - production  # Run tests against production (use carefully!)
  push:
    paths: ['tests/**', 'terraform/**']  # Auto-trigger when test or Terraform files change
    branches: [develop, main]

# =============================================================================
# PERMISSIONS: Required for Azure access and test execution
# =============================================================================
permissions:
  id-token: write  # Required for OIDC authentication
  contents: read   # Read repository contents and test files

jobs:
  # ===========================================================================
  # CONFIGURATION LOADING JOB
  # ===========================================================================
  # Standard configuration loading for consistent environment settings
  # Note: Azure authentication credentials are now managed via GitHub Secrets
  # ===========================================================================
  load-config:
    runs-on: self-hosted
    outputs:
      base-config: ${{ steps.load-base.outputs.config }}
      staging-config: ${{ steps.load-staging.outputs.config }}
      production-config: ${{ steps.load-production.outputs.config }}
    steps:
      - uses: actions/checkout@v4
      
      # Load shared configuration for notifications and Terraform versions
      # Note: Azure credentials are now securely managed via GitHub Secrets
      - name: Load base config
        id: load-base
        run: |
          CONFIG=$(cat config/base.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      # Load environment-specific configurations for testing
      - name: Load staging config
        id: load-staging
        run: |
          CONFIG=$(cat config/staging.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT
          
      - name: Load production config
        id: load-production
        run: |
          CONFIG=$(cat config/production.json | jq -c .)
          echo "config=$CONFIG" >> $GITHUB_OUTPUT

  # ===========================================================================
  # TERRAFORM TESTS JOB
  # ===========================================================================
  # Executes Go-based infrastructure tests using Terratest framework
  # Tests deploy real infrastructure, validate it, then clean up
  #
  # IMPORTANT: These tests create real Azure resources and incur costs
  # Always ensure proper cleanup and use staging environment for testing
  # ===========================================================================
  terraform-tests:
    runs-on: self-hosted
    needs: load-config
    environment: ${{ github.event.inputs.environment || 'staging' }}  # Requires approval for production testing
    steps:
      - uses: actions/checkout@v4
      
      # Setup Go runtime for Terratest execution
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'  # Use Go version compatible with Terratest
          
      # Setup Terraform for infrastructure deployment during tests
      - name: Setup Terraform
        uses: ./.github/actions/setup-terraform
        with:
          terraform_version: ${{ fromJson(needs.load-config.outputs.base-config).terraform.version }}
          working_directory: terraform/environments/${{ github.event.inputs.environment || 'staging' }}
          environment: ${{ github.event.inputs.environment || 'staging' }}
          
      # Authenticate with Azure for test resource creation (using GitHub Secrets)
      - name: Azure Login
        uses: ./.github/actions/azure-login
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          
      # Install Go dependencies for Terratest
      - name: Install Go dependencies
        working-directory: tests/terratest
        run: |
          # Download all required Go modules
          go mod download
          # Ensure go.mod and go.sum are properly synchronized
          go mod tidy
          
      # Execute basic infrastructure validation tests
      - name: Run basic tests
        if: github.event.inputs.test_suite == 'basic' || github.event.inputs.test_suite == 'all' || github.event_name == 'push'
        working-directory: tests/terratest
        run: |
          # Run basic test suite with extended timeout for Azure resource operations
          go test -v -timeout 30m -run TestTerraformBasicExample
          
      # Execute comprehensive integration tests (if requested)
      - name: Run integration tests
        if: github.event.inputs.test_suite == 'integration' || github.event.inputs.test_suite == 'all'
        working-directory: tests/terratest
        run: |
          echo "Integration tests would run here"
          echo "Add your integration test functions to basic_test.go or create separate test files"
          # Example: go test -v -timeout 60m -run TestTerraformIntegration
          
      # Generate JSON test results for analysis and reporting
      - name: Generate test report
        if: always()  # Generate report even if tests fail for debugging
        working-directory: tests/terratest
        run: |
          # Generate machine-readable test results in JSON format
          go test -v -json > test-results.json || true
          
      # Store test results as artifacts for review and debugging
      - name: Upload test results
        if: always()  # Always upload results for analysis
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ github.event.inputs.environment || 'staging' }}
          path: tests/terratest/test-results.json
          retention-days: 30
          
      # Notify team of successful test completion
      - name: Notify test success
        if: success()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: success
          environment: ${{ github.event.inputs.environment || 'staging' }}
          message: "Terraform tests completed successfully for ${{ github.event.inputs.test_suite || 'basic' }} test suite"
          
      # Alert team immediately if tests fail (may indicate infrastructure issues)
      - name: Notify test failure
        if: failure()
        uses: ./.github/actions/teams-notification
        with:
          webhook_url: ${{ fromJson(needs.load-config.outputs.base-config).notifications.teams_webhook }}
          status: failure
          environment: ${{ github.event.inputs.environment || 'staging' }}
          message: "Terraform tests failed for ${{ github.event.inputs.test_suite || 'basic' }} test suite"
```

Now let me update the base configuration file to remove the Azure authentication credentials since they'll be managed via GitHub Secrets:

```json
{
  "terraform": {
    "version": "1.7.0",
    "providers": {
      "azurerm": {
        "version": "~> 3.0"
      }
    },
    "cache": {
      "enabled": true,
      "directory": ".terraform.d/plugin-cache"
    }
  },
  "notifications": {
    "teams_webhook": "REPLACE_WITH_YOUR_TEAMS_WEBHOOK_URL"
  },
  "storage": {
    "backup_account": "REPLACE_WITH_BACKUP_STORAGE_ACCOUNT",
    "backup_container": "terraform-state-backups"
  },
  "runners": {
    "type": "self-hosted",
    "labels": ["self-hosted", "linux", "x64"]
  },
  "_comments": {
    "azure_credentials": "Azure authentication credentials (tenant_id, subscription_id, client_id) are now managed via GitHub Secrets for enhanced security. Add these to your repository secrets: AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID, AZURE_CLIENT_ID",
    "teams_webhook": "Replace with your actual Teams channel webhook URL for notifications",
    "backup_storage": "Replace with your actual Azure storage account name for state backups",
    "terraform_cache": "Provider caching improves performance by avoiding repeated downloads"
  }
}
```

Now let me update the README to document the new GitHub Secrets approach:

Now let me update the README with the new GitHub Secrets approach:

```markdown
# GitHub Actions Terraform Templates

A comprehensive set of GitHub Actions workflows and Terraform templates for Azure infrastructure deployment with OIDC authentication, automated testing, compliance scanning, and operational workflows.

## Features

- 🔐 **Azure OIDC Authentication** - Secure authentication without storing credentials
- 🚀 **Multi-Environment Support** - Staging and production environments with approval gates
- 📦 **Modular Architecture** - Reusable GitHub Actions and Terraform modules
- 🔍 **Compliance Scanning** - Azure Policy validation, TFLint, and Checkov security checks
- 🧪 **Infrastructure Testing** - Terratest integration for automated testing
- 📊 **Drift Detection** - Weekly automated drift detection
- 🔧 **Utility Operations** - State management, dependency graphs, targeted operations
- 📢 **Teams Notifications** - Integrated Microsoft Teams webhook notifications
- ⚡ **Performance Optimized** - Terraform provider caching and parallel execution

## Project Structure

```
/
├── .github/
│   ├── workflows/                 # GitHub Actions workflows
│   │   ├── terraform-main.yml           # Main deployment pipeline
│   │   ├── terraform-drift-detection.yml
│   │   ├── terraform-state-management.yml
│   │   ├── terraform-utilities.yml
│   │   ├── terraform-compliance.yml
│   │   └── terraform-testing.yml
│   └── actions/                   # Reusable GitHub Actions
│       ├── setup-terraform/
│       ├── azure-login/
│       ├── teams-notification/
│       └── terraform-cache/
├── config/                        # Environment configurations
│   ├── base.json                  # Shared configuration
│   ├── staging.json               # Staging-specific config
│   └── production.json            # Production-specific config
├── terraform/
│   ├── environments/              # Environment-specific deployments
│   │   ├── staging/
│   │   └── production/
│   └── modules/                   # Reusable Terraform modules
│       └── example/
├── tests/
│   ├── terratest/                 # Infrastructure tests
│   └── azure-policy/              # Compliance rules
└── README.md
```

## Quick Start

### 1. Prerequisites

- Azure subscription with appropriate permissions
- GitHub repository with Actions enabled
- Azure OIDC application registration configured
- Self-hosted GitHub Actions runners (optional but recommended)

### 2. Configuration

#### Azure OIDC Setup

1. Create an Azure AD application registration
2. Configure federated credentials for GitHub OIDC
3. Assign appropriate Azure RBAC roles to the service principal

#### GitHub Secrets Configuration

**Security Note:** Azure authentication credentials are managed via GitHub Secrets for enhanced security. Never store sensitive credentials in configuration files.

1. Go to your repository → Settings → Secrets and variables → Actions
2. Add the following **Repository Secrets**:

```
AZURE_TENANT_ID          # Your Azure Tenant ID
AZURE_SUBSCRIPTION_ID    # Your Azure Subscription ID  
AZURE_CLIENT_ID          # Your Azure Client ID for OIDC
```

3. (Optional) Add the following secret for Teams notifications:
```
TEAMS_WEBHOOK_URL        # Your Microsoft Teams webhook URL
```

#### Repository Configuration

Fill in the configuration files in the `config/` directory:

**config/base.json:**
```json
{
  "terraform": {
    "version": "1.7.0",
    "providers": {
      "azurerm": {
        "version": "~> 3.0"
      }
    }
  },
  "notifications": {
    "teams_webhook": "REPLACE_WITH_YOUR_TEAMS_WEBHOOK_URL"
  },
  "storage": {
    "backup_account": "REPLACE_WITH_BACKUP_STORAGE_ACCOUNT",
    "backup_container": "terraform-state-backups"
  }
}
```

**config/staging.json & config/production.json:**
```json
{
  "terraform": {
    "backend": {
      "storage_account_name": "your-state-storage-account",
      "container_name": "your-state-container",
      "resource_group_name": "your-state-resource-group",
      "key": "terraform.tfstate"
    }
  },
  "deployment": {
    "auto_approve": false,
    "notification_enabled": true
  }
}
```

#### GitHub Environment Protection

Set up GitHub Environment Protection Rules for approval gates:
   - Go to Settings → Environments
   - Create `staging` and `production` environments
   - Configure required reviewers and deployment branches
   - Set environment-specific secrets if needed

### 3. Workflows Overview

#### Main Deployment (`terraform-main.yml`)
- **Triggers:** Push to develop/main, PRs to main, manual dispatch
- **Features:** Parallel validation, plan caching, approval gates
- **Flow:** Validation → Compliance → Staging → Production

#### Drift Detection (`terraform-drift-detection.yml`)
- **Schedule:** Weekly on Sundays at 9 AM UTC
- **Purpose:** Detect infrastructure drift and generate reports

#### State Management (`terraform-state-management.yml`)
- **Operations:** Unlock, backup, restore state files
- **Schedule:** Weekly automated backups on Sundays at 2 AM UTC

#### Utilities (`terraform-utilities.yml`)
- **Operations:** tfupdate, dependency graphs, targeted apply/destroy
- **Triggers:** Manual dispatch, terraform file changes

#### Compliance (`terraform-compliance.yml`)
- **Tools:** TFLint, Checkov, Azure Policy validation
- **Triggers:** Pull requests, manual dispatch

#### Testing (`terraform-testing.yml`)
- **Framework:** Terratest with Go
- **Suites:** Basic, integration, or all tests

## Usage Examples

### Deploy to Staging
```bash
# Automatically triggers on push to develop branch
git push origin develop

# Or manually trigger
# Go to Actions → Terraform Main Deployment → Run workflow
```

### Run Drift Detection
```bash
# Manually trigger drift detection
# Go to Actions → Terraform Drift Detection → Run workflow
```

### Perform Targeted Operations
```bash
# Target specific resources for apply/destroy
# Go to Actions → Terraform Utilities → Run workflow
# Operation: target-apply
# Target resources: module.example.azurerm_resource_group.main
```

### Run Infrastructure Tests
```bash
# Run Terratest suite
# Go to Actions → Terraform Testing → Run workflow
# Test suite: basic
# Environment: staging
```

## Reusable Actions

### Setup Terraform
```yaml
- uses: ./.github/actions/setup-terraform
  with:
    terraform_version: 1.7.0
    working_directory: terraform/environments/staging
    environment: staging
```

### Azure Login
```yaml
- uses: ./.github/actions/azure-login
  with:
    tenant_id: ${{ secrets.AZURE_TENANT_ID }}
    subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    client_id: ${{ secrets.AZURE_CLIENT_ID }}
```

### Teams Notification
```yaml
- uses: ./.github/actions/teams-notification
  with:
    webhook_url: ${{ secrets.TEAMS_WEBHOOK_URL }}
    status: success
    environment: production
    message: "Deployment completed successfully"
```

## Security Considerations

### Authentication & Authorization
- **Azure OIDC:** All authentication uses OIDC tokens - no stored credentials
- **GitHub Secrets:** Sensitive data encrypted and managed by GitHub
- **Principle of Least Privilege:** Service principals have minimal required permissions

### RBAC Configuration
- **Staging Environment:** Contributor role on staging resource groups
- **Production Environment:** Custom role with minimal required permissions
- **State Storage:** Storage Blob Data Contributor role
- **Backup Storage:** Storage Blob Data Reader/Writer role

### GitHub Repository Settings
- Branch protection rules for main/develop branches
- Required status checks for all workflows
- Restrict push access to authorized users
- Enable signed commits (recommended)
- Environment protection rules with manual approvals

### Secret Management Best Practices
- Use GitHub repository secrets for sensitive data
- Rotate credentials regularly
- Audit secret access and usage
- Use environment-specific secrets when needed

## Troubleshooting

### Common Issues

1. **Authentication Failures**
   - Verify GitHub Secrets are correctly set: `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_CLIENT_ID`
   - Check OIDC configuration in Azure AD
   - Verify federated credential subject claims
   - Ensure service principal has appropriate roles

2. **State Lock Issues**
   - Use the state management workflow to unlock
   - Verify Azure Storage permissions
   - Check for concurrent workflow runs

3. **Plan Cache Misses**
   - Terraform provider versions changed
   - Lock file modifications
   - Clear cache and re-run workflow

4. **Secret Access Issues**
   - Verify secrets are defined at repository level
   - Check if environment-specific secrets are needed
   - Ensure proper secret naming (case-sensitive)

### Debugging

Enable debug logging by setting repository secrets:
- `ACTIONS_STEP_DEBUG=true`
- `ACTIONS_RUNNER_DEBUG=true`

### Validation Steps

Before running workflows, verify:
1. All required GitHub Secrets are configured
2. Azure OIDC federated credentials are properly set up
3. Service principal has necessary Azure RBAC roles
4. GitHub environments are configured with protection rules
5. Configuration files contain valid values (no placeholder text)

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with staging environment
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:
- Create GitHub Issues for bugs and feature requests
- Check the troubleshooting section
- Review workflow run logs for detailed error messages
```

Perfect! I've successfully updated all the workflow files and configuration to use GitHub Secrets for Azure authentication parameters instead of storing them in the base.json configuration file. Here's a summary of the changes:

## Changes Made

### 1. **Updated All Workflow Files**
All six workflow files now use GitHub Secrets for Azure authentication:
- `terraform-main.yml`
- `terraform-drift-detection.yml` 
- `terraform-state-management.yml`
- `terraform-utilities.yml`
- `terraform-compliance.yml`
- `terraform-testing.yml`

**Changed from:**
```yaml
tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
```

**To:**
```yaml
tenant_id: ${{ secrets.AZURE_TENANT_ID }}
subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
client_id: ${{ secrets.AZURE_CLIENT_ID }}
```

### 2. **Updated Configuration File**
- Removed Azure credentials from `config/base.json`
- Added helpful comments explaining the change
- Streamlined the configuration structure
- Added provider caching configuration

### 3. **Updated README Documentation**
- Added a dedicated "GitHub Secrets Configuration" section
- Updated all examples to use GitHub Secrets
- Enhanced security considerations section
- Added troubleshooting for secret-related issues
- Updated validation steps

## Required Setup Steps

To complete this transition, you'll need to:

### 1. **Configure GitHub Secrets**
Go to your repository → Settings → Secrets and variables → Actions, and add:
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID` 
- `AZURE_CLIENT_ID`

### 2. **Optional Teams Webhook Secret**
- `TEAMS_WEBHOOK_URL` (if using Teams notifications)

### 3. **Update Configuration Files**
Replace placeholder values in:
- `config/base.json` - Teams webhook URL and backup storage account
- `config/staging.json` and `config/production.json` - Backend storage settings

## Security Benefits

This change provides several security improvements:

1. **No Sensitive Data in Repository** - Credentials are never stored in code
2. **Encrypted Storage** - GitHub encrypts secrets at rest and in transit
3. **Access Control** - Only authorized workflows can access secrets
4. **Audit Trail** - GitHub logs secret access for security auditing
5. **Rotation Support** - Easy to update credentials without code changes

The workflows will now be more secure while maintaining the same functionality!

```yaml
tenant_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.tenant_id }}
subscription_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.subscription_id }}
client_id: ${{ fromJson(needs.load-config.outputs.base-config).azure.client_id }}
```

```yaml
tenant_id: ${{ secrets.AZURE_TENANT_ID }}
subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
client_id: ${{ secrets.AZURE_CLIENT_ID }}
```

