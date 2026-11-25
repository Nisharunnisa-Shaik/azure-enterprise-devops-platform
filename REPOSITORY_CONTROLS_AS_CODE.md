# Repository Controls as Code

This guide describes repository-level controls for protecting source code and release workflows, limiting GitHub Actions permissions, and consuming centrally maintained reusable workflows. It focuses on GitHub controls for this platform repository and its application repositories; Azure RBAC and workload identity are included where they determine pipeline privilege.

## What is configured where

Some policy is stored in the repository; other policy is configured in GitHub settings or at the organization level. A checked-in workflow cannot reliably enforce a rule against someone who can change or remove that workflow.

| Control | Source of truth | What it does |
| --- | --- | --- |
| `CODEOWNERS` | Repository file | Identifies owners and requests their review when a matching file changes |
| Workflow permissions and checks | `.github/workflows/*.yml` | Limits each job's token and defines validation/deployment steps |
| Branch rulesets | Repository or organization settings; optionally managed with Terraform/API | Enforces pull requests, code-owner review, required checks, and restrictions on updates/bypass |
| GitHub Actions policy | Repository/organization Actions settings | Limits which actions can run and sets default `GITHUB_TOKEN` permissions |
| Environments | Repository settings plus workflow `environment:` declarations | Scopes deployment secrets/variables and gates deployments with approvals or wait timers |
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
      id-token: write # Only when this job uses Azure OIDC federation.
    environment: prod
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/deploy.sh
```

`id-token: write` allows requesting an OIDC token; it does not itself grant Azure access. Azure access is still controlled by the federated identity's trust conditions and Azure RBAC. Do not grant `contents: write`, `packages: write`, or other write scopes to the whole workflow if only one job needs them.

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

Store environment-specific configuration in GitHub Environment variables and secrets, not in workflow source. Put long-lived application secrets in Key Vault rather than workflow YAML. Only jobs that declare the corresponding `environment:` should receive that environment's protected secrets.

For Azure access:

- Prefer GitHub OIDC federation over long-lived service-principal secrets. Restrict each federated credential to the expected organization, repository, branch or GitHub Environment subject; use separate identities for build, non-production deployment, and production deployment.
- Assign Azure roles at the narrowest practical scope. A build identity should push only to the required ACR/repository; a deployment identity should access only its target AKS cluster/namespace and required resource group.
- Keep the AKS kubelet's `AcrPull` permission separate from the deployment identity's Kubernetes permissions.
- If a self-hosted runner's managed identity is used as a temporary bootstrap, isolate the runner, restrict its group to trusted workflows, and remove broad credentials from the host. Do not run untrusted pull-request jobs on it.
- Require an approved environment for production identity use; do not make production credentials available to CI or development jobs.

The current templates use managed identity/Key Vault bootstrap and should be treated as a transition arrangement until workload identity federation and its Azure RBAC assignments are implemented.

## 6. Require meaningful pull-request checks

Configure branch rules to require checks that run on every relevant pull request. A recommended progression is:

1. **Build and tests:** compile, unit/integration tests, and publish test reports.
2. **Code quality:** SonarQube quality gate and code scanning.
3. **Secrets and dependencies:** secret scanning/push protection, dependency review, and vulnerability alerts or updates.
4. **Infrastructure and manifests:** Bicep/Terraform validation, policy checks, Kubernetes/Kustomize validation, and detection of unsafe public exposure or privileged containers.
5. **Supply chain:** container vulnerability scan, SBOM generation, and image signing/attestation; verify these against the immutable digest before promotion.
6. **Deployment evidence:** smoke tests, rollout health, approval record, commit SHA, and image digest for each environment.

Start new checks in report-only mode where necessary to measure existing findings, then make them blocking after owners have remediation guidance. Exceptions should have an owner, reason, compensating control, approval, and expiry date; they should not be implemented as a silent permanent bypass.

Protect status-check identity: use stable workflow/job names, avoid reusing one check name for unrelated jobs, and select required checks from a successful run on the protected branch. Revisit required checks when workflow names change.

## 7. Repository security and maintenance

- Enable secret scanning and push protection where available; rotate any credential that is committed even if the commit is later reverted.
- Enable Dependabot alerts and security updates; use dependency review to flag risky additions in pull requests.
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
- [Authenticate GitHub Actions to Azure with OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [Azure RBAC scope](https://learn.microsoft.com/en-us/azure/role-based-access-control/scope-overview)
