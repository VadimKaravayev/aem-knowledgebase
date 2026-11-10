# Chaos engineering — injecting failure on purpose

Deliberately breaking parts of a system, under control, to find out whether the resilience it is supposed to have actually exists. Docs-based. The patterns being tested — timeouts, retries, circuit breakers, bulkheads, fallbacks, redundancy — are in [[reliability-patterns]].

## What it is

- An **experiment**, not random vandalism: a stated hypothesis about how the system behaves under a specific failure, tested by causing that failure and measuring.
- It exists because resilience code is the code that never runs. A fallback, a failover or a retry policy is exercised only when something breaks, so without injected failure its first real test is the incident.
- It differs from testing in what it looks for: a test checks a known expected result; a chaos experiment looks for **what nobody predicted** — the optional dependency that turns out to be critical, the timeout that is longer than the caller's, the alert that never fires.

## The experiment loop

1. **Define steady state** as a measurable output of the system that reflects users, not machines: orders per minute, successful logins, video starts per second. CPU and memory are diagnostics, not the hypothesis.
2. **State the hypothesis**: "if one cache node dies, checkout success rate stays within 1% and p99 latency under 800 ms".
3. **Choose the smallest blast radius** that can disprove it: one instance, one percent of traffic, one test tenant.
4. **Set the abort condition** before starting — the metric value at which the experiment stops automatically — and a way to stop it by hand.
5. **Inject the failure**, compare against a control group or the baseline.
6. **Fix what was found**, then repeat with a wider radius. An experiment that passed is turned into an automated, recurring one so the property stays true.

## Before starting

- **Observability first**: without the steady-state metric on a dashboard and working alerts, an experiment produces an outage and no knowledge ([[observability]]).
- **Do not inject failures whose outcome is already known.** If the service has no timeout on a dependency, add the timeout; breaking it to prove the point teaches nothing.
- **A stop button that works**, and people who know the experiment is running.
- **Start outside production, do not stay there.** Staging lacks real traffic shape, real data volume and real configuration, so it proves less than it appears to. The aim is small, well-bounded experiments in production; the starting point is a test environment.

## What to inject

| Failure | What it tests |
|---|---|
| Kill an instance, pod or process | Redundancy, health checks, restart and rescheduling |
| **Latency** on a dependency | Timeouts, thread-pool exhaustion, circuit breakers |
| Errors from a dependency (`500`, `503`) | Retries, breakers, fallbacks |
| Packet loss, network partition, blackhole | Failure detection, split-brain handling |
| DNS failure | Hidden dependencies, cached resolution |
| CPU, memory, disk or file-descriptor exhaustion | Autoscaling, limits, noisy-neighbour isolation |
| Clock skew | Token expiry, ordering assumptions, certificates |
| Loss of a zone or region | Capacity headroom, failover, data replication |

**Latency finds more than death.** A dead dependency fails fast and is handled by the obvious code path; a slow one holds threads and connections and exposes missing timeouts and unbounded pools. If only one experiment is run, make it added latency.

## Game days

A scheduled exercise where a team injects a failure (or has it injected without knowing which) and works the incident as if it were real. It tests the people and the procedure as much as the system: whether the alert reached someone, whether the runbook is correct, whether access works at that hour. Automated experiments keep known properties true; game days find the unknown ones.

## Tools

### Chaos Monkey

- Built by Netflix during its move to the cloud and later released as open source. It **randomly terminates instances in production, during business hours**, so that engineers are present when something breaks.
- The purpose is cultural as much as technical: when any instance may disappear on any working day, every service gets built to survive it — stateless, redundant, automatically replaced.
- Scope is deliberately narrow: **instance termination only**. No latency, no network faults, no resource pressure. The current version depends on Spinnaker for deployment metadata.
- It was the first of the **Simian Army** — Latency Monkey (delays), Chaos Gorilla (a whole availability zone), Chaos Kong (a whole region) and others. The Simian Army project has been retired; the names remain the standard vocabulary for failure scale.

### Gremlin

