# GitOps Delivery Comparison: sfdx-hardis vs Gearset

## Executive Summary

This document compares two Salesforce DevOps approaches for managing deliveries with long-lived branches and permanent orgs (INT, UAT, PROD). Both tools support similar GitOps patterns but differ significantly in implementation, philosophy, and feature sets.

---

## 1. Branch Strategy Overview

### 1.1 sfdx-hardis: Long Branches with Permanent Orgs

sfdx-hardis uses a **BUILD/RUN** separation model with long-lived branches mapped to permanent Salesforce orgs.

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
    
    subgraph "Git Branches"
        FEATURE[feature/*]
        INTEGRATION[integration]
        UAT[uat]
        PREPROD[preprod]
        MAIN[main/master]
    end
    
    DEV_SB -->|"Merge Request"| FEATURE
    FEATURE -->|"MR to integration"| INTEGRATION
    INTEGRATION -->|"CI Deploy"| INT_ORG
    INTEGRATION -->|"MR to uat"| UAT
    UAT -->|"CI Deploy"| UAT_ORG
    UAT -->|"MR to preprod"| PREPROD
    PREPROD -->|"CI Deploy"| PREPROD_ORG
    PREPROD -->|"MR to main"| MAIN
    MAIN -->|"CI Deploy"| PROD_ORG
    
    style PROD_ORG fill:#2ecc71
    style PREPROD_ORG fill:#f39c12
    style UAT_ORG fill:#3498db
    style INT_ORG fill:#9b59b6
```

### 1.2 Gearset: Expanded Branching Model

Gearset uses an **expanded branching model** with promotion branches and automatic back-propagation.

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
    
    subgraph "Git Branches"
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
    PROMO1 -->|"Merge"| INT_BR
    INT_BR -->|"CI Deploy"| INT_ORG
    
    FEATURE1 -->|"Auto PR"| PROMO2
    PROMO2 -->|"Merge"| UAT_BR
    UAT_BR -->|"CI Deploy"| UAT_ORG
    
    UAT_BR -->|"Release PR"| MAIN
    MAIN -->|"CI Deploy"| PROD_ORG
    
    style PROD_ORG fill:#2ecc71
    style UAT_ORG fill:#3498db
    style INT_ORG fill:#9b59b6
```

---

## 2. Retrofit vs Back-Propagation

### 2.1 sfdx-hardis Retrofit Process

The retrofit mechanism in sfdx-hardis retrieves production changes and propagates them back to lower environments (typically preprod/uat) to keep environments synchronized.

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant PREPROD as preprod branch
    participant MAIN as main branch
    participant PROD as Production Org
    participant CI as CI/CD Pipeline
    participant RETROFIT as Retrofit Job
    
    Note over DEV,RETROFIT: Normal Deployment Flow
    DEV->>PREPROD: Merge Request
    PREPROD->>MAIN: Merge Request
    CI->>PROD: Deploy to Production
    
    Note over DEV,RETROFIT: Retrofit Process (Post-Production)
    RETROFIT->>PROD: Retrieve metadata from Production
    RETROFIT->>RETROFIT: Compare with productionBranch (main)
    RETROFIT->>RETROFIT: Identify changed metadata types
    RETROFIT->>PREPROD: Create Merge Request with changes
    
    Note over PREPROD: Retrofit MR contains:<br/>- CompactLayout<br/>- CustomField<br/>- CustomObject<br/>- FlexiPage<br/>- PermissionSet<br/>- ValidationRule<br/>- etc.
    
    DEV->>PREPROD: Review & Merge Retrofit MR
    CI->>PREPROD: Deploy retrofit to PreProd Org
```

**sfdx-hardis Retrofit Configuration (.sfdx-hardis.yml):**
```yaml
productionBranch: master
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

### 2.2 Gearset Back-Propagation Process

Gearset automatically creates back-propagation PRs after changes are merged to production, keeping all upstream environments synchronized.

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant FEATURE as Feature Branch
    participant INT as integration
    participant UAT as uat
    participant MAIN as main
    participant PROD as Production Org
    participant GS as Gearset Pipelines
    
    Note over DEV,GS: Forward Promotion
    DEV->>FEATURE: Commit changes
    GS->>INT: Auto-create Promotion PR
    GS->>INT: Merge & Deploy to INT Org
    GS->>UAT: Auto-create Promotion PR
    GS->>UAT: Merge & Deploy to UAT Org
    GS->>MAIN: Auto-create Promotion PR
    GS->>PROD: Merge & Deploy to Production
    
    Note over DEV,GS: Automatic Back-Propagation
    GS->>GS: Detect merge to main
    GS->>UAT: Auto-create Back-prop PR
    GS->>INT: Auto-create Back-prop PR
    
    Note over INT,UAT: Back-prop PRs contain:<br/>- All feature commits<br/>- Conflict resolutions<br/>- Merge commits
```

---

## 3. Release Branch Model Comparison

### 3.1 sfdx-hardis: UAT-based Release

In sfdx-hardis, the release is essentially the state of the UAT branch at a given point, which then flows to preprod and production.

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
    
    F1 --> INT
    F2 --> INT
    F3 --> INT
    INT -->|"Merge to uat"| UAT_RELEASE
    
    UAT_RELEASE -->|"UAT Testing<br/>Complete"| PREPROD
    PREPROD -->|"Final<br/>Validation"| MAIN
    
    style UAT_RELEASE fill:#e74c3c,color:#fff
    style PREPROD fill:#f39c12
    style MAIN fill:#2ecc71
```

### 3.2 Gearset: Explicit Release Branch

Gearset provides an explicit release branch mechanism for bundling multiple features into a single coordinated release.

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
    
    F1 -->|"Promote to INT"| INT
    F2 -->|"Promote to INT"| INT
    F3 -->|"Promote to INT"| INT
    
    INT -->|"Promote to UAT"| UAT
    
    F1 -->|"Add to Release"| REL
    F2 -->|"Add to Release"| REL
    F3 -->|"Add to Release"| REL
    
    REL -->|"Scheduled<br/>Deployment"| MAIN
    
    MAIN -->|"Back-prop"| UAT
    MAIN -->|"Back-prop"| INT
    
    style REL fill:#e74c3c,color:#fff
    style MAIN fill:#2ecc71
```

---

## 4. Complete Workflow Diagrams

### 4.1 sfdx-hardis Complete CI/CD Flow

```mermaid
flowchart TB
    subgraph "Developer Workflow"
        START((Start))
        CREATE_SB[Create Source-tracked<br/>Sandbox from INT]
        DEVELOP[Develop in Sandbox]
        PULL[Pull changes to local]
        COMMIT[Commit to feature branch]
        CREATE_MR[Create Merge Request<br/>to integration]
    end
    
    subgraph "CI/CD Pipeline - BUILD"
        VALIDATE_INT[Validate against INT]
        MERGE_INT[Merge to integration]
        DEPLOY_INT[Deploy to INT Org]
        
        CREATE_MR_UAT[Create MR to uat]
        VALIDATE_UAT[Validate against UAT]
        MERGE_UAT[Merge to uat]
        DEPLOY_UAT[Deploy to UAT Org]
    end
    
    subgraph "CI/CD Pipeline - RUN"
        CREATE_MR_PREPROD[Create MR to preprod]
        VALIDATE_PREPROD[Validate against PreProd]
        MERGE_PREPROD[Merge to preprod]
        DEPLOY_PREPROD[Deploy to PreProd Org]
        
        CREATE_MR_MAIN[Create MR to main]
        VALIDATE_PROD[Validate against PROD]
        MERGE_MAIN[Merge to main]
        DEPLOY_PROD[Deploy to Production]
    end
    
    subgraph "Retrofit Process"
        TRIGGER_RETROFIT[Trigger Retrofit Job]
        RETRIEVE_PROD[Retrieve from Production]
        CREATE_RETROFIT_MR[Create Retrofit MR<br/>to retrofitBranch]
        MERGE_RETROFIT[Merge Retrofit]
    end
    
    START --> CREATE_SB --> DEVELOP --> PULL --> COMMIT --> CREATE_MR
    CREATE_MR --> VALIDATE_INT
    VALIDATE_INT -->|Pass| MERGE_INT --> DEPLOY_INT
    DEPLOY_INT --> CREATE_MR_UAT --> VALIDATE_UAT
    VALIDATE_UAT -->|Pass| MERGE_UAT --> DEPLOY_UAT
    DEPLOY_UAT --> CREATE_MR_PREPROD --> VALIDATE_PREPROD
    VALIDATE_PREPROD -->|Pass| MERGE_PREPROD --> DEPLOY_PREPROD
    DEPLOY_PREPROD --> CREATE_MR_MAIN --> VALIDATE_PROD
    VALIDATE_PROD -->|Pass| MERGE_MAIN --> DEPLOY_PROD
    
    DEPLOY_PROD --> TRIGGER_RETROFIT
    TRIGGER_RETROFIT --> RETRIEVE_PROD --> CREATE_RETROFIT_MR --> MERGE_RETROFIT
```

### 4.2 Gearset Complete Pipeline Flow

```mermaid
flowchart TB
    subgraph "Developer Workflow"
        START((Start))
        CREATE_FEATURE[Create Feature Branch<br/>from main]
        DEVELOP[Develop in Dev Sandbox]
        COMMIT[Commit changes]
        PUSH[Push to remote]
    end
    
    subgraph "Gearset Pipelines - Forward Promotion"
        AUTO_PR_INT[Auto-create PR<br/>to integration]
        SEMANTIC_MERGE_INT[Semantic Merge Check]
        VALIDATE_INT[Validate against INT Org]
        PROMOTE_INT[Promote to INT]
        DEPLOY_INT[CI Deploy to INT Org]
        
        AUTO_PR_UAT[Auto-create PR<br/>to uat]
        SEMANTIC_MERGE_UAT[Semantic Merge Check]
        VALIDATE_UAT[Validate against UAT Org]
        PROMOTE_UAT[Promote to UAT]
        DEPLOY_UAT[CI Deploy to UAT Org]
        
        AUTO_PR_MAIN[Auto-create PR<br/>to main]
        VALIDATE_PROD[Validate against PROD]
        PROMOTE_MAIN[Promote to main]
        DEPLOY_PROD[CI Deploy to Production]
    end
    
    subgraph "Release Option"
        CREATE_RELEASE[Create Release Branch]
        ADD_FEATURES[Add Features to Release]
        VALIDATE_RELEASE[Validate Release]
        SCHEDULE_DEPLOY[Schedule Deployment]
    end
    
    subgraph "Back-Propagation"
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

## 5. Hotfix Handling Comparison

### 5.1 sfdx-hardis Hotfix Process

```mermaid
flowchart LR
    subgraph "Hotfix in RUN Layer"
        ISSUE[Production Issue]
        CREATE_HF[Create hotfix branch<br/>from preprod]
        FIX[Apply fix]
        MR_PREPROD[MR to preprod]
        DEPLOY_PP[Deploy to PreProd]
        MR_MAIN[MR to main]
        DEPLOY_PROD[Deploy to Production]
    end
    
    subgraph "Retrofit to BUILD"
        RETROFIT[Retrofit job]
        MR_INT[MR to integration]
        SYNC[Sync BUILD layer]
    end
    
    ISSUE --> CREATE_HF --> FIX --> MR_PREPROD --> DEPLOY_PP
    DEPLOY_PP --> MR_MAIN --> DEPLOY_PROD
    DEPLOY_PROD --> RETROFIT --> MR_INT --> SYNC
    
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
        DEPLOY_PROD[Deploy to Production]
    end
    
    subgraph "Automatic Sync"
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

## 6. Feature Comparison Table

| Feature | sfdx-hardis | Gearset |
|---------|-------------|---------|
| **Licensing** | Open-source (Free) | Commercial (Paid) |
| **Interface** | CLI + VS Code Extension | Web UI + CLI support |
| **Branch Strategy** | BUILD/RUN separation | Expanded branching model |
| **Promotion Branches** | Manual MR creation | Automatic (gs-pipeline/*) |
| **Conflict Resolution** | Manual Git-based | Semantic merge (automated) |
| **Back-sync Mechanism** | Retrofit (scheduled/manual) | Back-propagation (automatic) |
| **Release Branches** | Implicit (UAT state) | Explicit release branch |
| **CI/CD Providers** | GitLab, GitHub, Azure, Bitbucket | All major providers |
| **Validation** | sfdx/sf commands | Built-in + external tools |
| **Metadata Comparison** | CLI-based | Visual side-by-side |
| **Conflict UI** | Git/IDE native | Built-in visual resolver |
| **Test Automation** | Apex tests via CI | Apex + UI test integration |
| **Static Analysis** | Via MegaLinter/PMD | Built-in + PMD/CodeScan |
| **Quick Deploy** | Supported | Supported |
| **Scratch Org Support** | Yes | Yes |
| **Learning Curve** | Steeper (Git knowledge required) | Gentler (UI abstracts Git) |
| **Customization** | Highly customizable YAML | UI-configured |
| **Support** | Community + paid services | Dedicated support team |

---

## 7. Pros and Cons Analysis

### 7.1 sfdx-hardis

```mermaid
mindmap
  root((sfdx-hardis))
    Pros
      Free & Open Source
      Full control over process
      Transparent operations
      No vendor lock-in
      Highly customizable
      Native Git workflows
      Strong community
      CLI automation friendly
    Cons
      Steeper learning curve
      Manual conflict resolution
      Setup complexity
      Git expertise required
      Less visual tooling
      Retrofit is semi-manual
      Documentation scattered
```

#### Detailed Pros:
1. **Cost-effective**: No licensing costs, enterprise-ready features at zero cost
2. **Transparency**: All operations run native SFDX commands, visible in logs
3. **Flexibility**: Highly configurable via YAML, adaptable to any org structure
4. **No vendor lock-in**: Standard Git and Salesforce CLI, portable knowledge
5. **BUILD/RUN separation**: Clear distinction between development and production pipelines
6. **VS Code integration**: Visual interface for non-CLI users
7. **Active development**: Regular updates, responsive maintainers

#### Detailed Cons:
1. **Git expertise required**: Team needs solid Git fundamentals
2. **Manual retrofit management**: Scheduled jobs need configuration and monitoring
3. **Conflict resolution**: Standard Git conflicts, no intelligent merge
4. **Setup investment**: Initial configuration requires DevOps knowledge
5. **Limited visual diff**: Relies on Git/IDE for metadata comparison

### 7.2 Gearset

```mermaid
mindmap
  root((Gearset))
    Pros
      User-friendly UI
      Semantic merge
      Automatic back-prop
      Visual conflict resolution
      Release branch support
      Integrated testing
      Strong support
      Lower learning curve
    Cons
      License costs
      Vendor dependency
      Less customizable
      Black-box operations
      Limited CLI access
      Promotion branch complexity
      Can abstract too much
```

#### Detailed Pros:
1. **Semantic merge**: Understands Salesforce metadata, reduces false conflicts
2. **Automatic back-propagation**: Environments stay synchronized automatically
3. **Visual tools**: Side-by-side comparison, intuitive conflict resolution
4. **Release management**: Explicit release branches with scheduling
5. **Integrated testing**: Built-in Apex tests, static analysis, UI test integration
6. **Lower barrier to entry**: Admins can participate without deep Git knowledge
7. **Enterprise support**: Dedicated DevOps experts available

#### Detailed Cons:
1. **Cost**: Significant licensing investment for enterprise teams
2. **Vendor lock-in**: Workflow depends on Gearset's infrastructure
3. **Reduced transparency**: Some operations are abstracted
4. **Promotion branch overhead**: Additional branches can clutter repository
5. **Customization limits**: Less flexible than code-based solutions
6. **Learning Gearset way**: Must understand their branching model

---

## 8. Decision Matrix

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

### When to Choose sfdx-hardis:
- ✅ Budget constraints or cost-conscious organizations
- ✅ Teams with strong Git and DevOps expertise
- ✅ Need for maximum flexibility and customization
- ✅ Preference for open-source and transparent operations
- ✅ Already using CLI-heavy workflows
- ✅ Multiple similar projects (reusable configuration)

### When to Choose Gearset:
- ✅ Teams with mixed technical expertise (admins + devs)
- ✅ Organizations prioritizing time-to-value over cost
- ✅ Need for visual tools and reduced Git complexity
- ✅ Complex release management with bundled features
- ✅ Requirement for dedicated vendor support
- ✅ Preference for automatic environment synchronization

---

## 9. Migration Considerations

### 9.1 From sfdx-hardis to Gearset

```mermaid
flowchart LR
    A[Keep existing branches] --> B[Connect orgs to Gearset]
    B --> C[Map branches to environments]
    C --> D[Configure CI jobs]
    D --> E[Migrate to Pipelines]
    E --> F[Retire hardis config]
```

### 9.2 From Gearset to sfdx-hardis

```mermaid
flowchart LR
    A[Export branch structure] --> B[Setup hardis config]
    B --> C[Configure CI pipelines]
    C --> D[Setup retrofit jobs]
    D --> E[Train team on CLI]
    E --> F[Migrate incrementally]
```

---

## 10. Conclusion

Both sfdx-hardis and Gearset provide robust GitOps solutions for Salesforce CI/CD with long-lived branches and permanent orgs. The choice between them depends on:

| Factor | Favor sfdx-hardis | Favor Gearset |
|--------|-------------------|---------------|
| Budget | Limited | Available |
| Team Git skills | High | Mixed |
| Customization needs | High | Standard |
| Visual tooling | Optional | Required |
| Vendor relationship | Independence | Partnership |
| Time to implement | Can invest | Need speed |

The **retrofit (sfdx-hardis)** and **back-propagation (Gearset)** mechanisms serve the same fundamental purpose—keeping environments synchronized—but differ in automation level and triggering mechanisms. Organizations should evaluate their specific needs, team capabilities, and long-term strategy when selecting between these tools.
