# Deployment

This document describes the deployment process for `Kubescape`.

## Overview

[`Kubescape`](https://kubescape.io/docs/) is a Kubernetes security posture and vulnerability scanner. It checks the cluster's configuration against security frameworks (for example, privileged containers, risky RBAC roles or missing network policies), scans the container images that are running for known vulnerabilities, and can observe workload behavior at runtime. It reports findings and does not block anything by default.

We install it with the `kubescape-operator` Helm chart, which deploys these components into the `kubescape` namespace:

- **kubescape** — the scanner that evaluates cluster resources against security controls.
- **operator** — the coordinator. It triggers and schedules scans and talks to the other components.
- **kubevuln** — the image vulnerability scanner (uses Grype).
- **storage** — an API server that keeps scan results as Kubernetes resources inside the cluster.
- **node-agent** — a DaemonSet (one pod per node) that gathers node-level and runtime information.
- **grype-offline-db** — a copy of the vulnerability database, so scans do not need internet access.

- [Kubescape documentation](https://kubescape.io/docs/)
- [Kubescape operator Helm chart](https://github.com/kubescape/helm-charts/tree/main/charts/kubescape-operator)

## How we deploy it to DSOP

Deployment has two parts: getting the chart and images into our registry, and getting them onto the clusters.

### 1. Mirror the chart and images to ECR (`dsop/pipelines`)

[.gitlab-ci.yml](../.gitlab-ci.yml) includes the `dsop/pipelines` `external-application.yaml` pipeline and sets `APP: kubescape-operator`, `SOURCE_HELM_REPO: https://kubescape.github.io/helm-charts` and `HELM_VALUES_FILE: values.yaml`. The pipeline runs only on a schedule or a manual run from the GitLab web console. It:

- Pulls the latest `kubescape-operator` chart from the upstream Helm repo. It renders the chart with the root [values.yaml](../values.yaml) (which turns on `grypeOfflineDB`) so that every image the chart uses is found. That root file is only used for image discovery and is not deployed.
- Opens an "upgrade available" GitLab issue assigned to the [CODEOWNERS](../CODEOWNERS) when a newer chart exists.
- Runs the cDSO prebuilt-container pipeline for each image. Results are in [docs/CDSO_STATUS.md](./CDSO_STATUS.md).
- Copies the chart to `charts/kubescape-operator` and the images to `containers/kubescape-operator/<image>` in the core-test ECR (`186542266927.dkr.ecr.us-gov-west-1.amazonaws.com`). That one registry serves every environment.
- Tags this repo with the migrated chart version.

### 2. Deploy to clusters (`Shared ApplicationSets`)

Kubescape is not deployed from an app-specific gitops repo. It is deployed by the `kubescape` ApplicationSet in the `DSOP/Deployment/Gitops/Shared ApplicationSets` project (`templates/appset-kubescape.yaml`). For each cluster listed for Kubescape, Argo CD creates one Application named `<system>-kubescape`, which installs into the `kubescape` namespace with automated sync, prune and self-heal. Each Application has these sources:

- The `charts/kubescape-operator` chart from ECR, at the version in `kubescape.targetRevision` in the shared `values-<env>.yaml`.
- [helm-values/values.yaml](../helm-values/values.yaml), then `helm-values/values-<env>.yaml`, from this repo's `main` branch.

The `support-manifests` source in the ApplicationSet is currently commented out. The files in [support-manifests/](../support-manifests/) are empty placeholders, so **no** VictoriaMetrics scrape, Grafana dashboard or ExternalSecret is deployed for Kubescape.

Configuration in this repo ([helm-values/values.yaml](../helm-values/values.yaml)):

- Images are pointed at the ECR mirror, pinned at time of writing to: `kubescape` `v4.0.17`, `operator` `v0.2.204`, `kubevuln` `v0.3.571`, `storage` `v0.0.378`, `node-agent` `v0.3.299` and `grype-offline-db` `v6-latest`.
- `kubescape.downloadArtifacts: false` and `kubescape.skipUpdateCheck: true` stop Kubescape from fetching frameworks or checking for updates from the internet.
- `grypeOfflineDB.enabled: true` uses the bundled vulnerability database. `pullPolicy: Always` is set so the `v6-latest` image is refreshed when pods start.
- `certificates.strategy: initContainer` makes the pods generate their internal certificates at start-up. This is the chart's GitOps-safe option; the default (`template`) would regenerate certificates on every Argo CD sync.
- `storage.hostNetwork: true` runs the storage pod on the node's network.
- `customScheduling.tolerations: operator: Exists` lets the pods, including the `node-agent` DaemonSet, run on every node, including tainted ones.
- All other chart settings, including the enabled capabilities, are the chart defaults (see [default-helm-values.yaml](../default-helm-values.yaml)). No cloud account or access key is configured.
- No Terraform module is needed.

## Environments

- Kubescape is currently deployed to **dev** and **test** only. It is **not yet deployed to prod** (there is no `kubescape` block in the shared `values-prod.yaml`).
- **dev:** `core`, `enterprise` and `runners`. There is no `commercial` cluster in dev.
- **test:** `core`, `enterprise`, `runners` and `commercial`.
- Both dev and test use chart version `1.41.0`, set in the shared `values-dev.yaml` and `values-test.yaml`.
- [helm-values/values.yaml](../helm-values/values.yaml) is the shared baseline. [values-dev.yaml](../helm-values/values-dev.yaml), [values-test.yaml](../helm-values/values-test.yaml) and [values-prod.yaml](../helm-values/values-prod.yaml) are intentionally blank, so every environment currently runs the same configuration.
- To enroll an environment or system, add it to the `kubescape.systems` list in the shared `values-<env>.yaml`.

## Risks (integrations with other systems)

- **Broad access.** The operator and node-agent read cluster resources and node activity, and they run on every node, so they need elevated permissions.
- **Changes to `main` apply right away.** The ApplicationSet reads this repo's `main` branch with auto-sync, so a merge to `main` is applied to every enrolled cluster, in dev and test together.
- **Dependencies on other systems.** Image and chart pulls depend on the core-test ECR.
- **No monitoring or alerting yet.** Without the scrape and dashboard manifests deployed, findings are only visible inside each cluster, not in Grafana.
- **Moving image tag.** `grype-offline-db` uses `v6-latest` with `pullPolicy: Always`, so its contents can change between pod restarts.
- **Image coverage.** [cdso_config.yml](../cdso_config.yml) lists four containers (`kubescape`, `operator`, `kubevuln`, `storage`), while [docs/CDSO_STATUS.md](./CDSO_STATUS.md) shows scans for seven (it also lists `node-agent`, `grype-offline-db` and `kubescape-ops-tool`).

## Upgrade Process

1. Run the CI pipeline (manually from the GitLab web console, or wait for the schedule) to migrate the new chart and container versions to ECR. Check the cDSO results in `docs/CDSO_STATUS.md`.
2. Overwrite `default-helm-values.yaml` with the new chart's default values. Check for breaking changes or newly required parameters, and update any deprecated parameters in use.
3. Update the image `tag` values for `kubescape`, `operator`, `kubevuln`, `storage`, `node-agent` and `grype-offline-db` in `helm-values/values.yaml` to the new versions.
4. In `DSOP/Deployment/Gitops/Shared ApplicationSets/values-<env>.yaml`, update the `kubescape.targetRevision` key to the new chart version. The chart version is not set in the template.
5. Verify the rollout in `dev` before promoting to `test` (and to `prod` once Kubescape is deployed there).
6. Review the [Kubescape release notes](https://github.com/kubescape/kubescape/releases) and the [operator chart releases](https://github.com/kubescape/helm-charts/releases) for breaking changes.

*Note: the node-agent DaemonSet restarts node by node during an upgrade, and scans may be delayed briefly while the other components restart.*
