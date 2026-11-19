# Deployment

This document describes the deployment process for `Falco`.

## Overview

[`Falco`](https://falco.org/docs/) is a runtime security and threat-detection tool. A driver in the node's kernel watches system calls (file access, process starts, network activity) from every container on the node and compares them to a set of rules. When a rule matches, for example a shell opened inside a container, Falco raises an alert.

Falco detects and alerts only. It does not block or change anything.

The chart installs a single component:

- **falco** — a DaemonSet (one pod per node) that runs the Falco engine. An init container (`falco-driver-loader`) loads the kernel driver before the main container starts. The chart's `falcoctl` image (rules/plugin artifact tool) is also part of the chart, but its artifact install and follow features are disabled here.

- [Falco documentation](https://falco.org/docs/)
- [Falco Helm chart](https://github.com/falcosecurity/charts/tree/master/charts/falco)
- [Falco rules reference](https://falco.org/docs/concepts/rules/)

## How we deploy it to DSOP

Deployment has two parts: getting the chart and images into our registry, and getting them onto the clusters.

### 1. Mirror the chart and images to ECR (`dsop/pipelines`)

[.gitlab-ci.yml](../.gitlab-ci.yml) includes the `dsop/pipelines` `external-application.yaml` pipeline and sets `APP: falco` and `SOURCE_HELM_REPO: https://falcosecurity.github.io/charts`. The pipeline runs only on a schedule or a manual run from the GitLab web console. It:

- Pulls the latest `falco` chart from the upstream Helm repo and finds every container image the chart uses.
- Opens an "upgrade available" GitLab issue assigned to the [CODEOWNERS](../CODEOWNERS) when a newer chart exists.
- Runs the cDSO prebuilt-container pipeline for each image, using [cdso_config.yml](../cdso_config.yml). Results are in [docs/CDSO_STATUS.md](./CDSO_STATUS.md).
- Copies the chart to `charts/falco` and the images to `containers/falco/<image>` in the core-test ECR (`186542266927.dkr.ecr.us-gov-west-1.amazonaws.com`). That one registry serves every environment.
- Tags this repo with the migrated chart version.

### 2. Deploy to clusters (`Shared ApplicationSets`)

Falco is not deployed from an app-specific gitops repo. It is deployed by the `falco` ApplicationSet in the `DSOP/Deployment/Gitops/Shared ApplicationSets` project (`templates/appset-falco.yaml`). For each cluster listed for Falco, Argo CD creates one Application named `<system>-falco`, which installs into the `falco` namespace with automated sync, prune and self-heal. Each Application has these sources:

- The `charts/falco` chart from ECR, at the version in `falco.targetRevision` in the shared `values-<env>.yaml`.
- [helm-values/values.yaml](../helm-values/values.yaml), then `helm-values/values-<env>.yaml`, from this repo's `main` branch.
- [support-manifests/](../support-manifests/) from this repo's `main` branch: the VictoriaMetrics scrape config ([victoria-VMServiceScrape.yaml](../support-manifests/victoria-VMServiceScrape.yaml)).
- [dashboard/](../dashboard/) from this repo's `main` branch, on the `core` system only: the Grafana dashboard ([grafana-dashboard.yaml](../dashboard/grafana-dashboard.yaml)).

Configuration in this repo:

- Images are pointed at the ECR mirror in [helm-values/values.yaml](../helm-values/values.yaml): `falco` and `falco-driver-loader` at `0.45.0`, `falcoctl` at `0.14.2` at time of writing. All three are registered in [cdso_config.yml](../cdso_config.yml).
- The driver loader is enabled. The chart's default `driver.kind: auto` is not overridden.
- `tolerations: operator: Exists` lets the DaemonSet run on every node, including tainted ones.
- `falcoctl` artifact install and follow are disabled, so no rules are downloaded at runtime. The only rules are Falco's built-in set plus the custom rules below.
- Metrics are enabled (`metrics.enabled: true`, `falco.webserver.prometheus_metrics_enabled: true`) and scraped by VictoriaMetrics, then shown in the `falco-metrics` Grafana dashboard.
- Detections are viewed in the `falco-metrics` Grafana dashboard (rule matches counted from the `falco_rules_matches_total` metric). Falco also writes each alert to the pod logs (stdout). `falcosidekick` is not enabled, so alerts are not forwarded to any other system.
- No `ExternalSecret` or Terraform module is needed.

## Environments

- Falco is currently deployed to **dev** and **test** only. It is **not yet deployed to prod** (there is no `falco` block in the shared `values-prod.yaml`), and there is no date for prod yet.
- **dev:** `core`, `enterprise` and `runners`. There is no `commercial` cluster in dev.
- **test:** `core`, `enterprise`, `runners` and `commercial`.
- Both dev and test use chart version `9.2.0`, set in the shared `values-dev.yaml` and `values-test.yaml`.
- [helm-values/values.yaml](../helm-values/values.yaml) is the shared baseline. The six custom detection rules are in [values-dev.yaml](../helm-values/values-dev.yaml) and [values-test.yaml](../helm-values/values-test.yaml) (identical in both). [values-prod.yaml](../helm-values/values-prod.yaml) is intentionally blank, so prod would run without the custom rules until they are added.
- To enroll an environment or system, add it to the `falco.systems` list in the shared `values-<env>.yaml`. Adding Falco to prod also means filling in `helm-values/values-prod.yaml`.

## Risks (integrations with other systems)

- **Privileged node access.** Falco loads a kernel driver and reads system activity on every node, so the pods run with elevated privileges and tolerate all taints.
- **Alerts are only viewed in Grafana.** Detections show on the `falco-metrics` dashboard, which is deployed on the `core` system only. Nothing in this repo sends alerts to a SIEM, chat or ticketing system, so someone has to look at the dashboard.
- **Changes to `main` apply right away.** The ApplicationSet reads this repo's `main` branch with auto-sync, so a merge to `main` is applied to every enrolled cluster, in dev and test together.
- **Dependencies on other systems.** Image and chart pulls depend on the core-test ECR, and metrics depend on the VictoriaMetrics operator and, for the dashboard, the Grafana operator on `core`.
- **Open vulnerabilities.** [docs/CDSO_STATUS.md](./CDSO_STATUS.md) lists unmitigated vulnerabilities on the `falcoctl` image.

## Upgrade Process

1. Run the CI pipeline (manually from the GitLab web console, or wait for the schedule) to migrate the new chart and container versions to ECR. Check the cDSO results in `docs/CDSO_STATUS.md`.
2. Overwrite `default-helm-values.yaml` with the new chart's default values. Check for breaking changes or newly required parameters, and update any deprecated parameters in use.
3. Update the image `tag` values for `falco`, `falco-driver-loader` and `falcoctl` in `helm-values/values.yaml` to the new versions.
4. In `DSOP/Deployment/Gitops/Shared ApplicationSets/values-<env>.yaml`, update the `falco.targetRevision` key to the new chart version. The chart version is not set in the template.
5. Verify the rollout in `dev` before promoting to `test` (and to `prod` once Falco is deployed there).
6. Review the [Falco release notes](https://github.com/falcosecurity/falco/releases) for breaking changes to rules, drivers or configuration.

*Note: the DaemonSet restarts node by node during an upgrade. Detection on a node pauses briefly while its pod restarts.*
