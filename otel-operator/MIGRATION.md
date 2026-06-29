# k3d-config OTel → Prometheus Migration

Phased build-up to implement the multi-cluster observability pattern in a
single-cluster representation:

- Edge clusters (future) emit OTLP
- Main cluster runs `kube-prometheus-stack` + Grafana with kubernetes-mixin
  dashboards
- A translation layer in between makes OTLP-arrived data compatible with
  mixin label/metric-name expectations

In k3d this collapses to: otel-operator daemonset + gateway push OTLP
in-cluster to the kps Prometheus, which serves the mixin dashboards.

Source references:

- voize.de — translation-layer recipe (rule disables, forked PrometheusRule,
  `cluster` label injection)
- opentelemetry.io "Prometheus and OpenTelemetry — Better Together" —
  collector topology + OTLP receiver path
- Prior PRDs: `20260529-104500_k3d-otel-prometheus-kps-translation` (full
  translation guide), `20260602-102316_k3d-otel-operator-kps-install-v2-verify`
  (current baseline + deferred ISC-20)

## Phase 0 — Baseline (achieved 2026-06-02)

- kps healthy: Prometheus v3.12.0 pod 2/2 Running, prometheus-operator
  reconciles, 10 `monitoring.coreos.com` CRDs Established
- otel-operator healthy: 4-pod daemonset + 1-pod gateway, both at v0.150.1
- Gap: gateway metrics pipeline exports `[debug]` only; Prometheus CR has
  `enableOTLPReceiver: false`; no `externalLabels.cluster`

## Phase 1 — Open the OTLP pipe

**Edits**

- `kp-stack-values.yaml`: set `prometheus.prometheusSpec.enableOTLPReceiver: true`
- `kp-stack-values.yaml`: add `prometheus.prometheusSpec.externalLabels.cluster: k3d-otel-operator`
- Gateway `OpenTelemetryCollector` CR: add `otlphttp` exporter targeting
  the in-cluster Prometheus Service, wire it into the metrics pipeline
- Re-render kps, re-mount, recreate cluster

**Verify**

- `--web.enable-otlp-receiver` arg present on the Prometheus pod
- One gateway-emitted metric is queryable in Prometheus with
  `cluster=k3d-otel-operator`

Closes deferred ISC-20 from `20260602-102316_k3d-otel-operator-kps-install-v2-verify`.

## Phase 2 — Gateway enrichment + attribute promotion

**Edits**

- Gateway CR: add `[memory_limiter, k8s_attributes, resourcedetection, batch]`
  to the metrics pipeline processors, ordered before `otlphttp`
- `otlphttp` exporter config: set `promote_resource_attributes` to the
  12-attr list (drop `k8s.pod.uid` and `service.instance.id` from the
  2026-05-29 draft — unbounded cardinality on pod restarts)

**Verify**

- Incoming series carry K8s attrs as labels (`k8s_namespace_name`,
  `k8s_pod_name`, `k8s_node_name`, `k8s_container_name`, workload-kind names)
- Dropped attrs appear on `target_info` only, not on series labels

## Phase 3 — Target Allocator + KSM under OTel

**Edits**

- Gateway CR: enable Target Allocator
- `kp-stack-values.yaml`: re-enable `kubeStateMetrics` and the bundled
  `kube-state-metrics` sub-chart
- Confirm kps Prometheus CR Selectors remain `{}` so the kps-shipped KSM
  ServiceMonitor is reachable for TA discovery

**Verify**

- `kube_*` series in Prometheus carry the gateway's enrichment attrs (proof
  TA-via-OTel routed them), not kps direct-scrape attrs
- No double-ingest: confirm the kps Prometheus is NOT also scraping KSM

## Phase 4 — Disable broken default rules

**Edits**

- `kp-stack-values.yaml` `defaultRules.rules`:
  - voize 7: `k8sContainerCpuUsageSecondsTotal`, `k8sContainerMemoryCache`,
    `k8sContainerMemoryRss`, `k8sContainerMemorySwap`, `k8sContainerResource`,
    `k8sContainerMemoryWorkingSetBytes`, `k8sPodOwner`
  - node-exporter family: `nodeExporter`, `node`, `nodeNetwork`
  - control-plane orphans: `kubeApiserverAvailability`, `kubeApiserverBurnrate`,
    `kubeApiserverHistogram`, `kubeApiserverSlos`, `kubeControllerManager`,
    `kubeScheduler`, `kubeProxy`, `kubernetesSystemKubeProxy`, `etcd`,
    `kubernetesSystemApiserver`, `kubernetesSystemControllerManager`,
    `kubernetesSystemScheduler`
  - kubelet (replaced by `kubeletstats` receiver): `kubelet`,
    `kubernetesSystemKubelet`
  - Alertmanager (disabled in values): `alertmanager`

**Verify**

- prometheus-operator reconciles without rule-eval errors
- `PrometheusRule` object count drops to expected new total

## Phase 5 — Ship forked translation PrometheusRule

**Edits**

- New file in `/workspaces/k3d-config/otel-operator/`: a `PrometheusRule`
  mirroring the kube-prometheus-stack 86.1.0 `k8s.rules.yaml` with
  `job="kubelet"` and `metrics_path="/metrics/cadvisor"` filters stripped,
  rule names kept identical so downstream consumers don't notice
- Mount via the cluster spec

**Verify**

- Rule outputs populate from OTLP-arrived data:
  `container_cpu_usage_seconds_total:sum_irate`,
  `container_memory_working_set_bytes`, `container_memory_rss`,
  `container_memory_cache`, `container_memory_swap`,
  `namespace_workload_pod:kube_pod_owner:relabel` (Deployment / DaemonSet /
  StatefulSet / Job variants)

## Phase 6 — Gateway `transform`-alias for legacy metric names

**Edits**

- Gateway CR: add `transform/kps-name-alias` processor with OTTL
  `set(name, ...)` statements for the 4 voize-set source metrics:
  - `container.cpu.time` → `container_cpu_usage_seconds_total`
  - `container.memory.working_set` → `container_memory_working_set_bytes`
  - `container.memory.rss` → `container_memory_rss`
  - `container.memory.cache` → `container_memory_cache`
- Place after `k8s_attributes` so any resource-attr-conditional aliases
  see populated attributes

**Verify**

- Aliased metric names queryable in Prometheus alongside their OTel-native
  originals (proves the processor is applied without dropping anything)

## Phase 7 — Collector self-health ServiceMonitor

**Edits**

- New `ServiceMonitor` selecting the gateway + daemonset `otelcol_*`
  metrics endpoints
- Mount via the cluster spec

**Verify**

- PromQL: `otelcol_exporter_send_failed_metric_points` and
  `otelcol_processor_dropped_metric_points` return results
- This is the `up{}` replacement for the OTLP-push pipeline
