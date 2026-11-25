# Azure Enterprise DevOps Platform

Reusable GitHub Actions for building Java container images, scanning them with Prisma Cloud Compute, promoting immutable ACR digests, and deploying to AKS with Kustomize. This README describes the files and setup in this repository; the broader target design and assessment are in [Azure DevSecOps Platform Assessment](./Azure_DevSecOps_Platform_Assessment.md).

## Repository contents

```text
.github/
  actions/
    azure/
      acr/build/             Build and push an image to ACR
      acr/promote/           Import and verify an image by digest
      aks/deploy/            Apply Kustomize and update the AKS workload
      aks/smoke-test/        Check the endpoint; optionally roll back
      aks/cleanup/           Clear Azure CLI state and per-run kubeconfig
      aks/validate-emergency/ Validate incident, emergency mode, and rollback digest
      login/                 Azure CLI login using MSI or service principal
    config/load-environment/ Read and validate application deployment YAML
    scan/                    Run Prisma twistcli and save JSON results
    setup/maven-install/     Install Java and Maven
  workflows/
    maven-ci.yml             Maven tests, SonarQube, ACR build, and Prisma scan
    aks-deploy.yml           Config-driven AKS deployment and optional ACR promotion
```

The application caller workflows and configuration are in the sibling [`maven-application`](../maven-application/) folder. Workflow templates call composite actions; application workflows do not contain inline deployment commands.

## Workflow behavior

### Maven CI

[`maven-ci.yml`](.github/workflows/maven-ci.yml) checks out the requested application ref, runs Maven tests and SonarQube, builds the image in the Dev ACR, scans it with Prisma `twistcli`, and returns an immutable image reference and digest. The scan JSON is uploaded as a workflow artifact. A failed build, quality gate, or scan blocks downstream deployment.

The application caller ([`ci-cd.yml`](../maven-application/.github/workflows/ci-cd.yml)) uses repository variables `SONAR_HOST_URL`, `SONAR_PROJECT_KEY`, `ACR_NAME`, `ACR_IMAGE_NAME`, `ACR_SUBSCRIPTION_ID`, `BUILD_KEYVAULT_NAME`, `PRISMA_CONSOLE_URL`, and `PRISMA_TWISTCLI_URL`. `BUILD_KEYVAULT_SUBSCRIPTION` and `BUILD_AUTH_METHOD` are optional; auth defaults to MSI. Set `SONAR_TOKEN` as a GitHub secret. The build identity needs push access to the Dev ACR; Prisma credentials are read from Key Vault.

### AKS deployment

[`aks-deploy.yml`](.github/workflows/aks-deploy.yml) is called with an application ref and GitHub Environment. It loads non-secret settings from `configuration/maven-config.yaml`, logs in using the configured identity, optionally imports an image into the target ACR by digest, applies the selected Kustomize overlay, updates the workload, and runs an optional smoke test with rollback on failure.

Fill in the Dev, UAT, and Prod ACR names, resource groups, subscriptions, and login servers in the application config. Also provide each environment's AKS, Key Vault, namespace, workload/container names, Kustomize path, and smoke-test URL. Keep secret values in Key Vault.

## Release flows

- **Standard release:** CI builds and scans once in Dev; deploy and validate in Dev; import the same digest to UAT and validate; wait for `prod` approval; then import the UAT digest to Prod and deploy.
- **Emergency rollback:** dispatch [`emergency-deploy.yml`](../maven-application/.github/workflows/emergency-deploy.yml) from `main`, provide an incident ID, select `rollback`, and supply a known-good digest already in the Prod ACR. The protected Prod Environment approval and production smoke test still apply.
- **Emergency fix:** dispatch from `main` with an incident ID, select `build_fix`, and leave the digest input empty. Maven CI tests, builds, and scans the fix; the workflow deploys it to Dev and smoke-tests it, then the protected Prod approval gates importing that digest from Dev to Prod and deploying it. This expedited path skips UAT; the normal release still requires UAT validation. There is no emergency gate-bypass input.

Configure `dev`, `uat`, and `prod` Environments in the application repository before running deployment workflows. For `prod`, configure required reviewers, prevent self-review, restrict deployment refs to the approved branches/tags, and disable administrator bypass when a no-bypass policy is required.

## Azure access needed

- The runner's system-assigned identity bootstraps Key Vault access.
- The build MSI needs image-push access to Dev ACR.
- The UAT MSI needs ACR import/read permissions on Dev and UAT.
- The Prod MSI needs ACR import/read permissions on UAT and Prod for standard releases, and Dev and Prod for emergency fixes.
- Deployment identities receive only their target AKS permissions. Each AKS kubelet identity needs pull access to its environment's ACR.

For human repository permissions, environment evidence, permission rationale, and the complete platform design, see the [assessment](./Azure_DevSecOps_Platform_Assessment.md) and [Repository Controls as Code](./REPOSITORY_CONTROLS_AS_CODE.md).

## Prerequisites and current limits

The trusted `selfhosted-ubuntu` runner needs Azure CLI, `kubectl`, Ruby with the standard YAML/Psych library, Docker access for Prisma scanning, and network access to AKS, ACR, Key Vault, and Prisma Console. The application checkout currently contains scaffold/empty Maven, Dockerfile, and Kustomize files, and its environment-specific ACR/workload/URL values are placeholders; complete them before a real run.

Prisma container scanning is implemented. An IaC scanner is not available, and SBOM generation and image signing are not implemented; do not treat these as passing workflow gates. No live Azure deployment has been validated from this workspace.
