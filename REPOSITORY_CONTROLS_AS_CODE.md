# Repository Controls as Code

This guide describes repository-level controls for protecting source code and release workflows, limiting GitHub Actions permissions, and consuming centrally maintained reusable workflows. It focuses on GitHub controls for this platform repository and its application repositories; Azure RBAC and managed identities are included where they determine pipeline privilege.

## What is configured where

Some policy is stored in the repository; other policy is configured in GitHub settings or at the organization level. A checked-in workflow cannot reliably enforce a rule against someone who can change or remove that workflow.

| Control | Source of truth | What it does |
| --- | --- | --- |
| `CODEOWNERS` | Repository file | Identifies owners and requests their review when a matching file changes |
| Workflow permissions and checks | `.github/workflows/*.yml` | Limits each job's token and defines validation/deployment steps |
| Branch rulesets | Repository or organization settings; optionally managed with Terraform/API | Enforces pull requests, code-owner review, required checks, and restrictions on updates/bypass |
| GitHub Actions policy | Repository/organization Actions settings | Limits which actions can run and sets default `GITHUB_TOKEN` permissions |
| Environments | Repository settings plus workflow `environment:` declarations | Gates deployments with approvals or wait timers |
| Azure identity/RBAC | Microsoft Entra ID and Azure resource configuration | Limits the resources each build or deployment job can access |

Use repository settings as code where possible, for example by managing rulesets and Actions settings with the GitHub Terraform provider or an approved GitHub API automation. Keep that configuration in a separately protected infrastructure repository so a change to this repository cannot silently weaken its own protections.

## 1. Ownership and review

The root [`CODEOWNERS`](./CODEOWNERS) currently assigns the entire repository to `@Nisharunnisa-Shaik/devops`:

```text
* @Nisharunnisa-Shaik/devops
```

This means the DevOps team is requested as a code owner for every changed file. `CODEOWNERS` does **not** prevent a push, require approval, or protect itself. Enable a ruleset on the default branch that requires a pull request and at least one approval from a code owner. Keep the `CODEOWNERS` file itself covered by the catch-all rule.

If the desired policy is only DevOps may change workflow definitions while application teams can change application code, use more specific ownership patterns instead of the catch-all, for example:

```text
/.github/workflows/ @Nisharunnisa-Shaik/devops
/.github/actions/   @Nisharunnisa-Shaik/devops
```

Review the team mention against the actual organization/team slug in GitHub. Teams must be visible to the repository and have access to it for GitHub to apply them as code owners.

## 2. Protect the default branch

Create an active repository ruleset targeting the default branch (currently `main`, if that remains the default). A practical baseline is:

- Require changes to enter through a pull request; disallow direct updates to the protected branch.
- Require at least one approval and require approval from a code owner.
- Dismiss stale approvals when new commits are pushed; require approval of the most recent reviewable push.
- Require all review conversations to be resolved.
- Require the mandatory CI/security status checks before merge. Select the stable check names shown by actual workflow runs, and do not require optional deployment jobs that only run after merge.
- Require branches to be up to date with the base branch if the team can tolerate the additional queue/re-run cost.
- Block force pushes and branch deletion.
- Restrict bypass actors to a small, documented break-glass set. Prefer no routine administrator bypass; audit every emergency bypass.

For stronger separation of duties, require a second independent approval for production-impacting workflow or identity changes. Define that through a dedicated path ownership rule or a review policy that the team can consistently operate.

**Important:** requiring a code-owner approval lets other contributors propose changes, while requiring the DevOps team to be the only branch updater is a stricter policy. If only DevOps members may merge any change to the default branch, configure the ruleset's update/bypass restrictions accordingly. Avoid granting broad bypass just to make that restriction convenient.

## 3. Restrict GitHub Actions and `GITHUB_TOKEN`

In repository **Settings → Actions → General**:

1. Set the default workflow `GITHUB_TOKEN` permission to read-only.
2. Limit allowed actions and reusable workflows to approved GitHub-owned actions and reviewed internal/platform sources. Where practical, require actions to be pinned to a full commit SHA.
3. Decide whether workflows from forks require approval before running. Do not expose secrets or privileged self-hosted runners to untrusted pull-request code.
4. Review the allow-list and runner groups periodically; remove unused third-party actions and stale runners.

Set a read-only permission baseline in workflows, then grant only the extra permission to the job that needs it:

