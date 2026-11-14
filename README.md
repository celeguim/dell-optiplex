# Observability GitOps — Dell OptiPlex K3s

Add these paths to the existing repository (which already deploys nginx and Grafana).

## Prerequisites

- Argo CD working on K3s
- Existing Grafana in namespace `monitoring`, managed by Argo CD
- K3s `local-path` StorageClass available
- Enough free disk/memory for Prometheus
- Confirm the Helm chart version `92.7.0` is accessible and supported by your K3s version

## 1. Configure and push

Replace `REPLACE_GIT_REPO_URL` in `gitops/bootstrap/monitoring-application.yaml` with the exact same Git repository URL used in the nginx Application. Push everything to `main`.

## 2. Bootstrap the stack

```sh
sudo k3s kubectl apply -f gitops/bootstrap/monitoring-application.yaml
sudo k3s kubectl get application monitoring-stack -n argocd
sudo k3s kubectl get pods,pvc,svc -n monitoring
sudo k3s kubectl get prometheus -n monitoring
```

Note: The first sync installs CRDs, which may take several minutes to become established. If Argo CD reports missing CRDs, inspect its sync operation and retry after CRDs are ready. Do not delete the existing Grafana Application.

## 3. Validate Prometheus

Check the actual service name (it should be `monitoring-stack-kube-prom-prometheus` for this release):

```sh
sudo k3s kubectl get svc -n monitoring | grep prometheus
sudo k3s kubectl -n monitoring port-forward svc/monitoring-stack-kube-prom-prometheus 9090:9090
```

Open http://localhost:9090/targets from the machine where port-forward runs. The Prometheus UI should only be exposed via local port-forward; it has no public Ingress.

## 4. Integrate existing Grafana

MERGE `gitops/apps/monitoring/grafana-values-additions.yaml` into the existing `gitops/apps/grafana/values.yaml` (only the `datasources` section). Do not replace existing values. If `datasources` already exists, merge into its `datasources.yaml.datasources` array; avoid duplicate default datasources.

Commit and push. Wait for the pre-existing Grafana Argo CD Application to sync. In Grafana → Connections → Data sources, test `Prometheus`. The expected service is
`http://monitoring-stack-kube-prom-prometheus.monitoring.svc.cluster.local:9090`.

Useful starter dashboards can be imported in Grafana manually using dashboard IDs 1860 (Node Exporter Full) and 15757 (Kubernetes views; verify compatibility), or provisioned later as Git-managed dashboard JSON.

## 5. Basic PromQL checks

```promql
up
sum(rate(container_cpu_usage_seconds_total{container!="",image!=""}[5m])) by (namespace)
sum(container_memory_working_set_bytes{container!="",image!=""}) by (namespace)
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
kube_pod_status_phase{phase="Running"}
```

Metrics availability depends on the K3s version and enabled kubelet/cAdvisor collectors. To diagnose missing data, start with `up` and the Prometheus Targets view.

## Notes

- Grafana is **not** installed twice: `grafana.enabled: false`.
- Prometheus PVC uses local-path and is bound to the Dell node: no HA or remote durability.
- No Traefik Ingress or MetalLB IP is allocated for Prometheus; it stays ClusterIP.
- `alertmanager.enabled: false` keeps the first iteration light; add alerting after basic metric validation.
- Verify CRD upgrade steps before upgrading chart major versions.
