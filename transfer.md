# Kubescape Deployment Review & Technical Specification

## 1. Technical Review & Architectural Assessment

### 1.1 Executive Assessment
The current deployment documentation is well-structured, concise, and accurately reflects a mature DevSecOps deployment pattern within a hardened DoD / DSOP AWS GovCloud environment. It demonstrates clear adherence to air-gapped container patterns, automated ECR synchronization, and GitOps delivery.

---

### 1.2 Architectural Strengths

* **Air-Gapped & Zero-Internet Design:**
  * Setting `downloadArtifacts: false` and `skipUpdateCheck: true` guarantees that Kubescape does not attempt egress to public registries or GitHub endpoints.
  * Local bundling of `grypeOfflineDB` provides full vulnerability assessment capabilities in disconnected or strictly isolated VPCs.
* **GitOps-Safe Certificate Strategy:**
  * Selecting `certificates.strategy: initContainer` rather than the default `template` strategy resolves a common ArgoCD issue where templated certificates trigger perpetual drift and continuous out-of-sync reconciliations.
* **Decoupled ApplicationSet Orchestration:**
  * Utilizing a centralized `ApplicationSet` within `Shared ApplicationSets` standardizes deployment across multiple clusters (`core`, `enterprise`, `runners`, `commercial`) while keeping cluster enrollment declarative via `values-<env>.yaml`.
* **Automated Upstream Ingestion:**
  * The CI pipeline pattern in `dsop/pipelines` (`external-application.yaml`) establishes a clear supply-chain boundary, pulling from upstream, rendering discovery values, triggering cDSO container hardening pipelines, and hosting artifacts exclusively in internal ECR registries.

---

### 1.3 Identified Risks & Improvement Recommendations

| Category | Observed Pattern | Technical Risk | Recommended Action |
| :--- | :--- | :--- | :--- |
| **Blast Radius & Promotion** | ApplicationSet syncs from `main` with `automated.selfHeal: true`. | Changes merged to `main` deploy simultaneously to both **dev** and **test** environments. A broken Helm value or misconfiguration immediately impacts all clusters at once. | Implement target revision pinning or environment-specific branch/tag references in `values-<env>.yaml` (e.g., `dev` follows `main`, while `test` and `prod` track specific Git release tags). |
| **Immutability & Provenance** | `grype-offline-db` uses `v6-latest` with `pullPolicy: Always`. | Non-deterministic pod restarts. If two nodes restart hours apart, they may evaluate identical images against differing database definitions, complicating vulnerability triage and audit reproducibility. | Pin the vulnerability database to a dated, immutable tag or image digest (e.g., `v6-20241028` or SHA256 digest) updated via the automated CI schedule. |
| **Host Network Exposure** | `storage.hostNetwork: true` is enabled. | The `storage` pod binds directly to the node network stack, bypassing standard Kubernetes `NetworkPolicies` and exposing internal ports on the host. | Document the technical justification for `hostNetwork` (e.g., API aggregation routing) and ensure node-level security group rules restrict ingress to expected cluster CIDRs. |
| **Cluster Node Tolerations** | `customScheduling.tolerations: operator: Exists`. | The `node-agent` DaemonSet schedules across all nodes, including control plane, bootstrap, or dedicated workload nodes. | Ensure memory and CPU `requests` and `limits` are strictly enforced on `node-agent` to avoid starvation on sensitive or tainted nodes. |
| **Vulnerability Baseline Drift** | Scan findings are passive and do not enforce admission blocking. | Workloads with severe vulnerabilities or CIS benchmark failures will run without interruption unless an external controller or dashboard alerts on the `storage` CRDs. | Establish alerting thresholds or integrate Kubescape scan CRDs into centralized visualization (e.g., VictoriaMetrics / Grafana) to track posture degradation. |

---

## 2. Component Architecture Diagram

