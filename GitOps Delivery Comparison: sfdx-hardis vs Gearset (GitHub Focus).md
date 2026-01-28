# GitOps Delivery Comparison: sfdx-hardis vs Gearset (GitHub Focus)

## Executive Summary

This document compares two Salesforce DevOps approaches for managing deliveries with long-lived branches and permanent orgs (INT, UAT, PROD) using **GitHub** as the version control platform. Both tools support GitHub Actions and Pull Requests (PRs) but differ significantly in implementation, philosophy, and feature sets.

---

## 1. Branch Strategy Overview

### 1.1 sfdx-hardis: Long Branches with Permanent Orgs

sfdx-hardis uses a **BUILD/RUN** separation model with long-lived branches mapped to permanent Salesforce orgs. GitHub Actions workflows are auto-generated during setup.

```mermaid
flowchart TB
    subgraph "BUILD Layer - New Features"
        DEV_SB[("Developer Sandboxes<br/>(Source-tracked)")]
        INT_ORG[("Integration Org")]
        UAT_ORG[("UAT Org")]
    end
    
    subgraph "RUN Layer - Production Pipeline"
        PREPROD_ORG[("PreProd Org")]
        PROD_ORG[("Production Org")]
    end
    
    subgraph "GitHub Branches"
        FEATURE[feature/*]
        INTEGRATION[integration]
        UAT[uat]
        PREPROD[preprod]
        MAIN[main]
    end
    
    DEV_SB -->|"Push"| FEATURE
    FEATURE -->|"PR to integration"| INTEGRATION
    INTEGRATION -->|"GitHub Actions Deploy"| INT_ORG
    INTEGRATION -->|"PR to uat"| UAT
    UAT -->|"GitHub Actions Deploy"| UAT_ORG
    UAT -->|"PR to preprod"| PREPROD
    PREPROD -->|"GitHub Actions Deploy"| PREPROD_ORG
    PREPROD -->|"PR to main"| MAIN
    MAIN -->|"GitHub Actions Deploy"| PROD_ORG
    
    style PROD_ORG fill:#2ecc71
    style PREPROD_ORG fill:#f39c12
    style UAT_ORG fill:#3498db
    style INT_ORG fill:#9b59b6
```

### 1.2 Gearset: Expanded Branching Model

Gearset uses an **expanded branching model** with promotion branches and automatic back-propagation. Integrates natively with GitHub for PR management.

```mermaid
flowchart TB
    subgraph "Developer Environments"
        DEV_SB1[("Dev Sandbox 1")]
        DEV_SB2[("Dev Sandbox 2")]
    end
    
    subgraph "Pipeline Environments"
        INT_ORG[("Integration Org")]
        UAT_ORG[("UAT Org")]
        PROD_ORG[("Production Org")]
    end
    
    subgraph "GitHub Branches"
        FEATURE1[feature/user-story-1]
        FEATURE2[feature/user-story-2]
        PROMO1["gs-pipeline/feature1_-_int"]
        PROMO2["gs-pipeline/feature1_-_uat"]
        INT_BR[integration]
        UAT_BR[uat]
        MAIN[main]
    end
    
    DEV_SB1 --> FEATURE1
    DEV_SB2 --> FEATURE2
    
    FEATURE1 -->|"Auto PR"| PROMO1
    PROMO1 -->|"Merge PR"| INT_BR
    INT_BR -->|"Gearset CI Deploy"| INT_ORG
    
    FEATURE1 -->|"Auto PR"| PROMO2
    PROMO2 -->|"Merge PR"| UAT_BR
    UAT_BR -->|"Gearset CI Deploy"| UAT_ORG
    
    UAT_BR -->|"Release PR"| MAIN
    MAIN -->|"Gearset CI Deploy"| PROD_ORG
    
    style PROD_ORG fill:#2ecc71
    style UAT_ORG fill:#3498db
    style INT_ORG fill:#9b59b6
```

---

## 2. Retrofit vs Back-Propagation