```yaml
permissions:
  contents: read

jobs:
  validate:
    permissions:
      contents: read
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<reviewed-40-character-commit-sha>
        with:
          persist-credentials: false
      - run: ./scripts/validate.sh

  deploy:
    permissions:
      contents: read
    environment: prod
    runs-on: selfhosted-ubuntu
    steps:
      - uses: actions/checkout@<reviewed-40-character-commit-sha>
        with:
          persist-credentials: false
      - name: Load deployment configuration
        id: deployment_config
        uses: Nisharunnisa-Shaik/azure-enterprise-devops-platform/.github/actions/config/load-environment@<reviewed-40-character-commit-sha>
        with:
          config_file: configuration/maven-config.yaml
          environment_name: prod
      - name: Authenticate to Azure using MSI
        uses: Nisharunnisa-Shaik/azure-enterprise-devops-platform/.github/actions/azure/login@<reviewed-40-character-commit-sha>
        with:
          keyvault_name: ${{ steps.deployment_config.outputs.keyvault_name }}
          keyvault_subscription: ${{ steps.deployment_config.outputs.keyvault_subscription }}
          azure_subscription: ${{ steps.deployment_config.outputs.aks_subscription }}
          auth_method: ${{ steps.deployment_config.outputs.auth_method }}
          msi_id_secret_name: ${{ steps.deployment_config.outputs.msi_id_secret_name }}
          client_id_secret_name: ${{ steps.deployment_config.outputs.client_id_secret_name }}
          client_secret_secret_name: ${{ steps.deployment_config.outputs.client_secret_secret_name }}
          tenant_id_secret_name: ${{ steps.deployment_config.outputs.tenant_id_secret_name }}
      - run: ./scripts/deploy.sh
```

The deploy job uses the managed identity exposed to the trusted self-hosted runner; it does not request `id-token: write`. GitHub token permissions and Azure MSI permissions are separate: the workflow token does not grant Azure access, and Azure access comes from the managed identity's Azure RBAC assignments. Do not grant `contents: write`, `packages: write`, or other write scopes to the whole workflow if only one job needs them.

For untrusted pull requests:

- Prefer the `pull_request` trigger for validation. Avoid `pull_request_target` for workflows that check out or execute contributor-controlled code.
- Do not pass deployment credentials to pull-request jobs.
- Do not run fork-controlled code on a persistent self-hosted runner that has access to Azure, Key Vault, or internal networks. Use isolated/ephemeral runners for privileged jobs.
- Keep artifact contents untrusted until validated; do not execute scripts from a downloaded artifact in a privileged job.

## 4. Use reusable workflows as the governed pipeline interface

Reusable workflows reduce drift, but calling one does not by itself enforce its use. A caller that can change its workflow may remove the call unless a ruleset or organization policy requires the approved workflow.

For each reusable workflow:

- Define an explicit `workflow_call` contract with typed, documented inputs, named secrets, and outputs. Avoid `secrets: inherit`; pass only the named secrets needed.
- Keep mandatory security gates in the centrally owned reusable workflow. Allow application-specific tests through documented inputs or extension points without allowing callers to silently skip required gates.
- Publish reviewed, immutable releases. Call a full commit SHA (or a protected, immutable release reference if organizational policy guarantees that immutability), not a mutable `@main` branch.
- Pin third-party actions inside the reusable workflow to full commit SHAs and review version updates.
- Give the called workflow no more token permissions than the caller. Declare minimal `permissions` in the caller and job-level permissions in the reusable workflow.
- Version breaking input/output changes; provide a migration period and record the commit/tag used by each application.

Example caller pattern:

```yaml
jobs:
  build:
    uses: Nisharunnisa-Shaik/azure-enterprise-devops-platform/.github/workflows/maven-ci.yml@<reviewed-40-character-commit-sha>
    permissions:
      contents: read
    with:
      repo_name: ${{ github.repository }}
      branch_name: ${{ github.sha }}
      sonar_host_url: ${{ vars.SONAR_HOST_URL }}
      sonar_project_key: ${{ vars.SONAR_PROJECT_KEY }}
    secrets:
      sonar_token: ${{ secrets.SONAR_TOKEN }}
```

The current application caller uses reusable workflows referenced at `@main`; replace those references with reviewed immutable commit SHAs before relying on them as a production control.

Where the GitHub plan and repository ownership model support it, use an organization ruleset to require a centrally maintained workflow. Otherwise, protect the caller workflow with code-owner review, require its checks, and monitor for changes that remove or weaken the reusable-workflow call.

## 5. Separate deployment environments and Azure identities

Declare GitHub Environments in deployment jobs and configure environment protection in repository settings:

| Environment | Recommended gate | Credential scope |
| --- | --- | --- |
| `dev` | Automatic after required CI checks | Development subscription/resource group and namespace only |
| `uat` | Automatic or service-owner approval, after development validation | Non-production UAT resources only |
| `prod` | Required reviewers, prevent self-review, and successful UAT evidence | Production resources only |

Use a distinct ACR for each environment. Grant the build MSI `AcrPush` only on the development ACR for image push and pull. For `az acr import`, assign the promotion MSI the built-in **Container Registry Data Importer and Data Reader** role on the source and destination ACRs. This is one role with a combined name, not separate importer and data-reader roles; `AcrPush` does not authorize the import operation. Scope each assignment to only the registries required for the next promotion. Each AKS environment should pull from its own registry.