```
                             GitLab CI Pipeline (dsop/pipelines)
       +-----------------------------------------------------------------------------+
       | Upstream: kubescape.github.io/helm-charts                                   |
       |  ├─ Render image discovery matrix (root values.yaml)                        |
       |  ├─ Trigger cDSO Container Hardening Pipeline                               |
       |  └─ Import artifacts to Core-Test ECR (186542266927...)                      |
       |      ├─ charts/kubescape-operator:1.41.0                                    |
       |      └─ containers/kubescape-operator/<image>:<tag>                         |
       +--------------------------------------+--------------------------------------+
                                              |
                                              v
                              ArgoCD Shared ApplicationSet
                      (DSOP/Deployment/Gitops/Shared ApplicationSets)
       +--------------------------------------+--------------------------------------+
       | Target Clusters:                     | Config Sources:                              |
       |  - dev: core, enterprise, runners    |  1. ECR Chart: charts/kubescape-operator     |
       |  - test: core, enterprise, runners,  |  2. Git: helm-values/values.yaml             |
       |          commercial                  |  3. Git: helm-values/values-<env>.yaml       |
       +--------------------------------------+--------------------------------------+
                                              |
                                              v
+------------------------------------------------------------------------------------+
| EKS Cluster: Namespace 'kubescape'                                                 |
|                                                                                    |
|  +------------------------------------------------------------------------------+  |
|  | operator (Coordinator)                                                       |  |
|  | - Schedules configuration and compliance scans                               |  |
|  | - Directs kubescape and kubevuln jobs                                        |  |
|  +-------------------------+-----------------------------------+----------------+  |
|                            |                                   |                   |
|                            v                                   v                   |
|  +-----------------------------------+   +--------------------------------------+  |
|  | kubescape                         |   | kubevuln (Vulnerability Engine)      |  |
|  | - Framework & RBAC audits         |   | - Analyzes images against Grype DB   |  |
|  | - Evaluates security controls     |   +-------------------+------------------+  |
|  +-----------------+-----------------+                       |                     |
|                    |                                         | Pulls DB locally    |
|                    v                                         v                     |
|  +-----------------------------------+   +--------------------------------------+  |
|  | storage (Cluster API Server)      |   | grype-offline-db (Offline DB Pod)    |  |
|  | - hostNetwork: true               |   | - Bundled vulnerability definitions  |  |
|  | - Stores scan results as CRDs     |   | - Air-gapped / No external egress    |  |
|  +-----------------+-----------------+   +--------------------------------------+  |
|                    ^                                                               |
|                    | Runtime Events & Node Introspection                           |
|  +-----------------+------------------------------------------------------------+  |
|  | node-agent DaemonSet (Runs on all nodes via Toleration: operator: Exists)    |  |
|  | - Runtime anomaly detection & kernel event monitoring                        |  |
|  +------------------------------------------------------------------------------+  |
+------------------------------------------------------------------------------------+
```

---

## 3. Enhanced Deployment Specification

*The following technical specification is ready to be committed to the documentation repository.*

---

# Kubescape Deployment & Operations Guide

## 1. Overview
Kubescape is an open-source Kubernetes security posture management (KSPM) and vulnerability scanning platform. Within the DSOP platform, Kubescape performs automated risk analysis, configuration auditing against DoD and CIS baselines, image vulnerability scanning via Grype, and runtime event monitoring across EKS clusters.

Kubescape operates in **audit mode by default**; it reports vulnerabilities and configuration drift without actively terminating or blocking non-compliant pods.

### 1.1 Deployed Architecture Components
All components reside in the `kubescape` namespace, deployed via the `kubescape-operator` Helm chart:

* **`operator`:** Central controller responsible for lifecycle management, scheduling scan triggers, and orchestrating worker pods.
* **`kubescape`:** Core scanning engine that queries the Kubernetes API to evaluate cluster resources against compliance frameworks (e.g., CIS benchmarks, MITRE ATT&CK®, NSA-CISA).
* **`kubevuln`:** Container image vulnerability scanner utilizing an embedded Grype engine.
* **`storage`:** In-cluster API extension server that persists scan outputs as custom Kubernetes resources, allowing native querying via `kubectl`.
* **`node-agent`:** DaemonSet running on every cluster node to monitor system calls, process lifecycles, and runtime integrity.
* **`grype-offline-db`:** High-availability local copy of the vulnerability database, eliminating external network dependencies.

---

## 2. Supply Chain & Ingestion Pipeline

### 2.1 ECR import (`dsop/pipelines`)
The `.gitlab-ci.yml` pipeline triggers `external-application.yaml` on a defined schedule or via manual execution:

