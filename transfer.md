# Deployment

This document describes the deployment process for `Kubescape`.

## 1. Summary & Architecture Overview

### 1.1 Summary

[`Kubescape`](https://kubescape.io/docs/) is a Kubernetes security posture and vulnerability scanner. It checks the cluster's configuration against security controls (for example privileged containers, risky RBAC roles or missing network policies) and scans the container images that are running for known vulnerabilities. It reports findings and does not block anything by default.

We install it with the `kubescape-operator` Helm chart. All components run in the `kubescape` namespace:

- **operator** — the coordinator. It triggers and schedules scans and talks to the other components.
- **kubescape** — the scanner that evaluates cluster resources against security controls.
- **kubevuln** — the image vulnerability scanner (uses [Grype](https://github.com/anchore/grype)).
- **grype-offline-db** — the vulnerability database that kubevuln uses, run inside the cluster (see [1.3](#13-what-is-grype-offline-db)).
- **storage** — an in-cluster API server that keeps scan results as Kubernetes resources.
- **node-agent** — a DaemonSet (one pod per node) that gathers node-level and runtime information.

- [Kubescape documentation](https://kubescape.io/docs/)
- [Kubescape operator Helm chart](https://github.com/kubescape/helm-charts/tree/main/charts/kubescape-operator)

### 1.2 How we deploy it to DSOP

Deployment has two parts: importing the chart and images into our registry, and deploying them to the clusters.

**Import to ECR (`dsop/pipelines`).** [.gitlab-ci.yml](../.gitlab-ci.yml) includes the `dsop/pipelines` `external-application.yaml` pipeline and sets `APP: kubescape-operator`, `SOURCE_HELM_REPO: https://kubescape.github.io/helm-charts` and `HELM_VALUES_FILE: values.yaml`. The pipeline runs only on a schedule or a manual run from the GitLab web console. It:

- Pulls the `kubescape-operator` chart from the upstream Helm repo and renders it with the root [values.yaml](../values.yaml) (which turns on `grypeOfflineDB`) to find every image the chart uses. That root file is only used for image discovery and is not deployed.
- Opens an "upgrade available" GitLab issue assigned to the [CODEOWNERS](../CODEOWNERS) when a newer chart exists.
- Runs the cDSO prebuilt-container pipeline for each image. Results are in [CDSO_STATUS.md](./CDSO_STATUS.md).
- Imports the chart as `charts/kubescape-operator` and the images as `containers/kubescape-operator/<image>` into the core-test ECR (`186542266927.dkr.ecr.us-gov-west-1.amazonaws.com`). That one registry serves every environment.
- Tags this repo with the imported chart version.

**Deploy to clusters (`Shared ApplicationSets`).** Kubescape is not deployed from an app-specific gitops repo. It is deployed by the `kubescape` ApplicationSet in the `DSOP/Deployment/Gitops/Shared ApplicationSets` project (`templates/appset-kubescape.yaml`). For each cluster listed for Kubescape, Argo CD creates one Application named `<system>-kubescape`, which installs into the `kubescape` namespace with automated sync, prune and self-heal. Each Application has these sources:

1. The `charts/kubescape-operator` chart from ECR, at the version in `kubescape.targetRevision` in the shared `values-<env>.yaml`.
2. [helm-values/values.yaml](../helm-values/values.yaml), then `helm-values/values-<env>.yaml`, from this repo's `main` branch.

The `support-manifests` source in the ApplicationSet is currently commented out, and the files in [support-manifests/](../support-manifests/) are empty placeholders. So **no** VictoriaMetrics scrape, Grafana dashboard or ExternalSecret is deployed for Kubescape. No Terraform module is needed.

### 1.3 What is `grype-offline-db`?

To find vulnerabilities, `kubevuln` compares the software inside each image against a database of known vulnerabilities (CVEs). By default, Grype downloads that database from the internet. DSOP clusters should not depend on internet access, so we run the database inside the cluster instead.

- **What it is:** a container image that contains a ready-made copy of the Grype vulnerability database. It is imported into our ECR like every other image (`containers/kubescape-operator/grype-offline-db`).
- **How it runs:** the chart runs it as its own small pod in the `kubescape` namespace (enabled with `grypeOfflineDB.enabled: true`). `kubevuln` reads the database from this pod and not from the internet.
- **How it stays current:** a small CronJob restarts the pod every day at 00:05 (the chart default `schedule: "5 0 * * *"`). Because `pullPolicy: Always` is set, the restarted pod pulls the image again, so it picks up a newer image if one is available in ECR.
- **What that means:** the database is only as fresh as the last image imported to ECR. If nobody imports a newer `grype-offline-db` image, scans keep using the old data, and newly published vulnerabilities will not show up.

*Simple example:* it is like antivirus definitions delivered on a USB stick instead of downloaded automatically. Scans work with no internet, but someone has to deliver a new stick to get new definitions.

## 2. Component Architecture Diagram

```
     GitLab pipeline (dsop/pipelines, external-application.yaml)
     upstream chart + images --> cDSO scan --> import to core-test ECR
                                                      |
                                                      v
                          Shared ApplicationSets (Argo CD)
                 one Application per cluster:  <system>-kubescape
          chart from ECR  +  helm-values/values.yaml + values-<env>.yaml
                                                      |
                                                      v
   +----------------------- Namespace: kubescape ------------------------+
   |                                                                     |
   |   operator ---- triggers/schedules scans ----+                      |
   |                                              |                      |
   |              kubescape  (configuration scans)|                      |
   |              kubevuln   (image scans)  <---- grype-offline-db       |
   |                  |                           (vulnerability data,   |
   |                  v                            restarted daily)      |
   |              storage  (keeps the results as Kubernetes resources)   |
   |                                                                     |
   |   node-agent  (DaemonSet: one pod on every node)                    |
   +---------------------------------------------------------------------+
```

The arrows are simplified to show the main flow.

## 3. Configuration Breakdown

Settings in [helm-values/values.yaml](../helm-values/values.yaml). Everything else, including the enabled capabilities, is the chart default (see [default-helm-values.yaml](../default-helm-values.yaml)). No cloud account or access key is configured.

| Parameter | Value | Purpose |
| :--- | :--- | :--- |
| `kubescape`, `operator`, `kubevuln`, `storage`, `nodeAgent` `.image.repository` | `186542266927.dkr.ecr.us-gov-west-1.amazonaws.com/containers/kubescape-operator/<image>` | Pulls images from our ECR instead of the public registry. Pinned at time of writing to `kubescape` `v4.0.17`, `operator` `v0.2.204`, `kubevuln` `v0.3.571`, `storage` `v0.0.378` and `node-agent` `v0.3.299`. |
| `kubescape.downloadArtifacts` / `skipUpdateCheck` | `false` / `true` | Stops Kubescape from downloading frameworks or checking for updates from the internet. |
| `grypeOfflineDB.enabled` | `true` | Runs the vulnerability database inside the cluster (see [1.3](#13-what-is-grype-offline-db)). |
| `grypeOfflineDB.image` | `.../grype-offline-db`, tag `v6-latest`, `pullPolicy: Always` | A moving tag that is pulled again every time the pod restarts. |
| `certificates.strategy` | `initContainer` | Pods generate their internal certificates at start-up. This is the chart's GitOps-safe option; the default (`template`) regenerates certificates on every Argo CD sync. |
| `storage.hostNetwork` | `true` | Runs the storage pod on the node's network. |
| `customScheduling.tolerations` | `operator: Exists` | Lets the pods, including the `node-agent` DaemonSet, run on every node, including tainted ones. |

## 4. Environments

- Kubescape is currently deployed to **dev** and **test** only. It is **not yet deployed to prod** (there is no `kubescape` block in the shared `values-prod.yaml`).

| Environment | Clusters | Chart version |
| :--- | :--- | :--- |
| dev | `core`, `enterprise`, `runners` (no `commercial` cluster in dev) | `1.41.0` |
| test | `core`, `enterprise`, `runners`, `commercial` | `1.41.0` |
| prod | not deployed | n/a |

- The chart version is set in the shared `values-dev.yaml` and `values-test.yaml`.
- [helm-values/values.yaml](../helm-values/values.yaml) is the shared baseline. [values-dev.yaml](../helm-values/values-dev.yaml), [values-test.yaml](../helm-values/values-test.yaml) and [values-prod.yaml](../helm-values/values-prod.yaml) are intentionally blank, so every environment currently runs the same configuration.
- To enroll an environment or system, add it to the `kubescape.systems` list in the shared `values-<env>.yaml`.

## 5. Risks (integrations with other systems)

- **Broad access.** The operator and node-agent read cluster resources and node activity, and they run on every node, so they need elevated permissions.
- **Changes to `main` apply right away.** The ApplicationSet reads this repo's `main` branch with auto-sync, so a merge to `main` is applied to every enrolled cluster, in dev and test together.
- **Vulnerability data can go stale.** `grype-offline-db` only updates when a newer image is imported to ECR (see [1.3](#13-what-is-grype-offline-db)). Because the tag is `v6-latest`, what a restart pulls can also change between restarts.
- **No monitoring or alerting yet.** Without the scrape and dashboard manifests deployed, findings are only visible inside each cluster, not in Grafana.
- **Image coverage.** [cdso_config.yml](../cdso_config.yml) lists four containers (`kubescape`, `operator`, `kubevuln`, `storage`), while [CDSO_STATUS.md](./CDSO_STATUS.md) shows scans for seven (it also lists `node-agent`, `grype-offline-db` and `kubescape-ops-tool`).

## 6. Verification

```bash
# Argo CD application for a cluster (replace <system> with core, enterprise, runners or commercial)
kubectl get application -n argocd <system>-kubescape

# All Kubescape pods, and the node-agent DaemonSet
kubectl get pods -n kubescape -o wide
kubectl get daemonset -n kubescape node-agent

# Scan results stored in the cluster
kubectl api-resources --api-group=spdx.softwarecomposition.kubescape.io
```

## 7. Upgrades

See [UPGRADE.md](./UPGRADE.md).





###############################################################################  UPGRADE BELOW


# Upgrade

This document describes how to upgrade `Kubescape`. It was moved here from [DEPLOYMENT.md](./DEPLOYMENT.md) and is a starting point for the Upgrade.md work item.

## Upgrade Process

1. Run the CI pipeline (manually from the GitLab web console, or wait for the schedule) to import the new chart and container versions to ECR. Check the cDSO results in [CDSO_STATUS.md](./CDSO_STATUS.md).
2. Overwrite `default-helm-values.yaml` with the new chart's default values. Check for breaking changes or newly required parameters, and update any deprecated parameters in use.
3. Update the image `tag` values for `kubescape`, `operator`, `kubevuln`, `storage`, `node-agent` and `grype-offline-db` in `helm-values/values.yaml` to the new versions.
4. In `DSOP/Deployment/Gitops/Shared ApplicationSets/values-<env>.yaml`, update the `kubescape.targetRevision` key to the new chart version. The chart version is not set in the ApplicationSet template.
5. Verify the rollout in `dev` before promoting to `test` (and to `prod` once Kubescape is deployed there). Check that the `node-agent` DaemonSet rolls out node by node without problems.
6. Review the [Kubescape release notes](https://github.com/kubescape/kubescape/releases) and the [operator chart releases](https://github.com/kubescape/helm-charts/releases) for breaking changes.

*Note: the node-agent DaemonSet restarts node by node during an upgrade, and scans may be delayed briefly while the other components restart.*

