# Autoscaling — cloud instance groups and the Kubernetes HPA

Changing capacity automatically in response to load: what the cloud providers offer, how the Kubernetes Horizontal Pod Autoscaler decides, and where both go wrong. Docs-based; HPA details are for the `autoscaling/v2` API (stable since Kubernetes 1.23). Related: [[scalability]] (what makes a service safe to scale out), [[sharding]] (scaling data rather than compute).

## What autoscaling can and cannot do

- It adjusts the **number** (horizontal) or **size** (vertical) of instances to follow demand, so capacity is not provisioned for the peak all day.
- It only works for a tier that is safe to add to and remove from at any moment: stateless, fast to start, and drained gracefully on shutdown.
- It is **reactive and slow relative to a spike**: metric collection interval + evaluation + instance boot + application warm-up is typically minutes. Autoscaling handles a ramp; a step change is absorbed by headroom, queues, caches and load shedding ([[reliability-patterns]]).
- It does not fix a bottleneck behind the scaled tier. Doubling application instances against a fixed database multiplies connections and can turn a slowdown into an outage.

## Cloud autoscaling

Every provider has the same construct under a different name: AWS Auto Scaling groups, Azure Virtual Machine Scale Sets, Google Cloud managed instance groups. The parts:

- **A template** for new instances (image, size, startup script) and **min / max / desired** counts. Min is what survives a cold start and the loss of one zone; max is the cost and blast-radius cap.
- **Health checks** that replace failed instances — self-healing is the same mechanism as scaling, holding the count at desired.
- **Scaling policies**:

| Policy | How it decides | Use for |
|---|---|---|
| **Target tracking** | Keep a metric at a target (CPU 50%, requests per instance 1,000); works like a thermostat | The default; one number to tune |
| **Step** | Alarm thresholds mapped to fixed adjustments (+2 above 70%, +5 above 90%) | Non-linear responses |
| **Scheduled** | Set capacity by clock | Known patterns: business hours, a campaign launch |
| **Predictive** | Forecast from past daily and weekly cycles and scale ahead | Regular cycles with slow-booting instances |

- **Warm-up and cooldown**: a new instance's metrics are ignored until it is warm, and further actions wait for the last one to take effect. Without them the group overshoots, then flaps.
- **Scale-in is the dangerous direction**: the provider picks an instance and terminates it. Connection draining at the load balancer and a termination hook that lets in-flight work finish make it invisible; without them every scale-in drops requests.

### Choosing the metric

- Scale on something that **leads** saturation and that **falls when capacity is added**. CPU fits CPU-bound services. For I/O-bound ones use in-flight requests per instance or queue depth per worker.
- Do not scale on latency alone: it rises late, falls late, and rises for reasons more instances do not cure (a slow dependency), so the group scales out and helps nothing.
- For queue workers, scale on **backlog per worker** (or the age of the oldest message), not on CPU — an idle-looking worker pool with a growing queue is the case CPU misses.

### Scale to zero

Functions and container platforms (AWS Lambda, Cloud Run, Azure Container Apps, Knative, KEDA on Kubernetes) scale on requests or events and can drop to no instances. The price is the **cold start** on the first request, and per-instance concurrency limits that turn a burst into many cold starts at once.

## Kubernetes: three autoscalers

| Autoscaler | Changes | Reacts to |
|---|---|---|
| **Horizontal Pod Autoscaler (HPA)** | Replica count of a Deployment, StatefulSet or other scalable workload | Metrics |
| **Vertical Pod Autoscaler (VPA)** | CPU and memory *requests* of pods | Observed usage |
| **Cluster Autoscaler / Karpenter** | Number of nodes | Pods that cannot be scheduled |

They chain: the HPA adds pods; if no node has room the pods stay `Pending`; the cluster autoscaler sees pending pods and adds a node. A pod-level scale-out can therefore wait for a node to boot — minutes, not seconds — unless spare node capacity is kept.

