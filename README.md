> ### Confidentiality Notice
>
> This repository contains an anonymized portfolio case study based on a real-world project.
>
> Company names, client information, URLs, credentials, proprietary business logic, and sensitive implementation details have been removed or modified to protect confidentiality.
>
> Any code snippets included are examples only and do not represent any protected production codebase.

---

# Pipeline CI/CD v1.1.1 — Environment-Aware Build & E2E Quality Gate

### Vite + Cypress + Azure DevOps + Azure Static Web Apps

Refactor of an existing CI/CD pipeline to align application builds, Cypress E2E validation, environment configuration, and deployment targets according to the branch being processed.

---

## Project

The original pipeline was designed to always generate a Development build and execute Cypress against that build before determining the deployment target.

As the automation framework evolved, this approach introduced a limitation: **Production deployments were being validated using a Development build and Development configuration**.

The pipeline was therefore redesigned so that the build, Cypress configuration, and deployment target are aligned with the branch being processed.

---

Sí. Para este case study, considerando que queremos mantenerlo **compacto y fácil de navegar**, usaría este índice:

## Table of Contents

1. [Project](#project)
2. [Objective](#objective)
3. [My Contribution](#my-contribution)
4. [Stack](#stack)
5. [Previous Architecture — v1.0.1](#previous-architecture--v101)
6. [Why the Architecture Changed](#why-the-architecture-changed)
7. [New Architecture — v1.1.1](#new-architecture--v111)
8. [Environment-Aware Cypress Configuration](#environment-aware-cypress-configuration)
9. [Local Execution vs Pipeline Execution](#local-execution-vs-pipeline-execution)
10. [Cypress as the Deployment Quality Gate](#cypress-as-the-deployment-quality-gate)
11. [New CI/CD Flow](#new-cicd-flow)
12. [Deployment Strategy](#deployment-strategy)
13. [Pipeline Architecture](#pipeline-architecture)
14. [Pull Request Checks vs Pipeline Quality Gate](#pull-request-checks-vs-pipeline-quality-gate)
15. [Key Technical Decisions](#key-technical-decisions)
16. [Example Cypress Integration](#example-cypress-integration)
17. [Results](#results)
18. [Skills Demonstrated](#skills-demonstrated)
19. [Conclusion](#conclusion)


---

## Objective

Refactor the existing CI/CD workflow so that:

* `dev` generates and validates a Development build.
* `main` generates and validates a Production build.
* Cypress uses the corresponding environment configuration.
* The generated artifact is validated before deployment.
* Cypress failures prevent subsequent deployment stages.
* The same test suite can operate against DEV and PROD without changing its internal variable naming.

---

## My Contribution

Although pipeline architecture was not my primary role, I went beyond implementing the Cypress test stages and investigated how the CI/CD workflow operated as a whole.

I designed and implemented the required YAML changes to integrate the E2E framework into the existing delivery process, including:

→ Branch-based build conditions.

→ Environment-specific Cypress execution.

→ DEV and PROD pipeline variable mapping.

→ Artifact management between stages.

→ Cypress execution as a deployment quality gate.

→ Deployment conditions based on the processed branch.

→ Separation between local debugging configuration and pipeline configuration.

→ Validation of the resulting Build → Test → Deploy workflow.

---

## Stack

```txt
- Azure DevOps
- YAML Pipelines
- Vite
- Cypress
- JavaScript
- Azure Static Web Apps
- GitHub
- Mochawesome
```

---

# Previous Architecture — v1.0.1

The original pipeline always generated a Development build regardless of the target branch.

```text
Build DEV
    ↓
vite-dist
    ↓
localhost:8080
    ↓
Cypress DEV
    ↓
Deploy according to branch
```

The original strategy was:

| Situation      | Initial Build | Cypress | Build PROD | Deploy |
| -------------- | ------------- | ------- | ---------- | ------ |
| PR → `dev`     | DEV           | DEV     | —          | —      |
| Merge → `dev`  | DEV           | DEV     | —          | DEV    |
| PR → `main`    | DEV           | DEV     | —          | —      |
| Merge → `main` | DEV           | DEV     | PROD       | PROD   |

This provided a controlled Development build for E2E validation, but Production changes were still validated against the Development build before the Production artifact was generated.

---

# Why the Architecture Changed

The main limitation was the mismatch between the artifact being tested and the artifact eventually deployed to Production.

The previous process could be represented as:

```text
Build DEV
    ↓
Test DEV
    ↓
Build PROD
    ↓
Deploy PROD
```

The new strategy removes that mismatch by making the build itself environment-aware.

The resulting architecture is:

```text
Branch
   ↓
Environment-specific Build
   ↓
Environment-specific Cypress Configuration
   ↓
Quality Gate
   ↓
Deployment to Same Environment
```

This ensures that the artifact being validated corresponds to the environment it is intended to serve.

---

# New Architecture — v1.1.1

The Build stage now determines which build configuration to execute based on the branch being processed.

```yaml
- script: npm run build:dev
  displayName: "Build (dev)"
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/dev')

- script: npm run build:prod
  displayName: "Build (prod)"
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
```

Both paths publish the resulting application as the same pipeline artifact:

```text
vite-dist
```

The artifact is then served locally inside the pipeline:

```text
vite-dist
    ↓
localhost:8080
    ↓
Cypress
```

Therefore, Cypress continues validating the **artifact generated by the pipeline**, rather than an externally deployed application.

---

# Environment-Aware Cypress Configuration

The test suites maintain a **single common variable nomenclature**.

For example:

```javascript
Cypress.env("USER")
Cypress.env("PASSWORD")
Cypress.env("API_URL")
```

The suites do not need separate variables such as:

```text
USER_PROD
USER_DEV
API_URL_PROD
API_URL_DEV
```

Instead, the pipeline determines which values are injected during execution.

### Development

```yaml
condition: eq(variables['Build.SourceBranch'], 'refs/heads/dev')

CYPRESS_API_URL=$(CYPRESS_API_URL)
CYPRESS_USER=$(CYPRESS_USER)
CYPRESS_PASSWORD=$(CYPRESS_PASSWORD)
```

### Production

```yaml
condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')

CYPRESS_API_URL=$(CYPRESS_API_URL_PROD)
CYPRESS_USER=$(CYPRESS_USER_PROD)
CYPRESS_PASSWORD=$(CYPRESS_PASSWORD_PROD)
```

This keeps the automation code environment-agnostic while allowing Azure DevOps to provide the appropriate configuration.

The specific credentials and URLs are maintained through pipeline variables and are not exposed in the repository.

---

# Local Execution vs Pipeline Execution

The existing `package.json` scripts remain primarily intended for **local execution and debugging**.

For example:

```text
CYPRESS_USER="user@example.com"
```

The pipeline does not rely on those local values.

Instead, Azure DevOps injects the appropriate variables according to the branch condition.

This creates a clear separation:

```text
Local Execution
      ↓
package.json / local environment variables

Pipeline Execution
      ↓
Azure DevOps variables
      ↓
Branch condition
      ↓
DEV or PROD configuration
```

This allows the same test suites to execute without modifying their internal logic between environments.

---

# Cypress as the Deployment Quality Gate

One of the key changes in v1.1.1 was keeping explicitly:

```yaml
continueOnError: false
```

This makes Cypress an actual deployment gate rather than only an evidence-generation step.

```text
Build
  ↓
Cypress
  ↓
Tests successful?
  ├── YES → Test succeeded → Deploy
  │
  └── NO  → Test failed → STOP
                         ↓
                      No Deploy
```

The relevant configuration is explicitly maintained in both execution paths.

A Cypress failure therefore causes the `Test` stage to fail, preventing the dependent deployment stage from executing.

---

# New CI/CD Flow

## Development

When the pipeline processes the `dev` branch:

```text
Build DEV
    ↓
vite-dist
    ↓
localhost:8080
    ↓
Cypress + DEV configuration
    ↓
PASS → Deploy DEV
FAIL → Pipeline stops
```

## Production

When the pipeline processes the `main` branch:

```text
Build PROD
    ↓
vite-dist
    ↓
localhost:8080
    ↓
Cypress + PROD configuration
    ↓
PASS → Deploy PROD
FAIL → Pipeline stops
```

### Summary

| Merge to Branch | Build | Cypress Configuration | Deployment |
| --------------- | ----- | --------------------- | ---------- |
| `dev`           | DEV   | DEV                   | DEV        |
| `main`          | PROD  | PROD                  | PROD       |

The key difference from v1.0.1 is that **the build and E2E validation now correspond to the same target environment**.

---

# Deployment Strategy

Both deployment stages depend on the successful completion of the `Test` stage.

### Development

```yaml
- stage: DeployDev
  displayName: "Deploy to Azure Static Web App (Dev)"
  dependsOn: Test
  condition: and(
    succeeded(),
    eq(variables['Build.SourceBranchName'], 'dev')
  )
```

### Production

```yaml
- stage: DeployProd
  displayName: "Deploy to Azure Static Web App (Prod)"
  dependsOn: Test
  condition: and(
    succeeded(),
    eq(variables['Build.SourceBranchName'], 'main')
  )
```

This creates an explicit dependency between successful E2E validation and deployment.

---

# Pipeline Architecture

The resulting architecture can be summarized as:

```text
                    ┌───────────────┐
                    │ Branch / Merge│
                    └───────┬───────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
              dev branch           main branch
                 │                     │
                 ▼                     ▼
            Build DEV             Build PROD
                 │                     │
                 └──────────┬──────────┘
                            ▼
                       vite-dist
                            │
                            ▼
                      localhost:8080
                            │
                 ┌──────────┴──────────┐
                 │                     │
            Cypress DEV           Cypress PROD
                 │                     │
                 └──────────┬──────────┘
                            ▼
                     Quality Gate
                       /       \
                    PASS        FAIL
                     │            │
                     ▼            ▼
                  Deploy       STOP
```

---

# Pull Request Checks vs Pipeline Quality Gate

Pull Request checks and the post-merge pipeline are treated as separate validation mechanisms.

The deployment decision is controlled by the pipeline executed against the target branch:

```text
Pull Request
     ↓
PR Checks
     ↓
Merge
     ↓
Branch Pipeline
     ↓
Build
     ↓
Cypress
     ↓
Quality Gate
   /       \
PASS       FAIL
 ↓           ↓
Deploy      STOP
```

PR checks may be affected by repository-specific configuration, branch policies, permissions, or the execution context of the Pull Request.

Those configurations are outside the scope of this automation framework.

The **official quality gate controlling deployment** is the Cypress execution within the pipeline's `Test` stage after the change has been merged into its target branch.

> **Note:** Whether a Pull Request can be completed when an individual check fails is determined by the repository's Branch Policies. This configuration is independent from the Cypress framework and the deployment logic documented in this case study.

---

# Key Technical Decisions

### 1. Branch-Based Build Selection

The build configuration is selected according to the branch being processed.

```text
dev  → build:dev
main → build:prod
```

This eliminates the previous requirement to always build Development first.

### 2. Single Artifact Naming

Both environments publish:

```text
vite-dist
```

This simplifies artifact handling because the deployment stage consumes the artifact generated by the corresponding branch pipeline.

### 3. Environment Selection at Pipeline Level

The suites maintain a common variable interface while Azure DevOps determines the actual values.

This avoids environment-specific branching inside the test implementation.

### 4. Strict Cypress Quality Gate

```yaml
continueOnError: false
```

Cypress failures prevent subsequent deployment stages from executing.

### 5. Local Artifact Validation

The application continues to be served through:

```text
localhost:8080
```

This preserves the original benefit of testing the actual generated build artifact before deployment.

---

# Example Cypress Integration

### Development

```yaml
- script: |
    cd automated-tests

    export CYPRESS_BASE_URL=url_development
    export CYPRESS_API_URL=$(CYPRESS_API_URL)
    export CYPRESS_USER=$(CYPRESS_USER)
    export CYPRESS_PASSWORD=$(CYPRESS_PASSWORD)

    npx cypress run \
      --reporter mochawesome \
      --reporter-options "reportDir=cypress/reports,reportFilename=$(browser)Report,timestamp=mmddyyyy_HHMMss,overwrite=true,html=true,json=true"

  displayName: "Run Cypress Tests Dev"
  continueOnError: false
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/dev')
```

### Production

```yaml
- script: |
    cd automated-tests

    export CYPRESS_BASE_URL=url_production
    export CYPRESS_API_URL=$(CYPRESS_API_URL_PROD)
    export CYPRESS_USER=$(CYPRESS_USER_PROD)
    export CYPRESS_PASSWORD=$(CYPRESS_PASSWORD_PROD)

    npx cypress run \
      --reporter mochawesome \
      --reporter-options "reportDir=cypress/reports,reportFilename=$(browser)Report,timestamp=mmddyyyy_HHMMss,overwrite=true,html=true,json=true"

  displayName: "Run Cypress Tests Prod"
  continueOnError: false
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
```

Sensitive values are represented generically in this case study and remain managed through Azure DevOps pipeline variables.

---

# Results

The v1.1.1 refactor improved the alignment between the application artifact, test configuration, and deployment environment.

### Before

```text
Build DEV
   ↓
Test DEV
   ↓
Build PROD
   ↓
Deploy PROD
```

### After

```text
DEV:
Build DEV
   ↓
Test DEV
   ↓
Deploy DEV
```

```text
PROD:
Build PROD
   ↓
Test PROD
   ↓
Deploy PROD
```

### Main improvements

→ Production deployments are validated against a Production build.

→ Development deployments are validated against a Development build.

→ Cypress receives environment-specific configuration without modifying the test suites.

→ Cypress failures explicitly prevent deployment.

→ The generated artifact remains the object being validated before deployment.

→ Local debugging and CI/CD configuration remain separated.

→ The pipeline architecture became more predictable and environment-aware.

---

# Skills Demonstrated

* Azure DevOps YAML pipeline design and refactoring.
* CI/CD workflow analysis.
* Cypress E2E integration.
* Branch-based pipeline conditions.
* Environment-specific configuration management.
* Pipeline artifact management.
* Quality gate implementation.
* Deployment dependency management.
* CI/CD debugging and troubleshooting.
* Separation of test implementation from environment configuration.

---

# Conclusion

The v1.1.1 pipeline refactor addressed a limitation in the original CI/CD architecture by aligning the **build, E2E validation, configuration, and deployment target** with the branch being processed.

The resulting workflow validates the actual environment-specific artifact before deployment while maintaining a common automation suite across environments.

Although pipeline architecture was not the primary responsibility of the QA role, the implementation required understanding how the application build, Cypress execution, pipeline variables, artifacts, stages, conditions, and deployment process interacted as a complete delivery workflow.

The final architecture establishes a clear quality gate:

```text
Build Environment
       ↓
Environment-Specific Cypress Validation
       ↓
Quality Gate
       ↓
Deployment
```

This allows Cypress to function not only as an automation framework, but as an integrated **quality control mechanism within the CI/CD delivery process**.

---

<div style="text-align:center; color:#888; font-size:10px;">
    QA Documentation | Axel Van Dyck | QA Engineer 
</div>