1. **Upstream Ingestion:** Pulls the upstream chart from `https://kubescape.github.io/helm-charts` with `APP: kubescape-operator`.
2. **Dynamic Discovery:** Renders the root `values.yaml` with `grypeOfflineDB.enabled: true` to discover the full image dependency tree.
3. **ECR Staging:** Pushes validated charts and images into the unified core-test ECR repository (`186542266927.dkr.ecr.us-gov-west-1.amazonaws.com`):
   * Chart: `charts/kubescape-operator`
   * Containers: `containers/kubescape-operator/<image>`
4. **Release Tagging:** Commits the migration version tag to the deployment repository and opens a GitLab issue if an upstream version update is detected.

---

## 3. GitOps Deployment via ApplicationSets

Kubescape instances are deployed using the `kubescape` ApplicationSet defined in `DSOP/Deployment/Gitops/Shared ApplicationSets` (`templates/appset-kubescape.yaml`).

### 3.1 ArgoCD Application Architecture
For every cluster enrolled, the ApplicationSet provisions a child Application named `<system>-kubescape` configured with:
* Automated sync, resource pruning (`prune: true`), and drift correction (`selfHeal: true`).
* **Multi-Source Hierarchy:**
  1. **Helm Chart:** `charts/kubescape-operator` sourced from internal ECR at the version set by `kubescape.targetRevision`.
  2. **Base Values:** `helm-values/values.yaml` from the `main` branch.
  3. **Environment Overrides:** `helm-values/values-<env>.yaml` from the `main` branch.

### 3.2 Air-Gapped Configuration Settings (`helm-values/values.yaml`)

```yaml
# Internal ECR image mirrors
image:
  registry: 186542266927.dkr.ecr.us-gov-west-1.amazonaws.com/containers/kubescape-operator

# Strict air-gapped controls
kubescape:
  downloadArtifacts: false
  skipUpdateCheck: true

# Offline vulnerability database configuration
grypeOfflineDB:
  enabled: true
  pullPolicy: Always

# GitOps-safe certificate provisioning
certificates:
  strategy: initContainer

# Network and scheduling architecture
storage:
  hostNetwork: true

customScheduling:
  tolerations:
    - operator: Exists
```

#### Rationale for Core Settings:
* **`downloadArtifacts: false` & `skipUpdateCheck: true`:** Prevents components from making outbound public requests to GitHub, ensuring compliance with isolated VPC security groups.
* **`certificates.strategy: initContainer`:** Forces pods to generate self-signed internal TLS certificates at initialization rather than using Helm templating, preventing persistent ArgoCD GitOps diff flags.
* **`storage.hostNetwork: true`:** Enables the internal storage API server to bind directly to the host network interface to serve scan results cluster-wide.
* **`customScheduling.tolerations: operator: Exists`:** Allows the `node-agent` DaemonSet to schedule on all nodes (including control plane and tainted worker nodes) for cluster-wide runtime coverage.

---

## 4. Environment Matrix & Inventory

Kubescape is currently active in **dev** and **test** environments only. Production rollout is deferred until baseline evaluations conclude.

| Environment | Target Clusters | Chart Revision | Configuration Files |
| :--- | :--- | :--- | :--- |
| **dev** | `core`, `enterprise`, `runners` | `1.41.0` | `values.yaml` + `values-dev.yaml` |
| **test** | `core`, `enterprise`, `runners`, `commercial` | `1.41.0` | `values.yaml` + `values-test.yaml` |
| **prod** | *Unenrolled* | N/A | `values.yaml` + `values-prod.yaml` |

> **Note:** To enroll a new cluster or environment, add the cluster designation to the `kubescape.systems` array within the appropriate `values-<env>.yaml` file in the `Shared ApplicationSets` repository.

---

## 5. Security & Operational Runbook

### 5.1 Verification Commands

```bash
# Check status of the Kubescape child application in ArgoCD
kubectl get application -n argocd -l app.kubernetes.io/name=kubescape

# Verify all pods in the kubescape namespace
kubectl get pods -n kubescape -o wide

# Check DaemonSet scheduling across nodes
kubectl get daemonset -n kubescape node-agent

# Inspect vulnerability database pod status
kubectl logs -n kubescape -l app=grype-offline-db

# Query scan results stored as Custom Resources
kubectl get workloadconfigurationscans -n kubescape
kubectl get vulnerabilitymanifests -n kubescape
```

