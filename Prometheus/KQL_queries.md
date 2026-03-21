## 1) Are *all pods* running without errors?

**A. Pods by phase (should be 0 for Pending/Failed/Unknown)**

```promql
sum by (namespace, phase) (kube_pod_status_phase{phase=~"Pending|Failed|Unknown"})
```

**B. Non-Running pods (direct)**

```promql
count by (namespace) (kube_pod_status_phase{phase!="Running"} == 1)
```

**C. Pods with containers not ready**

```promql
sum by (namespace, pod) (
  max by (namespace, pod) (kube_pod_container_status_ready == 0)
)
```

> **Healthy =** no pods in Pending/Failed/Unknown and `kube_pod_container_status_ready == 1` for all containers.

***

## 2) Containers’ CPU/Memory are **not reaching resource limits**

### CPU headroom versus limit (per container)

**Instantaneous ratio of usage to limit** (alert if > 0.9 = 90%)

```promql
sum by (namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
)
/
sum by (namespace, pod, container) (
  kube_pod_container_resource_limits{resource="cpu", unit="core"}
)
```

To list **containers above 90%** of CPU limit now:

```promql
(
  sum by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
/
  sum by (namespace, pod, container) (kube_pod_container_resource_limits{resource="cpu", unit="core"})
) > 0.9
```

### Memory working set versus limit (per container)

```promql
sum by (namespace, pod, container) (
  container_memory_working_set_bytes{container!=""}
)
/
sum by (namespace, pod, container) (
  kube_pod_container_resource_limits{resource="memory", unit="byte"}
)
```

Containers **above 90% of memory limit**:

```promql
(
  sum by (namespace, pod, container) (container_memory_working_set_bytes{container!=""})
/
  sum by (namespace, pod, container) (kube_pod_container_resource_limits{resource="memory", unit="byte"})
) > 0.9
```

> Tip: If some containers have **no limits**, the denominator will be missing. Track them separately and enforce limits.

**Containers with NO CPU/Memory limits**:

```promql
# No CPU limit
count by (namespace, pod, container) (
  kube_pod_container_resource_limits{resource="cpu"} == 0
)
# No Memory limit
count by (namespace, pod, container) (
  kube_pod_container_resource_limits{resource="memory"} == 0
)
```

***

## 3) Node CPU and Memory **not more than 85%**

### Node CPU utilization %

(Uses node-exporter; adjust idle mode labels if needed)

```promql
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
* 100)
```

Nodes **over 85% CPU**:

```promql
(
  100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
) > 85
```

### Node Memory utilization %

(Exclude caches/buffers for a realistic view)

```promql
(
  1 - (
    (node_memory_MemAvailable_bytes)
    /
    (node_memory_MemTotal_bytes)
  )
) * 100
```

Nodes **over 85% memory**:

```promql
(
  1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
) * 100 > 85
```

> For AKS, this aligns well with what kubelet considers pressure; you can also track `kube_node_status_condition{condition="MemoryPressure",status="true"}`.

***

## 4) How many **pods restarted** in the last 24 hours?

**Total restarts by namespace (24h increase of restart counter):**

```promql
sum by (namespace) (
  increase(kube_pod_container_status_restarts_total[24h])
)
```

**Top N pods by restart count (last 24h):**

```promql
topk(10,
  sum by (namespace, pod) (increase(kube_pod_container_status_restarts_total[24h]))
)
```

***

## 5) How many **pods crashed** in the last 24 hours?

“Crash” commonly means the **container terminated unexpectedly**. Track termination reasons and exit codes:

**A. OOMKilled (last 24h)**

```promql
sum by (namespace, pod, container) (
  increase(kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}[24h])
)
```

**B. Error / CrashLoopBackOff indicators**

*   **CrashLoopBackOff** usually shows in `kube_pod_container_status_waiting_reason`:

```promql
sum by (namespace, pod, container) (
  max_over_time(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}[5m])
)
```

*   **Any non-zero exit codes** (requires `kube_pod_container_status_last_terminated_exitcode` from kube-state-metrics v2.12+; if not available, skip):

```promql
sum by (namespace, pod, container) (
  increase(kube_pod_container_status_terminated_reason{reason="Error"}[24h])
)
```

If you don’t have `*_terminated_*` series, you can approximate **crashes** by restarts + waiting reason:

```promql
sum by (namespace) (increase(kube_pod_container_status_restarts_total[24h]))
+
sum by (namespace) (
  max_over_time(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}[24h])
)
```

