# Azure Enterprise DevOps Platform

Reusable GitHub Actions templates for Java CI, Azure authentication, Azure Container Registry (ACR), Azure Key Vault, and AKS deployment. This README describes both the current repository and a proposed target platform design for Azure. The design is a specification: several of its security and release gates are not implemented in the current workflows yet.

## Repository contents today

```text
.github/
  actions/
    scan/                 Prisma twistcli image scan and JSON report
    config/
      load-environment/    Read and validate application deployment YAML
    azure/
      acr/build/           ACR build composite action
      akv/secrets/         Fetch secrets from Azure Key Vault
      aks/deploy/          Kustomize deploy, image rollout, and rollback action
      aks/smoke-test/      Retry an HTTP smoke test; optionally roll back on failure
      aks/cleanup/         Clear Azure CLI state and remove per-run kubeconfig
      aks/validate-emergency/ Validate incident ID and production image digest
      login/               Azure CLI login using MSI or service principal
    setup/
      maven-install/       Install Java and Maven on an Ubuntu runner
  workflows/
    maven-ci.yml           Maven, SonarQube, ACR build, Prisma scan, and digest output
    aks-deploy.yml         Key Vault login, AKS credentials, and Kustomize deployment
```

Reusable workflows are called with GitHub Actions `workflow_call`. Application Dockerfiles and Kustomize bases/overlays are expected in the checked-out application repository, not this template repository.

### Current workflows and known gaps

**Maven CI** (`.github/workflows/maven-ci.yml`) checks out the requested repo/ref, installs Java 21 and Maven 3.9.16, runs `mvn clean install` and SonarQube, then builds and pushes the image to the development ACR. The Prisma composite action downloads `twistcli` from the supplied HTTPS URL with `curl`, pulls and scans that image, writes a JSON report, and uploads the report as a workflow artifact. The workflow returns the image digest and immutable reference to its caller. Required inputs include the source repo/ref, SonarQube details, ACR details, build Key Vault, `prisma_console_url`, and `twistcli_url`. Required GitHub secret: `sonar_token`. Prisma Console credentials are read from Key Vault secrets named `PRISMA_USERNAME` and `PRISMA_PASSWORD` by default. The scan runner also needs Docker CLI/daemon access and network connectivity to Prisma Console.

**AKS deployment** (`.github/workflows/aks-deploy.yml`) is called once per GitHub Environment. It checks out the application repository, then calls `.github/actions/config/load-environment/action.yml` to load and validate that environment's settings from `configuration/maven-config.yaml`. The config includes AKS and Key Vault names/subscriptions, namespace, MSI/Key Vault secret names, ACR login server, Kustomize overlay, workload names, and smoke-test URL. The reusable workflow delegates validation, Azure login, cluster access, Kustomize apply, immutable image update, smoke testing, rollback, and cleanup to composite actions; it contains no inline `run` steps. It requires only `repo_name`, `branch_name`, and `environment_name` for normal deployment; the config-file path has a default. The `selfhosted-ubuntu` runner needs Ruby with the standard YAML/Psych library, Azure CLI, kubectl, network access to AKS, and a system-assigned managed identity with **Key Vault Secrets User** access. Target identities need permission to get AKS credentials and Kubernetes RBAC rights in the namespace. Separately, ensure the AKS kubelet identity has **AcrPull** on its registry.

Configure all non-secret deployment values in the checked-out application repository's `configuration/maven-config.yaml`. The config action fails with a clear error if the requested environment or required values are missing. The file contains Key Vault secret *names* only; actual MSI client IDs and service-principal credentials remain in Key Vault. GitHub Environments continue to provide deployment protection and approval rules, not AKS configuration values.

