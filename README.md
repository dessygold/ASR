# ASR
WORK REPO
I have a task to create o documentation for one of the applications as the codeowners, the name of the application is Kyverno, i was asked to reference another application that has been completed called "Trust Manager", i have downloaded all the files for the completed Trust manager for refence guide  and also downloaded all the files for the Kyverno application, see below the work item ticket that explains high level the story for this. Overwatch Documentation
Open
  Issue created 1 week ago by Tyler Greene
Issue
Overwatch Documentation
Summary
Create/update standardized documentation, bring cdso_config.yml up to date and conduct cross training on management of this application.

Definition of done
README updated
Containers identified
cDSO config file updated
docs/ files updated
upgrade process defined
associated terraform modules reviewed
additional work items created to address any gaps in documentation, configuration, optimization, etc.
Notes
This issue has been created as a generic catch-all work item for updating each of the DSOP applications to conform with the standardized repository structure. All team members in the CODEOWNERS file are responsible for updating the documentation, contributing to the project and managing the deployment of the application.

This ticket implies cross-training, research, integrating and shared ownership.

If this application requires substantial work, additional CODEOWNERS can be added.

Team members that pick up issues should make sure to add the start/end dates and track time spent. These metrics will help DSOP leadership revise sprint management, capacity planning, etc.

########################
Work item for nexus Grafana

Nexus IQ Grafana Dashboard
Open
  Issue created 1 month ago by Tyler Greene
Summary
Create or import a Grafana dashboard for this application

Definition of done
metrics are ingested into Victoria stack, verified in Grafana Explore
key metrics identified
dashboard created or imported
data source, metrics queries validated
Notes
Requires metrics endpoints to be exposed and Victoria scrape manifests applied


######################### Note from Gen AI
Nexus IQ is not a direct equivalent of Nexus Repository Manager for Prometheus scraping. Do not change the Nexus Repository VMServiceScrape only from 8081 to 8070/8071 and expect it to work: Nexus IQ exposes its operational metrics as JSON at http://localhost:8071/metrics, while VMServiceScrape expects Prometheus-format metrics. You will need a JSON-to-Prometheus exporter (such as json_exporter) or another approved exporter/translation method. 
sonatype.com
+1

What changes from the Nexus Repository scrape
Setting	Nexus Repository	Nexus IQ default	Recommendation
Application port	8081	8070	Do not use for operational metrics.
Admin/operational port	N/A	8071	This is where IQ exposes metrics, health, and admin endpoints.
Metrics path	/service/rest/metrics/prometheus	/metrics	IQ returns JSON, not Prometheus exposition format.
Authentication	Nexus metrics normally use basic auth with an account holding nx-metrics-all	IQ admin metrics endpoint is normally unauthenticated	Do not add basicAuth unless an intervening proxy explicitly requires it.
Default reachability	Service-accessible when exposed	8071 is normally bound to 127.0.0.1	A scrape from VMAgent will fail unless you use a sidecar or deliberately expose/admin-bind the port.
Sonatype documents Nexus IQ’s default application connector as 8070 and admin connector as 8071. Its /metrics endpoint is JSON and is specifically described as requiring an exporter for Prometheus consumption. The admin port is bound to localhost by default in recent IQ releases. 
sonatype.com
+1

Recommended pattern: json_exporter sidecar
The preferred Kubernetes pattern is:

Run a JSON exporter as a sidecar in the Nexus IQ pod.

Have that sidecar query http://localhost:8071/metrics.

Expose the sidecar’s Prometheus /probe endpoint through a Kubernetes Service.

Point VMServiceScrape to that Service port.

This preserves the default localhost-only protection on the Nexus IQ admin port rather than making port 8071 reachable across the cluster. Sonatype warns that the admin endpoint includes operational and administrative functions and should be tightly restricted. 
sonatype.com
+1

Candidate VMServiceScrape
Assumptions for this example:

Namespace remains nexus.

The Nexus IQ Service has a port named metrics that maps to a json_exporter sidecar on port 7979.

The exporter module is called nexusiq.

The Service labels shown below are examples; replace them with labels actually present on the Service, not merely the Deployment/Pod.

apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: nexusiq
  namespace: nexus
spec:
  endpoints:
    - port: metrics
      path: /probe
      interval: 30s
      scrapeTimeout: 10s
      params:
        module:
          - nexusiq
        target:
          - http://localhost:8071/metrics
      relabelConfigs:
        - replacement: "nexusiq"
          targetLabel: job
        - action: replace
          sourceLabels:
            - __meta_kubernetes_pod_name
          targetLabel: instance
  selector:
    matchLabels:
      app.kubernetes.io/name: nexus-iq-server
      app.kubernetes.io/instance: nexusiq
A JSON exporter conventionally exposes its conversion endpoint at /probe and commonly listens on 7979; the target JSON URL and module are passed as parameters. 
github.com
+1

Service requirement for the sidecar
Your IQ Service needs a dedicated named port pointing to the exporter container—not to IQ’s normal application port:

apiVersion: v1
kind: Service
metadata:
  name: nexusiq-metrics
  namespace: nexus
  labels:
    app.kubernetes.io/name: nexus-iq-server
    app.kubernetes.io/instance: nexusiq
spec:
  selector:
    app.kubernetes.io/name: nexus-iq-server
    app.kubernetes.io/instance: nexusiq
  ports:
    - name: metrics
      port: 7979
      targetPort: metrics
      protocol: TCP
And the sidecar container should declare:

ports:
  - name: metrics
    containerPort: 7979
    protocol: TCP
Using port: metrics in VMServiceScrape is preferable to hard-coding a numeric targetPort; it ties the scrape to the Service port intended for metrics. VMServiceScrape discovers Kubernetes Services through its label selector and supports selecting a specific service port. 
victoriametrics.com
+1

Items to remove from the existing Nexus YAML
For a Nexus IQ scrape via the default admin endpoint, remove these Repository-specific fields:

targetPort: 8081
path: /service/rest/metrics/prometheus
basicAuth:
  username:
    name: nexus-monitoring-creds
    key: username
  password:
    name: nexus-monitoring-creds
    key: password
They are appropriate to the Nexus Repository Manager Prometheus endpoint, but not to the default Nexus IQ JSON operational-metrics endpoint. 
sonatype.com
+1

Important validation items
Before applying the scrape:

Identify the actual IQ chart and release labels.
The official non-HA chart’s defaults use IQ application/admin ports 8070/8071, but your deployed Service labels and port names may differ. 
github.com

Capture a sanitized sample of the IQ JSON.
Run this from inside the IQ pod:

curl -s http://localhost:8071/metrics?pretty=true
The JSON structure determines the json_exporter module’s JSONPath mappings. Sonatype documents the optional pretty=true parameter for readable output. 
sonatype.com

Build the exporter module from actual returned keys.
Do not assume the metric names/JSON paths. They may vary by IQ version and enabled functionality.

Verify the rendered scrape target.
Confirm that VMAgent receives Prometheus text from:

curl -G \
  'http://<nexusiq-pod-ip>:7979/probe' \
  --data-urlencode 'module=nexusiq' \
  --data-urlencode 'target=http://localhost:8071/metrics'
Keep the IQ admin port private.
Avoid setting SERVER_ADMIN_CONNECTOR_BIND_HOST=0.0.0.0 unless there is a reviewed network policy/proxy design restricting access. Sonatype explicitly cautions against broadly exposing that port. 
sonatype.com
+1