### 2.1 sfdx-hardis Retrofit Process

The retrofit mechanism in sfdx-hardis retrieves production changes and propagates them back to lower environments (typically preprod/uat) via automated GitHub PRs.

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant GH as GitHub
    participant PREPROD as preprod branch
    participant MAIN as main branch
    participant PROD as Production Org
    participant GHA as GitHub Actions
    participant RETROFIT as Retrofit Job
    
    Note over DEV,RETROFIT: Normal Deployment Flow
    DEV->>GH: Create PR to preprod
    GH->>PREPROD: Merge PR
    DEV->>GH: Create PR to main
    GH->>MAIN: Merge PR
    GHA->>PROD: Deploy to Production
    
    Note over DEV,RETROFIT: Retrofit Process - Post Production
    GHA->>RETROFIT: Trigger Retrofit Workflow
    RETROFIT->>PROD: Retrieve metadata from Production
    RETROFIT->>RETROFIT: Compare with main branch
    RETROFIT->>RETROFIT: Identify changed metadata types
    RETROFIT->>GH: Create PR to retrofitBranch
    
    Note over GH: Retrofit PR contains:<br/>- CompactLayout<br/>- CustomField<br/>- CustomObject<br/>- FlexiPage<br/>- PermissionSet<br/>- ValidationRule<br/>- etc.
    
    DEV->>GH: Review and Merge Retrofit PR
    GHA->>PREPROD: Deploy retrofit to PreProd Org
```

**sfdx-hardis Retrofit Configuration (.sfdx-hardis.yml):**
```yaml
productionBranch: main
retrofitBranch: preprod
sourcesToRetrofit:
  - CompactLayout
  - CustomApplication
  - CustomField
  - CustomLabel
  - CustomMetadata
  - CustomObject
  - FlexiPage
  - Layout
  - PermissionSet
  - ValidationRule
retrofitIgnoredFiles:
  - force-app/main/default/flexipages/Dashboard.flexipage-meta.xml
```

**GitHub Actions Workflow (auto-generated):**
```yaml
name: Retrofit from Production
on:
  workflow_dispatch:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM
jobs:
  retrofit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Retrofit
        run: sf hardis:org:retrieve:sources:retrofit
```

### 2.2 Gearset Back-Propagation Process

Gearset automatically creates back-propagation PRs in GitHub after changes are merged to main, keeping all upstream environments synchronized.

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant GH as GitHub
    participant FEATURE as Feature Branch
    participant INT as integration
    participant UAT as uat
    participant MAIN as main
    participant PROD as Production Org
    participant GS as Gearset
    
    Note over DEV,GS: Forward Promotion via GitHub PRs
    DEV->>FEATURE: Commit changes
    DEV->>GH: Push to GitHub
    GS->>GH: Auto-create PR to integration
    GS->>GH: Merge PR to integration
    GS->>INT: Deploy to INT Org
    
    GS->>GH: Auto-create PR to uat
    GS->>GH: Merge PR to uat
    GS->>UAT: Deploy to UAT Org
    
    GS->>GH: Auto-create PR to main
    GS->>GH: Merge PR to main
    GS->>PROD: Deploy to Production
    
    Note over DEV,GS: Automatic Back-Propagation PRs
    GS->>GS: Detect merge to main
    GS->>GH: Auto-create Back-prop PR to uat
    GS->>GH: Auto-create Back-prop PR to integration
    
    Note over GH: Back-prop PRs contain:<br/>- All feature commits<br/>- Conflict resolutions<br/>- Merge commits
```

---

## 3. Release Branch Model Comparison

### 3.1 sfdx-hardis: UAT-based Release with GitHub PRs

In sfdx-hardis, the release is essentially the state of the UAT branch at a given point, which then flows to preprod and production via sequential GitHub PRs.