The application caller at `maven-application/.github/workflows/ci-cd.yml` runs `push`/manual CI. Set repository variables `SONAR_HOST_URL`, `SONAR_PROJECT_KEY`, `ACR_NAME` (the development ACR), `ACR_IMAGE_NAME`, `ACR_SUBSCRIPTION_ID`, `BUILD_KEYVAULT_NAME`, `PRISMA_CONSOLE_URL`, and `PRISMA_TWISTCLI_URL`; optionally set `BUILD_KEYVAULT_SUBSCRIPTION` and `BUILD_AUTH_METHOD` (defaults to `msi`). Set `SONAR_TOKEN` as a GitHub secret. The build identity needs **AcrPush** on the development ACR. The scan identity needs **AcrPull** on that ACR and read access to the two Prisma Console Key Vault secrets.

The target promotion model uses separate development, UAT, and production ACRs: build and scan in the development ACR, deploy to development AKS, then—after developer validation—import the same image digest into the UAT ACR and deploy to UAT. After UAT validation and production approval, import that digest into the production ACR and deploy to production. Each destination ACR and its identity needs its own scoped configuration. The current caller scaffold does not yet implement these cross-ACR imports; it currently passes the development image reference to each deployment, so do not treat UAT/production promotion as complete until that is changed.

Create GitHub Environments named `dev`, `uat`, and `prod` in the application repository before running deployment workflows. These environments control deployment records and protection rules; AKS settings, ACR login servers, and smoke-test URLs come from `configuration/maven-config.yaml`. Smoke tests run on `selfhosted-ubuntu`, so that runner must be able to reach the application endpoints. In `prod`, configure required reviewers, prevent self-review, and restrict deployments to `main`. The normal production deployment pauses for this environment's approval after UAT validation.

The application repository's `maven-application/.github/workflows/emergency-deploy.yml` is a manual, production-only path. It requires an incident/change ID and an immutable image digest already present in the production ACR, then calls the same reusable AKS deployment workflow and is subject to the same `prod` approval. It does not rebuild or promote the image. It reads production AKS, ACR, and smoke-test settings from `configuration/maven-config.yaml`, runs the production smoke test after rollout, and attempts rollback if the test fails. Configure each environment under **Settings → Environments**; do not rely on an automatically created, unprotected environment.

The Maven application checkout currently contains empty `pom.xml`, Dockerfile, and Kustomize files, so the workflow is a scaffold and cannot complete until those files are populated. Update the environment-specific AKS, Key Vault, ACR, workload-name, and smoke-test entries in `configuration/maven-config.yaml` with the real non-secret configuration before running deployments.

The Azure login action retains direct MSI/service-principal inputs for existing callers. The AKS and build workflows use its Key Vault mode, which requires the runner's system-assigned identity for bootstrap. The `prod` gate only applies after required reviewers are configured in GitHub Environment settings. The workflow currently uses rollout status and rollback, not canary or blue/green delivery. Other controls in this README—such as IaC policy gates, SBOM generation/signing, and DORA collection—are design recommendations, not implemented workflow gates yet.

## 1. Platform design

### Pipeline templates and artifact flow

Use versioned, centrally maintained reusable workflows with stable input/output contracts:

1. **Pull request validation:** checkout, compile, unit/integration tests, SonarQube, and IaC validation/policy checks when infrastructure changes. GitHub Advanced Security secret scanning/push protection and Dependabot alerts are enabled on each application repository; they are not jobs in the current Maven CI workflow. A Dependency Review action can be added as a separate PR check if required.
2. **Build once in development ACR:** build the application and OCI image from the merge commit. Run Prisma `twistcli` image scanning and retain its JSON report. Generate an SBOM and sign the digest as those gates are implemented.
3. **Promote between registries, don't rebuild:** after validation in each environment, import the same image digest from development ACR to UAT ACR, then to production ACR. Deploy each destination's digest to its matching AKS environment; verify the digest after every import. Environment-specific Kustomize overlays contain configuration differences, not rebuilt image bytes.
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