### 5.2 Step-by-Step Upgrade Procedure

1. **Trigger Mirror Pipeline:** Run the GitLab CI pipeline manually to mirror the target chart and containers into the GovCloud ECR registry. Check `docs/CDSO_STATUS.md` for image scan results.
2. **Review Default Values:** Compare upstream default values with `default-helm-values.yaml` to identify deprecated keys or newly required flags.
3. **Update Image Tags:** Update the component tags in `helm-values/values.yaml` for:
   * `kubescape`
   * `operator`
   * `kubevuln`
   * `storage`
   * `node-agent`
   * `grype-offline-db`
4. **Update Chart Version:** In `DSOP/Deployment/Gitops/Shared ApplicationSets/values-<env>.yaml`, update `kubescape.targetRevision` to the new chart version.
5. **Controlled Rollout:** Apply and verify changes in the `dev` environment first. Validate that `node-agent` rolls out sequentially across nodes without evicting critical monitoring or application workloads.
6. **Promote to Test:** Following successful verification in `dev`, promote the configuration to `test`.






############################################################################
REFERENCED Guide .md

# Deployment

This document describes the deployment process for `Grafana-operator`.

## 1. Summary & Architecture Overview

### 1.1 Summary
This deployment provides an enterprise-grade, security-hardened deployment of Grafana on an Amazon EKS cluster within an AWS GovCloud (`us-gov-west-1`) environment (e.g., DoD / DSOP IL5 compliance baselines). 

The setup leverages the **Grafana Operator** (`grafana.integreatly.org/v1beta1`) via Helm and ArgoCD GitOps Deployment achieve:
- **Zero-trust runtime footprint** with distroless container images and strictly non-root UID execution (`10001`).
- **Air-gapped plugin provisioning** using an init container that extracts validated plugins from a secure S3 bucket (`dsop-artifacts`), avoiding public internet traffic.
- **Enterprise DoD PKI Integration** utilizing centralized certificates mounted from `cert-manager`'s `trust-manager`.
- **Identity & Access Management (IAM)** via Keycloak Generic OAuth with CAC / EAMS-A support and role-attribute claim mapping.
- **AWS Application Load Balancer (ALB)** integration with modern TLS policies (`ELBSecurityPolicy-TLS13-1-2-2021-06`) and cookie stickiness.