```mermaid
flowchart LR
    subgraph "Feature Development"
        F1[Feature 1]
        F2[Feature 2]
        F3[Feature 3]
    end
    
    subgraph "BUILD Phase"
        INT[integration]
        UAT_RELEASE["uat<br/>(Release Candidate)"]
    end
    
    subgraph "RUN Phase"
        PREPROD[preprod]
        MAIN[main]
    end
    
    F1 -->|"PR"| INT
    F2 -->|"PR"| INT
    F3 -->|"PR"| INT
    INT -->|"PR to uat"| UAT_RELEASE
    
    UAT_RELEASE -->|"UAT Testing<br/>Complete - PR"| PREPROD
    PREPROD -->|"Final<br/>Validation - PR"| MAIN
    
    style UAT_RELEASE fill:#e74c3c,color:#fff
    style PREPROD fill:#f39c12
    style MAIN fill:#2ecc71
```

**GitHub Branch Protection Rules (recommended):**
```
main:
  - Require PR reviews: 2 approvers
  - Require status checks: deployment validation
  - No direct pushes

preprod:
  - Require PR reviews: 1 approver
  - Require status checks: deployment validation

uat:
  - Require status checks: deployment validation

integration:
  - Require status checks: deployment validation
```

### 3.2 Gearset: Explicit Release Branch with GitHub Integration

Gearset provides an explicit release branch mechanism for bundling multiple features into a single coordinated release, all managed via GitHub PRs.

```mermaid
flowchart TB
    subgraph "Feature Branches"
        F1[Feature 1]
        F2[Feature 2]
        F3[Feature 3]
    end
    
    subgraph "Release Management"
        REL["release/v2.0<br/>(Carved from main)"]
    end
    
    subgraph "Environment Branches"
        INT[integration]
        UAT[uat]
        MAIN[main]
    end
    
    F1 -->|"PR to INT"| INT
    F2 -->|"PR to INT"| INT
    F3 -->|"PR to INT"| INT
    
    INT -->|"PR to UAT"| UAT
    
    F1 -->|"Add to Release"| REL
    F2 -->|"Add to Release"| REL
    F3 -->|"Add to Release"| REL
    
    REL -->|"Scheduled<br/>Release PR"| MAIN
    
    MAIN -->|"Back-prop PR"| UAT
    MAIN -->|"Back-prop PR"| INT
    
    style REL fill:#e74c3c,color:#fff
    style MAIN fill:#2ecc71
```

---

## 4. Complete GitHub Workflow Diagrams

### 4.1 sfdx-hardis Complete GitHub Actions Flow

```mermaid
flowchart TB
    subgraph "Developer Workflow"
        START((Start))
        CREATE_SB[Create Source-tracked<br/>Sandbox from INT]
        DEVELOP[Develop in Sandbox]
        PULL[Pull changes to local]
        COMMIT[Commit to feature branch]
        PUSH[Push to GitHub]
        CREATE_PR[Create Pull Request<br/>to integration]
    end
    
    subgraph "GitHub Actions - BUILD"
        VALIDATE_INT[Validate against INT<br/>on PR open]
        MERGE_INT[Merge PR to integration]
        DEPLOY_INT[Deploy to INT Org]
        
        CREATE_PR_UAT[Create PR to uat]
        VALIDATE_UAT[Validate against UAT]
        MERGE_UAT[Merge PR to uat]
        DEPLOY_UAT[Deploy to UAT Org]
    end
    
    subgraph "GitHub Actions - RUN"
        CREATE_PR_PREPROD[Create PR to preprod]
        VALIDATE_PREPROD[Validate against PreProd]
        MERGE_PREPROD[Merge PR to preprod]
        DEPLOY_PREPROD[Deploy to PreProd Org]
        
        CREATE_PR_MAIN[Create PR to main]
        VALIDATE_PROD[Validate against PROD]
        MERGE_MAIN[Merge PR to main]
        DEPLOY_PROD[Deploy to Production]
    end
    
    subgraph "Retrofit via GitHub Actions"
        TRIGGER_RETROFIT[Scheduled/Manual Trigger]
        RETRIEVE_PROD[Retrieve from Production]
        CREATE_RETROFIT_PR[Create Retrofit PR<br/>to preprod]
        MERGE_RETROFIT[Review and Merge PR]
    end
    
    START --> CREATE_SB --> DEVELOP --> PULL --> COMMIT --> PUSH --> CREATE_PR
    CREATE_PR --> VALIDATE_INT
    VALIDATE_INT -->|Pass| MERGE_INT --> DEPLOY_INT
    DEPLOY_INT --> CREATE_PR_UAT --> VALIDATE_UAT
    VALIDATE_UAT -->|Pass| MERGE_UAT --> DEPLOY_UAT
    DEPLOY_UAT --> CREATE_PR_PREPROD --> VALIDATE_PREPROD
    VALIDATE_PREPROD -->|Pass| MERGE_PREPROD --> DEPLOY_PREPROD
    DEPLOY_PREPROD --> CREATE_PR_MAIN --> VALIDATE_PROD
    VALIDATE_PROD -->|Pass| MERGE_MAIN --> DEPLOY_PROD
    
    DEPLOY_PROD --> TRIGGER_RETROFIT
    TRIGGER_RETROFIT --> RETRIEVE_PROD --> CREATE_RETROFIT_PR --> MERGE_RETROFIT
```