Store non-secret per-environment deployment settings in the application repository's `configuration/maven-config.yaml` and load them with the centrally maintained config action. Keep GitHub Environments for deployment protection and approvals. Put credentials and application secrets in Key Vault; never place secret values in the config file or workflow YAML. Only jobs that declare the corresponding `environment:` receive its protection rules and environment secrets.

For Azure access using MSI:

- Use the system-assigned identity of a trusted self-hosted runner to bootstrap access to the relevant Key Vault. Grant it only the required secret-read permissions.
- Where more isolation is needed, use separate user-assigned MSIs for ACR publishing, non-production deployment, and production deployment. Attach only the required identities to the appropriate runner pool and select them explicitly.
- Assign Azure roles at the narrowest practical scope. A build identity should push only to the required ACR/repository; a deployment identity should access only its target AKS cluster/namespace and required resource group.
- Keep the AKS kubelet's `AcrPull` permission separate from the deployment identity's Kubernetes permissions.
- Restrict privileged runner groups to trusted workflows. Do not run untrusted pull-request code on a runner that can access MSI, Key Vault, ACR, AKS, or internal networks.
- For ARC runners on AKS, verify the supported MSI integration and prevent unrelated pods from reaching the node identity endpoint.
- Require an approved environment for production identity use; do not make production credentials available to CI or development jobs.

The current templates use MSI and Key Vault bootstrap. The service-principal option remains for compatibility, but MSI is the standard authentication method in this design.

## 6. Require meaningful pull-request checks

Configure branch rules to require checks that run on every relevant pull request. A recommended progression is:

1. **Build and tests:** compile, unit/integration tests, and publish test reports.
2. **Code quality:** SonarQube quality gate and code scanning.
3. **Repository security settings:** enable GHAS secret scanning/push protection and Dependabot alerts/security updates in each application repository. These operate at repository level and are not CI workflow jobs. Add Dependency Review as a separate pull-request check only if the team wants it as a required status check.
4. **Supply chain:** Prisma Cloud Compute `twistcli` image scan, SBOM generation, and image signing/attestation; retain the scan JSON and verify evidence against the immutable digest before promotion.
5. **Deployment evidence:** smoke tests, rollout health, approval record, commit SHA, and image digest for each environment.

Start new checks in report-only mode where necessary to measure existing findings, then make them blocking after owners have remediation guidance. Exceptions should have an owner, reason, compensating control, approval, and expiry date; they should not be implemented as a silent permanent bypass.

Protect status-check identity: use stable workflow/job names, avoid reusing one check name for unrelated jobs, and select required checks from a successful run on the protected branch. Revisit required checks when workflow names change.

## 7. Repository security and maintenance

- Enable GHAS secret scanning and push protection in each source application repository; these are GitHub repository controls, not CI steps. Rotate any credential that is exposed even if the commit is later reverted.
- Enable the dependency graph, Dependabot alerts, and security updates in each source application repository; alerts are repository-level. Add Dependency Review as a separate PR workflow only if it is intended to gate merges.
- Enable code scanning for supported languages and review alerts as owned work, not as an ignored dashboard.
- Keep workflow dependencies and runner images updated. Use a scheduled review/update workflow for pinned action SHAs.
- Limit repository administrators and Actions runner administration to the smallest practical group. Use organization teams rather than individual grants where possible.
- Keep audit logs, deployment histories, approvals, and workflow run evidence for the retention period required by the organization.
- Document break-glass access, require a ticket and time limit, alert on use, and review each event afterward.

## 8. Rollout checklist

1. Verify the default branch and the exact DevOps team slug; ensure the team has repository access.
2. Apply the CODEOWNERS policy and configure an active ruleset requiring pull requests and code-owner approval.
3. Restrict ruleset bypass, force pushes, and branch deletion; decide whether branch updates are limited to DevOps or only require DevOps approval.
4. Set Actions default token permissions to read-only and restrict allowed actions/workflows.
5. Update callers from mutable reusable-workflow references to reviewed immutable SHAs.
6. Add minimal workflow/job `permissions`, separate Azure identities by purpose/environment, and configure GitHub Environments with approvals.
7. Add required CI/security checks one at a time; confirm each check is stable before making it mandatory.
8. Test the rules with a non-DevOps contributor PR, a workflow change, an environment deployment, and a documented emergency path.
9. Review membership, bypass actors, identity scopes, and exceptions at least quarterly.

## References

- [GitHub CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [GitHub rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [GitHub Actions token permissions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication)
- [GitHub Actions security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [Reusable workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows)
- [Deployment environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
- [Azure managed identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- [Azure RBAC scope](https://learn.microsoft.com/en-us/azure/role-based-access-control/scope-overview)