> In practice, **OOMKilled** + **CrashLoopBackOff** + **restart increases** gives good coverage.

***

## 6) Other **important AKS metrics** you should watch

### A. Pod scheduling pressure

*   **Unschedulable pods** (e.g., insufficient CPU/Memory/Pods):

```promql
sum by (namespace) (kube_pod_status_scheduled{condition="false"})
```

### B. Node conditions (readiness & resource pressure)

```promql
# NotReady nodes
sum by (node) (kube_node_status_condition{condition="Ready",status="false"})

# Memory pressure
sum by (node) (kube_node_status_condition{condition="MemoryPressure",status="true"})

# Disk pressure
sum by (node) (kube_node_status_condition{condition="DiskPressure",status="true"})
```

### C. Kubelet/Container Runtime health (basic)

```promql
up{job="kubelet"} == 0
```

### D. API server saturation (requests in-flight)

```promql
sum (apiserver_current_inflight_requests) by (request_kind)
```

### E. PersistentVolume/PVC states

```promql
# Pending PVCs (storage pressure)
sum by (namespace) (kube_persistentvolumeclaim_status_phase{phase="Pending"})

# PV capacity vs. used (requires kubelet/cAdvisor volume stats)
sum by (persistentvolumeclaim, namespace) (kubelet_volume_stats_used_bytes)
/
sum by (persistentvolumeclaim, namespace) (kubelet_volume_stats_capacity_bytes)
```

### F. Evictions & OOM signals

```promql
# Pods evicted by node memory pressure (counter)
sum by (namespace) (increase(kube_pod_container_status_terminated_reason{reason="Evicted"}[24h]))
```

### G. DNS latency (CoreDNS)

```promql
histogram_quantile(0.99, sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) by (le))
```

***

## 7) Ready-to-use **dashboard panels** (quick copy)

**Panel: Cluster pods not running**

```promql
sum (kube_pod_status_phase{phase!="Running"})
```

**Panel: Containers over 90% Memory Limit**

```promql
sum by (namespace, pod, container) (
  (
    container_memory_working_set_bytes{container!=""}
  ) /
  (
    kube_pod_container_resource_limits{resource="memory", unit="byte"}
  )
) > 0.9
```

**Panel: Node Memory %**

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

**Panel: Pod restarts last 24h**

```promql
sum (increase(kube_pod_container_status_restarts_total[24h]))
```

***

## 8) Suggested **alert rules** (example thresholds)

```yaml
# High Node CPU
- alert: NodeCPUHigh
  expr: (100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 85
  for: 10m
  labels: {severity: warning}
  annotations: {summary: "Node CPU > 85%"}

# High Node Memory
- alert: NodeMemoryHigh
  expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
  for: 10m
  labels: {severity: warning}
  annotations: {summary: "Node Memory > 85%"}

# Container near Memory Limit
- alert: ContainerNearMemLimit
  expr: (
    sum by (namespace, pod, container) (container_memory_working_set_bytes{container!=""})
    /
    sum by (namespace, pod, container) (kube_pod_container_resource_limits{resource="memory", unit="byte"})
  ) > 0.9
  for: 5m
  labels: {severity: warning}
  annotations: {summary: "Container memory usage > 90% of limit"}

# Container near CPU Limit
- alert: ContainerNearCPULimit
  expr: (
    sum by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
    /
    sum by (namespace, pod, container) (kube_pod_container_resource_limits{resource="cpu", unit="core"})
  ) > 0.9
  for: 5m
  labels: {severity: warning}
  annotations: {summary: "Container CPU usage > 90% of limit"}

# Pod Restarts Spike (24h)
- alert: PodRestartsHigh
  expr: sum(increase(kube_pod_container_status_restarts_total[24h])) > 0
  for: 0m
  labels: {severity: info}
  annotations: {summary: "Pods restarted in last 24h"}
```

***

## 9) Notes specific to AKS

*   **Working set vs. RSS:** For memory pressure early warnings, prefer `container_memory_working_set_bytes`.
*   **Multiple node pools:** Break down queries by node labels (e.g., `agentpool`, `nodepool`):
    ```promql
    (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
    by (agentpool)
    ```
*   **Kubelet secured endpoints:** Make sure your Prometheus scrape configs include AKS kubelets with proper TLS and bearer tokens (kube-prometheus-stack does this out-of-the-box).



