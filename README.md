# Azure Enterprise DevOps Platform

Reusable GitHub Actions templates for Java CI, Azure authentication, Azure Container Registry (ACR), Azure Key Vault, and AKS deployment. This README describes both the current repository and a proposed target platform design for Azure. The design is a specification: several of its security and release gates are not implemented in the current workflows yet.

## Repository contents today

```text
.github/
  actions/
    azure/
      acr/build/           ACR build composite action
      akv/secrets/         Fetch secrets from Azure Key Vault
      aks/deploy/          Empty placeholder action
      login/               Azure CLI login using MSI or service principal
    setup/
      maven-install/       Install Java and Maven on an Ubuntu runner
  workflows/
    maven-ci.yml           Maven, SonarQube, one ACR image build, and digest output
    aks-deploy.yml         Key Vault login, AKS credentials, and Kustomize deployment
```

Reusable workflows are called with GitHub Actions `workflow_call`. Application Dockerfiles and Kustomize bases/overlays are expected in the checked-out application repository, not this template repository.

### Current workflows and known gaps

**Maven CI** (`.github/workflows/maven-ci.yml`) checks out the requested repo/ref, installs Java 21 and Maven 3.9.16, runs `mvn clean install` and SonarQube, then builds and pushes one image tagged with the commit SHA. It returns the ACR image digest and immutable image reference to the caller. Required inputs include the source repo/ref, SonarQube project/host, ACR name/image/subscription, and build Key Vault. Required secret: `sonar_token`.

**AKS deployment** (`.github/workflows/aks-deploy.yml`) is called once per GitHub Environment. It uses the self-hosted runner's system-assigned identity to read target login credentials from Key Vault, gets credentials for the environment's AKS cluster, applies its Kustomize overlay, sets the Deployment image to the supplied immutable digest, and waits for rollout. A failed rollout attempts `kubectl rollout undo`.