### 1.2. How we deploy it to DSOP
Grafana is imported via `dsop/pipelines` project within pipelines/external-application-grafana.yaml file (see [.gitlab-ci.yml](../.gitlab-ci.yml)), which pulls the chart & containers images from the upstream Helm repo (`https://grafana.github.io/helm-charts`, set via `SOURCE_HELM_REPO` key/value pair).
The imported Container images are then pulled from our Core-Test ECR (`186542266927.dkr.ecr.us-gov-west-1.amazonaws.com/containers/grafana-operator/grafana`) in [Grafana Operator Manifests](https://code.cdso.army.mil/dsop/apps/grafana-manifests.git) project within the [Charts/values.yaml](https://code.cdso.army.mil/dsop/apps/grafana-manifests/-/blob/main/chart/values.yaml?ref_type=heads) file, pinned to `13.1.3-distroless-slim`.

The architecture leverages the Grafana Operator orchestrated through a coordinated three-tier ArgoCD deployment with sync waves:(see [core/app-of-apps/templates](https://code.cdso.army.mil/dsop/deploy/gitops/core/-/tree/dev/app-of-apps/templates?ref_type=heads)). The imported helm chart from the `dsop/pipelines`, is set to used in the `core/app-of-apps/templates` repo within the `app-grafana-1-operator.yaml` file.

- Wave 1 (app-grafana-1-operator.yaml): grafana operator deployment and Custom Resource Definitions (CRDs) pulled directly from an internal ECR Helm registry.

- Wave 2 (app-grafana-2-instance.yaml): Underlying storage (pvc.yaml), and the core Grafana instance configs.

- Wave 3 (app-grafana-3-dashboards.yaml): Creation of Grafana datasources, Dashboards & Folders.

Grafana is only deployed to Core clusters, configured through the `DSOP/Deployment/Gitops/Core Applications` project and Requires no Terraform module and no cloud infrastructure of its own — it only needs an existing EKS cluster with CRD installed.
The environment-specific value files [values-dev.yaml](https://code.cdso.army.mil/dsop/apps/grafana-manifests/-/blob/main/helm-values/values-dev.yaml?ref_type=heads), [values-test.yaml](https://code.cdso.army.mil/dsop/apps/grafana-manifests/-/blob/main/helm-values/values-test.yaml?ref_type=heads), [values-prod.yaml](https://code.cdso.army.mil/dsop/apps/grafana-manifests/-/blob/main/helm-values/values-prod.yaml?ref_type=heads) overrides beyond what's applied in in [Grafana Operator Manifests](https://code.cdso.army.mil/dsop/apps/grafana-manifests.git) project baseline within the [Charts/values.yaml](https://code.cdso.army.mil/dsop/apps/grafana-manifests/-/blob/main/chart/values.yaml?ref_type=heads) file.


---

## 2. Component Architecture Diagram

```
                       +----------------------------------+
                       |   AWS ALB (Internal, HTTPS 443)  |
                       |   TLS 1.3/1.2 Security Policy    |
                       +-----------------+----------------+
                                         |
                                         v
       +-------------------------------------------------------------------+
       | Kubernetes Namespace: monitoring                                  |
       | Node Selector: workload=monitoring                                |
       | Toleration: dedicated=monitoring:NoSchedule                       |
       |                                                                   |
       |  +-------------------------------------------------------------+  |
       |  | Pod (Grafana CRD Managed via ArgoCD)                        |  |
       |  |                                                             |  |
       |  |  +-------------------------------------------------------+  |  |
       |  |  | Init Container: install-plugins                       |  |  |
       |  |  | Image: standard-tools:1.6.5                           |  |  |
       |  |  | Role: Fetches plugins from s3://dsop-artifacts        |  |  |
       |  |  +---------------------------+---------------------------+  |  |
       |  |                              |                              |  |
       |  |                              v Writes to                    |  |
       |  |                  +-----------------------+                  |  |
       |  |                  | PVC: grafana-data     |                  |  |
       |  |                  | (/var/lib/grafana)    |                  |  |
       |  |                  +-----------------------+                  |  |
       |  |                              ^ Reads from                   |  |
       |  |                              |                              |  |
       |  |  +---------------------------+---------------------------+  |  |
       |  |  | Core Container: grafana                               |  |  |
       |  |  | Image: grafana:13.1.3-distroless-slim                 |  |  |
       |  |  | SQLite3 DB on PVC; Recreate Strategy (1 Replica)      |  |  |
       |  |  | CA Trust Bundle: /etc/ssl/certs/dsop-certificates.crt |  |  |
       |  |  +---------------------------+---------------------------+  |  |
       |  +------------------------------|------------------------------+  |
       +---------------------------------|---------------------------------+
                                         | Authenticates via OIDC
                                         v
                   +--------------------------------------------+
                   | Keycloak (CAC / EAMS-A Realm: DSOP)        |
                   | https://keycloak.dso.army.mil              |
                   +--------------------------------------------+
```

---


## 3. Configuration Breakdown

### 3.1 Base Images & Container Hardening

| Parameter | Value / Mechanism | Purpose & Security Context |
| :--- | :--- | :--- |
| `grafanaImage` | `.../grafana:13.1.3-distroless-slim` | Uses a minimal distroless image containing zero shells, package managers, or common utility binaries, drastically minimizing CVE attack surfaces. |
| `pluginInstallerImage` | `.../standard-tools:1.6.5` | Standard utility container containing the AWS CLI and basic extraction tools (`unzip`, `tar`) for init tasks. |
| `securityContext` | `runAsUser: 10001`, `runAsNonRoot: true`, `capabilities: drop: ["ALL"]` | Enforces standard unprivileged runtime rules compliant with DoD Container Hardening guides. |
| `fsGroup` | `10001` | Ensures mounted volumes (`grafana-data`) inherit the required file ownership permissions for user `10001`. |

### 3.2 Air-Gapped Plugin Delivery Mechanism

Because GovCloud and IL5 environments restrict outbound internet egress to the Grafana Marketplace, plugins are pre-staged in S3 and pulled at container startup:

1. **Pre-Staging:** Plugins are maintained via a `.plugins` descriptor and staged into `s3://dsop-artifacts/grafana-plugins/`.
2. **Init Container Execution:**
   - Mounts the shared volume `grafana-data` onto `/var/lib/grafana`.
   - Iterates over `.Values.plugins` array within the [Charts/values.yaml](https://code.cdso.army.mil/dsop/apps/grafana-manifests/-/blob/main/chart/values.yaml?ref_type=heads) file.
   - Downloads each archive via `aws s3 cp s3://{{ .bucketPath }}/{{ $file }} /tmp/{{ $file }}`.
   - Unpacks `.zip`, `.tar.gz`, or `.tgz` bundles directly into `/var/lib/grafana/plugins`.
   - Deletes temporary archives to preserve disk space.
3. **Unsigned Plugin Configuration:**
   - The VictoriaMetrics logs data source (`victoriametrics-logs-datasource`) is explicitly configured in `allowLoadingUnsignedPlugins`.

#### Deployed Plugins

| Plugin Name | Version | Archive Type | Unsigned Allowed |
| :--- | :--- | :--- | :--- |
| `victoriametrics-metrics-datasource` | `v0.25.2` | Zip (default) | Signed (Pre-approved) |
| `victoriametrics-logs-datasource` | `v0.32.0` | Zip (default) | **Yes** (`allowLoadingUnsignedPlugins`) |
| `grafana-polystat-panel` | `2.1.16` | Zip (default) | Signed |
| `grafana-metricsdrilldown-app` | `2.5.1` | Zip (default) | Signed |
| `grafana-lokiexplore-app` | `2.5.2` | Zip (default) | Signed |

---

## 4. Authentication, Authorization & Security

### 4.1 Authentication with Keycloak & CAC / EAMS-A
Authentication to grafana UI is handled by Keycloak which require CAC auth via EAMS-A

### 4.2 PKI Trust Infrastructure
* **Trust Store Synchronization:** The pod mounts `/etc/ssl/certs/dsop-certificates.crt` from the `dsop-certs` ConfigMap.
* **Cert-Manager Integration:** Maintained automatically using cert-manager’s `trust-manager` (`trust.cert-manager.io` Bundle resource) to inject DoD and WCF root certificates.
* **Mutual TLS / Client Verification:** The OAuth client uses `tls_client_ca: "/etc/ssl/certs/dsop-certificates.crt"` to establish cryptographically verified communication with `keycloak.dso.army.mil`.

---

## 5. Storage

```yaml
deployment:
  spec:
    replicas: 1
    strategy:
      type: Recreate
```

* **Storage Engine:** Defaults to local file-backed **SQLite3** located at `/var/lib/grafana/grafana.db` on the `grafana-data` PersistentVolumeClaim.
* **Single-Replica Constraint:** Because SQLite3 is a single-writer file-based database, running multiple concurrent replicas against the same PVC leads to database locks and corruption.
* **Deployment Strategy:** Set to `Recreate` instead of `RollingUpdate`. The existing pod must release the PVC before the replacement pod mounts it.

---

## 6. Operational & Maintenance Procedures

### 6.1 Updating or Adding a Plugin
1. Add the plugin archive file (`.zip` or `.tar.gz`) into the S3 bucket:
   ```bash
   aws s3 cp <plugin-file> s3://dsop-artifacts/grafana-plugins/
   ```
2. Update the `.plugins` list in source control and update `values.yaml`:
   ```yaml
   plugins:
     - name: <plugin-name>
       version: "<version>"
       bucketPath: dsop-artifacts/grafana-plugins
   ```
3. If the plugin is unsigned, append its identifier to `allowLoadingUnsignedPlugins`:
   ```yaml
   allowLoadingUnsignedPlugins:
     - <plugin-name>
   ```
4. Commit and push the changes. ArgoCD will synchronize and trigger pod recreation.