**Example GitHub Actions Workflow (.github/workflows/deploy.yml):**
```yaml
name: Deploy to Salesforce
on:
  pull_request:
    branches: [integration, uat, preprod, main]
  push:
    branches: [integration, uat, preprod, main]

jobs:
  validate:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    container: hardisgroupcom/sfdx-hardis:latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate Deployment
        run: sf hardis:project:deploy:sources:dx --check

  deploy:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    container: hardisgroupcom/sfdx-hardis:latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Org
        run: sf hardis:project:deploy:sources:dx
```

### 4.2 Gearset Complete GitHub Pipeline Flow

```mermaid
flowchart TB
    subgraph "Developer Workflow"
        START((Start))
        CREATE_FEATURE[Create Feature Branch<br/>from main]
        DEVELOP[Develop in Dev Sandbox]
        COMMIT[Commit changes]
        PUSH[Push to GitHub]
    end
    
    subgraph "Gearset Pipelines - GitHub PRs"
        AUTO_PR_INT[Auto-create PR<br/>to integration]
        SEMANTIC_MERGE_INT[Semantic Merge Check]
        VALIDATE_INT[Validate against INT Org]
        PROMOTE_INT[Merge PR to integration]
        DEPLOY_INT[Gearset CI Deploy to INT]
        
        AUTO_PR_UAT[Auto-create PR<br/>to uat]
        SEMANTIC_MERGE_UAT[Semantic Merge Check]
        VALIDATE_UAT[Validate against UAT Org]
        PROMOTE_UAT[Merge PR to uat]
        DEPLOY_UAT[Gearset CI Deploy to UAT]
        
        AUTO_PR_MAIN[Auto-create PR<br/>to main]
        VALIDATE_PROD[Validate against PROD]
        PROMOTE_MAIN[Merge PR to main]
        DEPLOY_PROD[Gearset CI Deploy to PROD]
    end
    
    subgraph "Release Option"
        CREATE_RELEASE[Create Release Branch]
        ADD_FEATURES[Add Features to Release]
        VALIDATE_RELEASE[Validate Release]
        SCHEDULE_DEPLOY[Schedule Release PR]
    end
    
    subgraph "Automatic Back-Propagation"
        DETECT_MERGE[Detect merge to main]
        BACKPROP_UAT[Create Back-prop PR<br/>to uat]
        BACKPROP_INT[Create Back-prop PR<br/>to integration]
        SYNC_ENVS[Environments Synchronized]
    end
    
    START --> CREATE_FEATURE --> DEVELOP --> COMMIT --> PUSH
    PUSH --> AUTO_PR_INT --> SEMANTIC_MERGE_INT --> VALIDATE_INT
    VALIDATE_INT -->|Pass| PROMOTE_INT --> DEPLOY_INT
    
    DEPLOY_INT --> AUTO_PR_UAT --> SEMANTIC_MERGE_UAT --> VALIDATE_UAT
    VALIDATE_UAT -->|Pass| PROMOTE_UAT --> DEPLOY_UAT
    
    DEPLOY_UAT --> AUTO_PR_MAIN
    DEPLOY_UAT -.->|Optional| CREATE_RELEASE
    CREATE_RELEASE --> ADD_FEATURES --> VALIDATE_RELEASE --> SCHEDULE_DEPLOY
    SCHEDULE_DEPLOY --> PROMOTE_MAIN
    
    AUTO_PR_MAIN --> VALIDATE_PROD
    VALIDATE_PROD -->|Pass| PROMOTE_MAIN --> DEPLOY_PROD
    
    DEPLOY_PROD --> DETECT_MERGE
    DETECT_MERGE --> BACKPROP_UAT
    DETECT_MERGE --> BACKPROP_INT
    BACKPROP_UAT --> SYNC_ENVS
    BACKPROP_INT --> SYNC_ENVS
```