Required inputs: `repo_name`, `branch_name`, and `environment_name`. AKS and Key Vault settings can be passed explicitly or supplied as GitHub Environment variables: `AKS_NAME`, `AKS_RESOURCE_GROUP`, `AKS_SUBSCRIPTION`, `AKS_NAMESPACE`, and `KEYVAULT_NAME`. `KEYVAULT_SUBSCRIPTION` is optional; `AUTH_METHOD` selects `msi` or `service_principal` and defaults to `msi`. Key Vault secret names default to `MSI_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, and `AZURE_TENANT_ID`. For MSI target login, the user-assigned identity must be attached to the runner; for service-principal login, its client ID, secret, and tenant ID must be in Key Vault. When an `image_reference` is provided, also set `CONTAINER_NAME` and `DEPLOYMENT_NAME` in that Environment. The runner needs the `selfhosted-ubuntu` label, Azure CLI, kubectl, network access to AKS, and a system-assigned managed identity with **Key Vault Secrets User** access. The target identity needs permission to get AKS credentials and Kubernetes RBAC rights in the namespace. Separately, ensure the AKS kubelet identity has **AcrPull** on the registry.

The application caller at `maven-application/.github/workflows/ci-cd.yml` runs `push`/manual CI, then promotes the same digest through `dev`, `uat`, and `prod`. Set repository variables `SONAR_HOST_URL`, `SONAR_PROJECT_KEY`, `ACR_NAME`, `ACR_IMAGE_NAME`, `ACR_SUBSCRIPTION_ID`, and `BUILD_KEYVAULT_NAME`; optionally set `BUILD_KEYVAULT_SUBSCRIPTION` and `BUILD_AUTH_METHOD` (defaults to `msi`). Set `SONAR_TOKEN` as a GitHub secret. The build runner identity must read the build Key Vault and have **AcrPush** on the registry.

Create GitHub Environments named `dev`, `uat`, and `prod`. Set the AKS variables above in each environment; set `SMOKE_TEST_URL` in `dev` and `uat`. Smoke tests run on `selfhosted-ubuntu`, so that runner must be able to reach the application endpoints. Configure required reviewers on `prod` to gate production deployment. The caller fails closed if a smoke-test URL is missing or does not return a successful HTTP response.

The Maven application checkout currently contains empty `pom.xml`, Dockerfile, and Kustomize files, so the workflow is a scaffold and cannot complete until those files are populated. Also configure the deployment environment variables to match the actual Kubernetes Deployment/container names.

The Azure login action retains direct MSI/service-principal inputs for existing callers. The AKS and build workflows use its Key Vault mode, which requires the runner's system-assigned identity for bootstrap. The `prod` gate only applies after required reviewers are configured in GitHub Environment settings. The workflow currently uses rollout status and rollback, not canary or blue/green delivery. Other controls in this README—such as IaC policy gates, SBOM generation/signing, and DORA collection—are design recommendations, not implemented workflow gates yet.

## 1. Platform design

### Pipeline templates and artifact flow

Use versioned, centrally maintained reusable workflows with stable input/output contracts:

1. **Pull request validation:** checkout, compile, unit/integration tests, SonarQube, dependency and secret checks, IaC validation and policy checks.
2. **Build once:** build the application and OCI image from the merge commit. Generate an SBOM, scan the image, sign the image digest, and push it to ACR. Publish the immutable digest and evidence as workflow outputs/artifacts.
3. **Promote, don't rebuild:** deploy the same `image@sha256:digest` to development, staging, then production. Environment-specific Kustomize overlays contain only configuration differences; they must not rebuild or retag different image bytes.
4. **Record each promotion:** store commit SHA, image digest, SBOM/signature references, workflow run, approvals, deployment result, and Azure Monitor links together.

ACR Tasks provide cloud-based image builds and support on-demand and trigger-based builds; use the ACR digest as the promotion identity rather than a mutable tag. [ACR Tasks overview](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview)

### Environments and account boundaries

Use separate Azure subscriptions for production and nonproduction, grouped under management groups with common policy assignments. Keep shared networking, central logging, and the platform ACR in a platform subscription. Use resource groups and AKS namespaces to isolate application teams; avoid broad subscription-wide contributor grants to application pipelines.

| Environment | Purpose | Promotion control |
| --- | --- | --- |
| Development | Fast integration and functional tests | Automatic after CI gates pass |
| Staging | Production-like validation and rollout rehearsal | Service-owner approval for high-risk changes |
| Production | Customer traffic | Protected GitHub environment, named approvers, successful staging evidence, and monitored rollout |

Use GitHub environment protection rules for deployment approvals and environment-scoped variables/secrets. Assign Azure RBAC at the narrowest applicable scope (resource or resource group, not subscription by default). [GitHub environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments), [Azure RBAC scope](https://learn.microsoft.com/en-us/azure/role-based-access-control/scope-overview)

### Azure identities and account roles

Prefer GitHub Actions OIDC federation to a Microsoft Entra application or user-assigned managed identity. Give each environment a distinct federated identity and role assignment; do not share production credentials with CI or nonproduction. The current MSI/service-principal composite action is a transition point, not the target identity pattern. [Authenticate GitHub Actions to Azure with OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)

| Role/group | Responsibility and scope |
| --- | --- |
| Platform engineering | Own shared reusable workflows, Azure Policy initiatives, network/monitoring foundations, and ACR; no routine application deployment approval |
| Application team | Maintain service code and manifests; deploy to its namespace/resource group using workload-specific identity |
| Security | Own policy baselines, vulnerability exceptions, signing trust, and periodic evidence review |
| Production approver/on-call | Approve production promotion and emergency changes; cannot approve their own change alone |
| Workload identity | CI may push to the designated ACR repository; deploy identity may update only its target namespace/resources; read-only monitoring access where practical |

### Team extension points

- Accept safe configuration through documented `workflow_call` inputs (Java version, test command, Dockerfile/context, Kustomize overlay, image metadata, and environment).
- Keep security gates mandatory in the centrally maintained workflow; allow teams to add tests or build steps but not silently disable required checks.
- Pin reusable workflow/action references to reviewed releases or commit SHAs; use a documented deprecation window and migration guide for breaking input changes.
- Keep workload manifests and service-specific thresholds in the app repo; keep global policy, identity conventions, and release contracts in the platform repo.
- Require a short-lived, reviewed exception with owner, reason, compensating control, and expiry for deviations.

## 2. Controls as code

Implement gates as pull-request checks and promotion checks. A failed mandatory gate blocks merge or deployment; exceptions are explicit, approved, and time limited.

For repository-level implementation details—including `CODEOWNERS`, branch rulesets, Actions permissions, reusable workflow governance, and GitHub Environments—see [Repository Controls as Code](./REPOSITORY_CONTROLS_AS_CODE.md).

| Control | Gate / evidence | Azure or platform-native implementation |
| --- | --- | --- |
| Tests and quality | Compile, unit and integration tests; SonarQube quality gate; retain logs and test reports | Existing Maven/Sonar steps are a starting point |
| IaC | Bicep lint/build, parameter validation, and ARM what-if; reject unreviewed destructive changes | Azure Resource Manager what-if previews expected resource changes without applying them. [Bicep what-if](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-what-if) |
| Cloud/Kubernetes policy | Validate resource locations, diagnostic settings, public exposure, encryption, required tags, and AKS pod/security requirements; move from audit to deny only after remediation | Azure Policy compliance and remediation; Azure Policy add-on for AKS. [Azure Policy](https://learn.microsoft.com/en-us/azure/governance/policy/overview), [Azure Policy for AKS](https://learn.microsoft.com/en-us/azure/aks/use-azure-policy) |
| Secrets | Block committed credentials; use Key Vault at runtime and OIDC for pipeline-to-Azure auth; never print secret values | GitHub secret scanning/push protection plus Azure Key Vault RBAC and Entra federation. [Key Vault RBAC](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide), [OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect) |
| Dependencies | Review dependency changes, generate an inventory, and fail on unapproved critical/high vulnerabilities or known exploited packages | Dependabot/dependency review in GitHub; Microsoft Defender for Cloud DevOps security for code, secrets, IaC, and dependency posture. [Defender for DevOps](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-devops-introduction) |
| Container and SBOM | Generate SPDX or CycloneDX SBOM per image digest; scan image; sign digest; verify signature and SBOM before promotion; block critical vulnerabilities unless a security-approved exception exists | Push to ACR; use Defender for Cloud container vulnerability assessment and retain SBOM/signature references alongside the image. [Container vulnerability management](https://learn.microsoft.com/en-us/azure/defender-for-cloud/agentless-vulnerability-assessment-azure) |
| Production release | Require staging evidence, production environment approval, and healthy deployment metrics | GitHub protected environment approval; Azure Monitor alert state and dashboard linked in the release record. [Azure Monitor alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview) |

For AKS, assign Azure Policy initiatives and enforce approved pod configuration at the cluster; for Azure resources, keep policy definitions/assignments in reviewed IaC. Begin new controls in audit mode, fix existing violations, then switch to deny/enforcement with a documented exception path.

## 3. Progressive delivery

Use Azure Monitor for platform signals, Azure Monitor managed Prometheus/Container Insights for AKS, and Application Insights/OpenTelemetry for request-level telemetry. Configure alert rules and action groups before enabling automatic rollout progression. [AKS monitoring](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable), [Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview), [Azure Monitor alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)

| Compute type | Strategy and progression | Rollback |
| --- | --- | --- |
| AKS | Blue/green: deploy the candidate Deployment/Service alongside the stable version, run readiness checks and internal smoke tests, then switch the Ingress/Service backend. Observe for 30 minutes before removing the old version. Use Azure Monitor managed Prometheus and Container Insights for SLO and resource signals. | Switch the backend/selector to the retained stable version, stop candidate rollout, and preserve logs/events for diagnosis. |
| Azure App Service | Blue/green with deployment slots. Deploy and warm the staging slot, run smoke tests, optionally route 5% → 25% → 50% → 100% of traffic to the slot, then swap to production after gates pass. | Set slot traffic back to production or swap the previous production slot back. Keep the prior version available through the observation period. [App Service slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots) |

Suggested default progression gates (tune per service SLO and traffic volume):

- Hold each traffic step for at least 10 minutes and at least 1,000 requests; for low-volume services, require synthetic probes instead of relying on a small request sample.
- Continue only when availability is at least 99.9%, HTTP 5xx is at most 1% and no more than 0.5 percentage points above the stable baseline, and p95 latency stays within the service SLO and 10% of baseline.
- For AKS, require no new crash loops/OOM kills and sustained CPU below 80% and memory below 85% of allocatable capacity.
- Roll back if a hard SLO is breached once, or another threshold is breached in two consecutive five-minute windows. Stop promotion and page the on-call owner; do not automatically retry a failing version.

Database changes use expand/contract: first add nullable/backward-compatible schema, deploy code compatible with old and new schemas, backfill with observable/restartable jobs, then remove old schema only in a later release after all consumers have migrated. Take and verify backups before destructive changes. App rollback must not depend on reversing an irreversible schema change.

## 4. Emergency change path

An expedited change is for an active incident or material security risk, not for schedule pressure.

1. Open or reference an incident/change record with severity, affected service, business impact, and named incident commander.
2. Obtain approval from the incident commander and service owner; security approves any control exception. The deployer cannot be the sole approver.
3. Activate Azure privileges just in time through Microsoft Entra PIM with MFA, justification/ticket number, least privilege, and a short activation window (target: 1 hour). Alert on any emergency/break-glass identity use. [Microsoft Entra PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
4. Keep minimum automated gates: compile and focused tests, secret scan, IaC what-if/policy check for infrastructure changes, image vulnerability scan and signature verification, smoke test, and production health monitoring. Only the incident commander plus security may approve a documented gate exception.
5. Deploy one immutable image digest to the smallest affected scope; watch the same health thresholds as a standard rollout. Capture commit, digest, test/scan outputs, approval identities, PIM activation, pipeline run, and before/after Azure Monitor signals.
6. Within one business day, complete the full test/policy suite, remove temporary role assignments and exceptions, reconcile manifests/IaC to the emergency state, rotate any exposed credentials, and record a blameless review and follow-up actions.

## 5. Platform and control metrics

Use deployment records from GitHub Actions and incident/availability signals from Azure Monitor/Application Insights. Join events by service, environment, commit SHA, and image digest; report service-level and portfolio-level views.

| Metric | Definition | Collection |
| --- | --- | --- |
| Deployment frequency | Successful production deployments per service per week | Production environment deployment events; count deploys, not commits |
| Lead time for change | Median and 75th percentile from commit merged to production deployment | Git commit/PR timestamps joined to successful deployment records |
| Change failure rate | Production deployments followed within 7 days by rollback, hotfix, or attributable incident ÷ all production deployments | Deployment records joined to incident/change records |
| Time to restore service | Median time from incident start to verified service recovery | Incident timestamps plus Azure Monitor alert/recovery and service-owner confirmation |
| Change rework rate | Production deployments requiring corrective deployment within 7 days ÷ all production deployments | Deployment and incident records |
| Policy compliance | Compliant assigned resources/AKS policy checks ÷ evaluated resources/checks | Azure Policy compliance data |
| Supply-chain coverage | Production image digests with SBOM, successful scan, and verified signature ÷ deployed image digests | ACR digest and workflow evidence |
| Identity hygiene | Production workflows using OIDC/managed identity; active exceptions past expiry | Workflow configuration, Entra sign-in/audit logs, exception register |

Safeguards: publish formulas and time windows; count all production deployments, including emergency changes; do not exclude failed deployments or relabel incidents to improve scores; segment by service and deployment risk; pair speed metrics with reliability/security outcomes; review source records periodically; use metrics for improvement, not individual ranking or quotas. Azure Monitor alerts can trigger response workflows, but an alert alone is not proof of an incident or recovery.

## 6. Incremental migration plan

| Phase | Adoption step | Exit criteria |
| --- | --- | --- |
| 0. Discover | Inventory repositories, pipeline variants, service owners, environments, credentials, IaC, deployment targets, and current delivery metrics | Named owner and baseline for every production service |
| 1. Pilot | Select one low-risk Java service; call reusable Maven/Sonar workflow; store images in ACR; add OIDC, Key Vault, and Azure Monitor; keep current production release as fallback | CI parity, reproducible image digest, documented rollback, no production outage |
| 2. Shadow controls | Run IaC and Azure Policy in audit mode; generate SBOM/scans and compare with existing checks; do not block teams on untriaged legacy findings | Findings assigned, false positives reviewed, owners and remediation dates agreed |
| 3. Opt-in promotion | Add staging and production environments, approvals, AKS/App Service progressive deployment, and immutable digest promotion for willing teams | A full low-risk release and rollback demonstrated with evidence |
| 4. Broad adoption | Onboard teams in waves; publish migration guides and office hours; enforce only controls with tested remediation and exception paths | All production services use versioned templates or have an approved dated exception |
| 5. Tighten and retire | Convert audited policies to deny, remove static credentials and legacy pipelines, expire exceptions, and review DORA/control metrics quarterly | No unowned exceptions; platform SLO and security gates meet targets |

Avoid a big-bang rewrite: keep current pipelines running while teams adopt templates, provide adapters for existing test/build steps, and migrate one service at a time. Do not enforce a new deny policy until impact analysis, remediation guidance, and an emergency exception procedure are ready.

## Provider-native evidence

The Azure service choices and control capabilities described above are based on Microsoft documentation:

- [GitHub Actions to Azure using OpenID Connect](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [Azure RBAC scope](https://learn.microsoft.com/en-us/azure/role-based-access-control/scope-overview) and [Key Vault RBAC](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)
- [Microsoft Entra Privileged Identity Management](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview), [Azure Policy for AKS](https://learn.microsoft.com/en-us/azure/aks/use-azure-policy), and [Bicep what-if](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-what-if)
- [Defender for Cloud DevOps security](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-devops-introduction) and [container vulnerability management](https://learn.microsoft.com/en-us/azure/defender-for-cloud/agentless-vulnerability-assessment-azure)
- [ACR Tasks](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview)
- [AKS monitoring](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable), [Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview), and [Azure Monitor alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
- [App Service deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)
- [GitHub deployment environments and protection rules](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