## Horizontal Pod Autoscaler

### How it decides

A control loop (every 15 seconds by default) computes, per metric:

```
desiredReplicas = ceil( currentReplicas × currentMetricValue / targetMetricValue )
```

Four pods averaging 90% CPU against a 60% target → `ceil(4 × 90 / 60)` = 6 pods.

- **Tolerance**: if the ratio is within 10% of 1.0 nothing happens, so small wobbles do not cause churn.
- **Several metrics**: each produces a desired count and the **largest wins**.
- **Pods that are not ready or have no metrics yet** are counted conservatively — they dampen a scale-up and never trigger a scale-down — so a slow-starting batch of new pods does not cause a second, unnecessary scale-up.

### A manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

### Metric types

| Type | Source | Example |
|---|---|---|
| `Resource` | metrics-server (CPU, memory per pod) | Average CPU utilization 60% |
| `ContainerResource` | Same, for one named container | Ignore a sidecar's CPU |
| `Pods` | Custom metrics API (for example a Prometheus adapter) | Requests per second per pod |
| `Object` | Custom metrics API, one Kubernetes object | Requests per second on an Ingress |
| `External` | External metrics API | Queue length in a cloud message service |

### Traps

- **Utilization is a percentage of the pod's *request*, not its limit.** A container with no CPU request has no utilization and the HPA reports the metric as unknown. A request set far below real usage reads as permanently over target.
- **metrics-server must be installed** for `Resource` metrics; custom and external metrics each need an adapter. A missing source fails quietly: the HPA exists and never acts (`kubectl describe hpa` shows why).
- **Do not set `replicas` in a Deployment the HPA owns.** Every `kubectl apply` resets the count to the manifest's value and the HPA then fights back.
- **Scale-down is deliberately slow**: a 300-second stabilization window by default — the HPA acts on the highest recommendation of the last five minutes. Scale-up has no window and may double the pods (or add 4, whichever is more) every 15 seconds. Tune both under `behavior`.
- **Memory is a poor scaling metric** for runtimes that do not hand memory back. A JVM keeps its heap once grown, so memory utilization rises and never falls and the workload never scales in. Scale JVM services on CPU or a request metric.
- **Startup CPU looks like load**: a JVM compiling hot code at start can exceed the target with no traffic. Readiness probes plus the controller's initial CPU-ignore period cover it; a readiness probe that passes too early undoes that.
- **HPA and VPA on the same CPU or memory metric fight** — one changes the denominator the other divides by. Use VPA for recommendations only, or drive the HPA from a custom metric.
- **`minReplicas: 0` is not available by default.** Scaling to zero needs an event-driven autoscaler such as KEDA, which manages an HPA and adds the zero ↔ one step.
- **`maxReplicas` is also a safety limit**: a retry storm or a bot produces real metrics. The cap, plus a budget alert, is what stands between a traffic anomaly and an invoice.
- **PodDisruptionBudgets and graceful termination** decide whether scale-in is clean: handle `SIGTERM`, stop accepting work, finish in-flight requests within `terminationGracePeriodSeconds`.

## What this means for AEM

- **AEMaaCS scales itself**: Adobe runs the orchestration and scales the service tiers automatically; none of the autoscaling settings above are exposed to the customer. What remains the customer's job is code that tolerates it — no reliance on instance-local state, fast startup, and nothing that breaks when a pod is removed mid-request ([aemaacs-architecture-overview](../cloud-service/aemaacs-architecture-overview.md)).
- **AEM 6.5 on AMS or on premise has no elasticity**: capacity is a sizing decision made ahead of time, and adding a publish instance is a provisioning task with a content sync, not a reaction to load ([ams-vs-aemaacs](../cloud-service/ams-vs-aemaacs.md)).
- On both, the CDN and dispatcher cache are the first answer to a traffic spike — a cached response needs no instance at all ([[caching]]).