---

## 5. Hotfix Handling Comparison (GitHub)

### 5.1 sfdx-hardis Hotfix Process

```mermaid
flowchart LR
    subgraph "Hotfix in RUN Layer"
        ISSUE[Production Issue]
        CREATE_HF[Create hotfix branch<br/>from preprod]
        FIX[Apply fix]
        PR_PREPROD[PR to preprod]
        DEPLOY_PP[GitHub Actions<br/>Deploy to PreProd]
        PR_MAIN[PR to main]
        DEPLOY_PROD[GitHub Actions<br/>Deploy to Production]
    end
    
    subgraph "Retrofit to BUILD"
        RETROFIT[Retrofit GitHub Action]
        PR_INT[PR to integration]
        SYNC[Sync BUILD layer]
    end
    
    ISSUE --> CREATE_HF --> FIX --> PR_PREPROD --> DEPLOY_PP
    DEPLOY_PP --> PR_MAIN --> DEPLOY_PROD
    DEPLOY_PROD --> RETROFIT --> PR_INT --> SYNC
    
    style ISSUE fill:#e74c3c,color:#fff
```

### 5.2 Gearset Hotfix Process

```mermaid
flowchart LR
    subgraph "Hotfix Process"
        ISSUE[Production Issue]
        CREATE_HF[Create hotfix branch<br/>from main]
        FIX[Apply fix]
        PR_MAIN[PR directly to main]
        DEPLOY_PROD[Gearset Deploy<br/>to Production]
    end
    
    subgraph "Automatic GitHub PRs"
        BACKPROP[Automatic Back-prop]
        PR_UAT[PR to uat]
        PR_INT[PR to integration]
        SYNC[All envs synced]
    end
    
    ISSUE --> CREATE_HF --> FIX --> PR_MAIN --> DEPLOY_PROD
    DEPLOY_PROD --> BACKPROP
    BACKPROP --> PR_UAT --> SYNC
    BACKPROP --> PR_INT --> SYNC
    
    style ISSUE fill:#e74c3c,color:#fff
```

---

## 6. Feature Comparison Table (GitHub Focus)