- A **commercial, hosted** chaos platform ("failure as a service") started by engineers who had run this practice at large companies. An agent is installed on hosts, containers or Kubernetes nodes; experiments are launched from a web console or API.
- Faults come in three families: **resource** (CPU, memory, disk, I/O), **state** (shutdown, process kill, time shift) and **network** (blackhole, latency, packet loss, DNS).
- What is being paid for: a broad fault library that works across virtual machines, containers and Kubernetes; precise targeting and blast-radius control; a **halt button** and automatic rollback; scheduled scenarios and reporting.
- Fits organizations that want the practice without building and maintaining the tooling, and that have mixed, not only Kubernetes, infrastructure.

### Litmus

- **Open source, Kubernetes-native**, a Cloud Native Computing Foundation project. Experiments are Kubernetes resources, so they are versioned, reviewed and applied like any other manifest.
- Three custom resources: **`ChaosExperiment`** (the definition of a fault and its parameters), **`ChaosEngine`** (binds an experiment to a target application and starts it), **`ChaosResult`** (the verdict).
- **ChaosHub** is a catalogue of ready-made experiments — pod delete, container kill, pod network latency and loss, CPU and memory hog, node drain, plus faults for cloud provider resources.
- **Probes** check steady state during the run (HTTP request, command, Kubernetes resource, Prometheus query) and decide pass or fail, which makes an experiment usable as a pipeline gate. A web console (ChaosCenter) chains experiments into workflows.

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: web-chaos
  namespace: shop
spec:
  engineState: active
  appinfo:
    appns: shop
    applabel: app=web
    appkind: deployment
  chaosServiceAccount: pod-delete-sa
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION   # seconds
              value: "60"
            - name: CHAOS_INTERVAL         # seconds between kills
              value: "10"
            - name: FORCE
              value: "false"
```

### Choosing

| | Chaos Monkey | Gremlin | Litmus |
|---|---|---|---|
| Licence | Open source | Commercial service | Open source |
| Runs on | Cloud instance groups via Spinnaker | Hosts, containers, Kubernetes | Kubernetes |
| Faults | Instance termination | Resource, state, network | Pod, node, network, resource, cloud |
| Experiments defined as | Configuration and schedule | Console or API | Kubernetes manifests |
| Best fit | Proving instances are disposable | Mixed infrastructure, little time for tooling | Kubernetes teams, experiments in Git and CI |

Others worth knowing: **Chaos Mesh** (also Kubernetes-native, also a CNCF project), the cloud providers' own fault-injection services (AWS Fault Injection Service, Azure Chaos Studio), **Chaos Toolkit** (experiments as declarative files, provider-neutral), and **Toxiproxy** (a proxy that adds latency and faults to a connection — for integration tests rather than production).

## Traps

- **Chaos without a hypothesis** is an outage with a calendar invite. Nothing is learned because nothing was predicted.
- **Findings nobody fixes.** The value is in the fix; an experiment programme that only produces reports erodes trust in itself.
- **Staging-only confidence**: passing in an environment with a tenth of the traffic and none of the real data.
- **Testing only death, never slowness** (above).
- **No abort condition**, or one watched by a person who has stepped away.
- **Running it once.** Systems change weekly; a property verified last quarter is unverified now. Automate the experiments that matter.
- **Starting too big.** Killing a region before the service survives the loss of one instance.

## What this means for AEM

- **On AEMaaCS the platform is not yours to break**: there is no access to pods, nodes or the network, so infrastructure faults cannot be injected and Adobe's own resilience is not something a customer verifies this way.
- **What is testable is your own code's behaviour toward its dependencies**: an AEM page or servlet that calls a commerce, search or identity service should be exercised with that service slow and failing — on the local SDK or a development environment, with a fault-injecting proxy in front of the dependency. The usual finding is an HTTP client with no timeout holding request threads.
- **On AEM 6.5 that you operate**, the classic experiments apply directly: stop one publish instance behind the load balancer, block replication to it, fill a disk, and check what the dispatcher serves meanwhile.