Use Azure Managed Service Identity (MSI) from trusted self-hosted runners; OIDC and federated credentials are not used in this design. The runner's system-assigned identity bootstraps Key Vault access, then a scoped user-assigned managed identity can be selected for build or deployment access. Keep build, non-production, and production permissions separate, and assign each identity only the Azure roles and resource scopes it needs. The service-principal path remains available in the composite action for compatibility, but MSI is the standard path. [Azure managed identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)

| Role/group | Responsibility and scope |
| --- | --- |
| Platform engineering | Own shared reusable workflows, Azure Policy initiatives, network/monitoring foundations, and ACR; no routine application deployment approval |
| Application team | Maintain service code and manifests; deploy to its namespace/resource group using workload-specific identity |
| Security | Own policy baselines, vulnerability exceptions, signing trust, and periodic evidence review |
| Production approver/on-call | Approve production promotion and emergency changes; cannot approve their own change alone |
| Managed identities | Build MSI may push to the designated ACR; deployment MSI may update only its target namespace/resources; read-only monitoring access where practical |

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
| Secrets | Enable GHAS secret scanning and push protection in each application repository; use Key Vault and runner MSI for Azure access. These are repository-level controls, not Maven CI jobs. | GitHub Advanced Security repository settings plus Key Vault RBAC and managed identity assignments. [Key Vault RBAC](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide), [Azure managed identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) |
| Dependencies | Enable the dependency graph and Dependabot alerts/security updates in each application repository. These alerts are repository-level, not Maven CI jobs; use Dependency Review as a separate PR check only if required. | GitHub repository security settings and Dependabot; Microsoft Defender for Cloud DevOps security can add organization-level posture visibility. [Defender for DevOps](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-devops-introduction) |
| Container and SBOM | Scan image with Prisma Cloud Compute `twistcli` and retain JSON results; generate SPDX or CycloneDX SBOM; sign digest and verify evidence before promotion | Prisma scan runs against the development ACR image; retain the JSON artifact with the image digest. Defender for Cloud can provide additional registry posture visibility. |
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

## 4. Emergency change path

An expedited change is for an active incident or material security risk, not for schedule pressure.

1. Open or reference an incident/change record with severity, affected service, business impact, and named incident commander.
2. Obtain approval from the incident commander and service owner; security approves any control exception. The deployer cannot be the sole approver.
3. Activate Azure privileges just in time through Microsoft Entra PIM with MFA, justification/ticket number, least privilege, and a short activation window (target: 1 hour). Alert on any emergency/break-glass identity use. [Microsoft Entra PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
4. Keep minimum pipeline gates: compile and focused tests, IaC what-if/policy checks for infrastructure changes, image vulnerability scan/signature verification, smoke test, and production health monitoring. Keep application-repository GHAS secret scanning/push protection and Dependabot alerts enabled independently; they are not CI stages. Only the incident commander plus security may approve a documented gate exception.
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
| Identity hygiene | Production workflows using approved MSI; active exceptions past expiry | Runner/identity configuration, Entra audit logs, exception register |

Safeguards: publish formulas and time windows; count all production deployments, including emergency changes; do not exclude failed deployments or relabel incidents to improve scores; segment by service and deployment risk; pair speed metrics with reliability/security outcomes; review source records periodically; use metrics for improvement, not individual ranking or quotas. Azure Monitor alerts can trigger response workflows, but an alert alone is not proof of an incident or recovery.

## 6. Incremental migration plan

| Phase | Adoption step | Exit criteria |
| --- | --- | --- |
| 0. Discover | Inventory repositories, pipeline variants, service owners, environments, credentials, IaC, deployment targets, and current delivery metrics | Named owner and baseline for every production service |
| 1. Pilot | Select one low-risk Java service; call reusable Maven/Sonar workflow; store images in ACR; configure runner MSI, Key Vault, and Azure Monitor; keep current production release as fallback | CI parity, reproducible image digest, documented rollback, no production outage |
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