| Feature | sfdx-hardis | Gearset |
|---------|-------------|---------|
| **Licensing** | Open-source (Free) | Commercial (Paid) |
| **GitHub Integration** | Native GitHub Actions | Native API integration |
| **PR Creation** | Manual (developer creates PRs) | Automatic (promotion branches) |
| **PR Validation** | GitHub Actions workflow | Gearset CI jobs |
| **Branch Strategy** | BUILD/RUN separation | Expanded branching model |
| **Promotion Branches** | Not used | Automatic (gs-pipeline/*) |
| **Conflict Resolution** | Standard Git (manual) | Semantic merge (automated) |
| **Back-sync Mechanism** | Retrofit (scheduled GitHub Action) | Back-propagation (automatic PRs) |
| **Release Branches** | Implicit (UAT state) | Explicit release branch with PRs |
| **PR Comments** | Deployment results posted | Validation results posted |
| **Branch Protection** | Standard GitHub rules | Respects GitHub rules |
| **Status Checks** | Via GitHub Actions | Via Gearset webhooks |
| **Quick Deploy** | Supported | Supported |
| **PR Review Integration** | Native GitHub reviews | Native GitHub reviews |
| **GitHub App** | Not required | Gearset GitHub App |

---

## 7. Pros and Cons Analysis (GitHub Context)

### 7.1 sfdx-hardis with GitHub

```mermaid
mindmap
  root((sfdx-hardis + GitHub))
    Pros
      Free and Open Source
      Native GitHub Actions
      Full control over workflows
      Standard Git PR process
      No vendor lock-in
      Highly customizable YAML
      Strong community support
      Transparent operations
    Cons
      Manual PR creation
      Steeper learning curve
      Standard Git conflicts
      Retrofit is scheduled
      More GitHub Actions config
      Git expertise required
```

#### Detailed Pros:
1. **Cost-effective**: No licensing costs, uses free GitHub Actions minutes
2. **Native GitHub Actions**: Auto-generated workflows, easy to customize
3. **Standard PR workflow**: Developers use familiar GitHub PR process
4. **Transparency**: All operations visible in GitHub Actions logs
5. **Flexibility**: YAML configuration adapts to any org structure
6. **No vendor lock-in**: Standard Git and Salesforce CLI commands
7. **BUILD/RUN separation**: Clear distinction for production pipeline

#### Detailed Cons:
1. **Manual PR creation**: Developers must create PRs between branches
2. **Git expertise required**: Team needs solid Git fundamentals
3. **Standard conflicts**: No intelligent Salesforce metadata merge
4. **Retrofit timing**: Scheduled jobs, not real-time synchronization
5. **Setup investment**: Initial GitHub Actions configuration required

### 7.2 Gearset with GitHub

```mermaid
mindmap
  root((Gearset + GitHub))
    Pros
      Automatic PR creation
      Semantic merge algorithm
      Auto back-prop PRs
      Visual conflict resolution
      Release branch PRs
      Respects branch protection
      Lower learning curve
      Strong support
    Cons
      License costs
      Vendor dependency
      Promotion branch clutter
      Less customizable
      Gearset-specific knowledge
      Black-box operations
```

#### Detailed Pros:
1. **Automatic PR creation**: Promotion PRs created automatically in GitHub
2. **Semantic merge**: Understands Salesforce metadata, reduces PR conflicts
3. **Automatic back-propagation**: PRs created automatically after production merge
4. **Visual tools**: Conflict resolution UI integrated with GitHub PRs
5. **Release management**: Explicit release branches with scheduled PR merges
6. **Branch protection**: Respects all GitHub branch protection rules
7. **Lower barrier**: Admins can work without deep Git/GitHub knowledge

#### Detailed Cons:
1. **Cost**: Significant licensing investment
2. **Promotion branches**: gs-pipeline/* branches can clutter repository
3. **Vendor lock-in**: Workflow depends on Gearset infrastructure
4. **Learning curve**: Must understand Gearset's branching model
5. **Reduced transparency**: Some operations abstracted from GitHub

---

## 8. GitHub PR Workflow Comparison

### 8.1 PR Lifecycle Comparison

```mermaid
flowchart TB
    subgraph "sfdx-hardis PR Flow"
        SH_DEV[Developer creates PR]
        SH_GHA[GitHub Actions validates]
        SH_REVIEW[Manual review]
        SH_MERGE[Merge PR]
        SH_DEPLOY[GitHub Actions deploys]
        SH_NEXT[Developer creates next PR]
        
        SH_DEV --> SH_GHA --> SH_REVIEW --> SH_MERGE --> SH_DEPLOY --> SH_NEXT
    end
    
    subgraph "Gearset PR Flow"
        GS_PUSH[Developer pushes]
        GS_AUTO[Gearset auto-creates PR]
        GS_SEMANTIC[Semantic merge check]
        GS_VALIDATE[Gearset validates]
        GS_REVIEW[Manual review]
        GS_PROMOTE[Click Promote in Gearset]
        GS_DEPLOY[Gearset deploys]
        GS_NEXT[Gearset auto-creates next PR]
        
        GS_PUSH --> GS_AUTO --> GS_SEMANTIC --> GS_VALIDATE --> GS_REVIEW --> GS_PROMOTE --> GS_DEPLOY --> GS_NEXT
    end
```

### 8.2 Key GitHub PR Differences

| Aspect | sfdx-hardis | Gearset |
|--------|-------------|---------|
| **PR Creation** | Manual by developer | Automatic by Gearset |
| **PR Source Branch** | Feature branch directly | Promotion branch (gs-pipeline/*) |
| **PR Target** | Next environment branch | Next environment branch |
| **Validation Trigger** | PR open event | PR open + Gearset webhook |
| **Merge Action** | GitHub UI or CLI | Gearset UI (Promote button) |
| **Post-merge Deploy** | GitHub Actions on push | Gearset CI job |
| **Next Environment** | Manual PR creation | Automatic PR creation |
| **Conflict Detection** | Standard Git | Semantic merge |
| **PR Comments** | GitHub Actions results | Gearset validation results |

---

## 9. Decision Matrix

```mermaid
quadrantChart
    title Tool Selection Based on Team Profile
    x-axis Low Git Expertise --> High Git Expertise
    y-axis Small Budget --> Large Budget
    quadrant-1 Gearset Preferred
    quadrant-2 Either Tool Works
    quadrant-3 sfdx-hardis Preferred
    quadrant-4 sfdx-hardis with Support
    Gearset: [0.3, 0.8]
    sfdx-hardis: [0.8, 0.2]
    Hybrid: [0.6, 0.5]
```

### When to Choose sfdx-hardis with GitHub:
- ✅ Budget constraints or cost-conscious organizations
- ✅ Teams with strong Git and GitHub expertise
- ✅ Need for maximum flexibility in GitHub Actions
- ✅ Preference for standard GitHub PR workflows
- ✅ Already comfortable with GitHub CLI and Actions
- ✅ Multiple projects (reusable workflow templates)

### When to Choose Gearset with GitHub:
- ✅ Teams with mixed technical expertise (admins + devs)
- ✅ Organizations prioritizing automation over customization
- ✅ Need for automatic PR creation and back-propagation
- ✅ Complex release management with bundled features
- ✅ Requirement for semantic merge capabilities
- ✅ Preference for visual tools while using GitHub

---

## 10. Conclusion

Both sfdx-hardis and Gearset provide robust GitOps solutions for Salesforce CI/CD with GitHub, long-lived branches, and permanent orgs. The choice depends on:

| Factor | Favor sfdx-hardis | Favor Gearset |
|--------|-------------------|---------------|
| Budget | Limited | Available |
| GitHub Actions expertise | High | Low-Medium |
| PR automation needs | Standard is fine | Maximum automation |
| Conflict resolution | Standard Git OK | Need semantic merge |
| Team GitHub skills | Strong | Mixed |
| Customization needs | High | Standard |

**Key Takeaway**: 
- **sfdx-hardis** provides a **standard GitHub PR experience** with manual PR creation between branches and GitHub Actions for validation/deployment
- **Gearset** provides an **automated PR experience** with automatic promotion PRs, semantic merge, and automatic back-propagation PRs

The **retrofit (sfdx-hardis)** and **back-propagation (Gearset)** mechanisms serve the same fundamental purpose—keeping environments synchronized via GitHub PRs—but differ in automation level:
- **Retrofit**: Scheduled GitHub Action creates a PR from production changes
- **Back-propagation**: Immediate automatic PRs after merge to main
