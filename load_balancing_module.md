# ASE Load Balancing Module Design

## Introduction

This document defines the design of the Load Balancing Module in the ASE Semantic Routing and Load Balancing architecture. The Load Balancing Module is the second decision stage in the request path and is responsible for selecting which concrete provider endpoint should execute a request whose target logical model has already been resolved.

This document is not a generic overview of traffic distribution. It is the governing design for the execution-balancing module that receives an authoritative logical-model decision from the Semantic Routing Module, resolves the corresponding endpoint pool, and selects an execution target under runtime health, capacity, affinity, and reliability constraints.

Within the overall document set, this file defines instance-level dispatch only. System-wide architecture is defined in `architecture.md`, and model-level decision making is defined in `semantic_routing_module.md`.

## Background

### Module Problem

The Load Balancing Module solves the following problem:

> Given a request with an already resolved target logical model, and a set of provider endpoints capable of serving that logical model, select an execution target that preserves availability, stability, and service efficiency.

This is a provider-endpoint-selection problem, not a logical-model-selection problem. The semantic question has already been answered upstream. What remains is to map the selected logical model onto a serving endpoint under current operational conditions.

In production, this problem is complicated by partial endpoint failures, uneven request sizes, long-lived streaming requests, heterogeneous backends, administrative drain behavior, and inconsistent telemetry quality across serving systems.

### Why This Module Must Exist

The Load Balancing Module exists because choosing where a request should execute is a different problem from choosing what logical model should execute it.

Once the logical model is fixed, the system must reason about endpoint weight, provider choice, backend health, in-flight load, queue pressure, affinity, retries, redispatch, region, and recovery behavior. None of these concerns should require the module to reinterpret prompt semantics or revise the logical-model decision under normal operation.

ASE therefore isolates the Load Balancing Module as the module that owns endpoint choice and runtime dispatch behavior. It must honor the logical-model assignment it receives and operate within that constraint.

### Design Objectives

The module is designed to achieve the following outcomes:

- dispatch only to provider endpoints that are eligible to serve the selected logical model
- route around unhealthy, overloaded, or administratively drained endpoints
- keep retries, redispatch, and failover bounded and predictable
- maintain scheduling overhead low enough for the online request path
- preserve strict separation from semantic logical-model selection
- support heterogeneous backends with varying telemetry richness

### Governing Principles

The module follows five governing principles.

First, the Load Balancing Module is endpoint-centric, not logical-model-centric. Second, health and eligibility must be evaluated before throughput optimization. Third, scheduling decisions should be based on explicit runtime state and declarative policy rather than hidden side effects. Fourth, retries and redispatch must be finite and observable. Fifth, affinity is a preference that improves locality, not an excuse to route to unhealthy endpoints.

ASE aligns this module with mature load-balancer practice. The scheduling vocabulary used by NGINX, the health and retry controls documented by HAProxy, and rich runtime metrics from systems such as vLLM provide the right conceptual foundation for the module. See [R3], [R4], and [R5].

## Scope

### In Scope

This document defines:

- logical-model-to-pool resolution
- endpoint discovery and eligibility evaluation
- health-aware instance scheduling
- runtime dispatch behavior
- retry, redispatch, and bounded failover
- drain and recovery behavior
- affinity and stickiness policy
- runtime metrics ingestion for scheduling
- observability requirements for dispatch decisions
- configuration and operational requirements specific to instance-level routing

### Out of Scope

This document does not define:

- semantic logical-model selection
- prompt classification
- policy-driven model-family choice
- semantic safety gating that belongs before logical-model resolution
- plugin logic tied to semantic decision making

Those responsibilities belong to `semantic_routing_module.md` or to broader ASE control-plane components outside this module.

## Design

### Module Summary

The Load Balancing Module is the authoritative endpoint-selection module in ASE. It accepts a request whose `model` already carries a resolved logical model, maps that logical model to an endpoint pool, filters provider endpoints using health and eligibility rules, applies scheduling policy, and dispatches the request to a concrete execution target.

The key property of the module is that it resolves a provider endpoint, not a logical model. Its value comes from making runtime dispatch reliable, observable, and operationally tunable without collapsing back into semantic decision making.

### Role in the Two-Stage Process

ASE follows a `select -> route` process.

- The Semantic Routing Module owns semantic analysis and logical-model selection.
- The Load Balancing Module owns endpoint-pool resolution and provider-endpoint routing.
- This module corresponds to the execution `route` stage, not the semantic `select` stage.

This means the Load Balancing Module is primarily a resource-scheduling and availability module. It decides where the already selected capability should execute. It does not decide what capability class the request needs.

### Module at a Glance

The diagram below summarizes the Load Balancing Module before the more detailed internal design diagram.

```mermaid
flowchart LR
    A[Logical Model] --> B[Endpoint Pool Resolution]
    B --> C[Eligible Provider Endpoints]
    C --> D[Scheduler<br/>weight + affinity + locality + load]
    D --> E[Selected Provider Endpoint]
    E --> F[Dispatch / Retry / Failover]
```

### Architectural Position

The Load Balancing Module appears in the request path as follows:

`Client Request -> Semantic Routing Module -> Request Enrichment (logical model in model field) -> Load Balancing Module -> Selected Provider Endpoint`

Its contract is:

- input: request with resolved logical model in `model` plus route metadata
- output: selected provider endpoint plus dispatch behavior governed by runtime policy

That contract is normative. The Load Balancing Module may decide where the selected logical model runs, but it may not change what logical model the request has been assigned.

The module should also produce a stable dispatch outcome contract for observability and downstream control paths.

| Field | Requirement Level | Purpose |
| --- | --- | --- |
| `request_id` | required | preserve request identity across semantic and infrastructure decisions |
| `model` | required | record which semantic logical-model decision this dispatch honored |
| `pool_id` | required | identify the endpoint pool chosen for execution |
| `provider_id` | optional | identify which provider or serving system supplied the chosen endpoint |
| `endpoint_id` | required on success | identify the concrete provider endpoint that was selected |
| `dispatch_status` | required | distinguish success, immediate failure, retry exhaustion, and pool unavailability |
| `retry_count` | optional | preserve how much bounded recovery was attempted |
| `redispatch_count` | optional | record whether recovery required switching endpoints |
| `dispatch_reason` or algorithm detail | optional | explain why the endpoint was chosen or why dispatch failed |

### System Design Diagram

The diagram below shows the internal flow of the Load Balancing Module after logical-model resolution has already completed.

```mermaid
flowchart LR
    Request[Enriched Request<br/>logical model + route metadata]

    subgraph Runtime[Runtime State and Configuration]
        Registry[Endpoint Registry]
        Health[Health Manager]
        Metrics[Metrics Adapter]
        Policy[Reliability and Affinity Policy]
    end

    subgraph LB[Load Balancing Module]
        Pool[Pool Resolver]
        Candidates[Candidate Set Builder]
        Filter[Eligibility and Health Filter]
        Scheduler[Scheduler]
        Dispatch[Reliability Controller<br/>dispatch, retry, redispatch]
        Backend[Chosen Provider Endpoint]
        Failure[Infrastructure Failure<br/>no eligible endpoint or retry exhausted]
    end

    Obs[Observability]

    Request --> Pool --> Candidates --> Filter --> Scheduler --> Dispatch --> Backend
    Registry --> Pool
    Registry --> Candidates
    Health --> Filter
    Metrics --> Scheduler
    Policy --> Scheduler
    Policy --> Dispatch

    Filter -->|no eligible endpoint| Failure
    Dispatch -->|retry exhausted| Failure

    Backend -. outcome .-> Health
    Backend -. runtime metrics .-> Metrics
    Scheduler -. scheduling trace .-> Obs
    Dispatch -. dispatch metrics .-> Obs
    Failure -. failure events .-> Obs
```

### Architectural Invariants

The following invariants are mandatory for this module.

1. The Load Balancing Module must not begin until the Semantic Routing Module has resolved the authoritative logical model in `model`.
2. Endpoint selection must remain constrained to the endpoint pool for that logical model.
3. Endpoint health and hard eligibility must be evaluated before scheduling optimization.
4. Retry and redispatch behavior must be explicit, finite, and observable.
5. Failure classification must preserve the distinction between semantic success and infrastructure failure.

These invariants define acceptable dispatch behavior even as backend technologies and scheduling implementations evolve.

### Internal Architecture

The module is composed of six logical components.

| Component | Primary Responsibility | Architectural Output |
| --- | --- | --- |
| Pool Resolver | map the resolved logical model to a logical endpoint pool | pool identity and pool configuration |
| Endpoint Registry | provide endpoint inventory and static metadata | candidate endpoint universe for the selected pool |
| Health Manager | maintain endpoint serviceability state | health and drain state used for hard filtering |
| Scheduler | choose the best provider endpoint among eligible candidates | selected provider endpoint and scheduling rationale |
| Reliability Controller | apply retry, redispatch, and dispatch-time failover policy | dispatch outcome and bounded recovery behavior |
| Metrics Adapter | normalize generic and backend-native telemetry | scheduler-facing runtime metrics |

The architecture is intentionally staged. Pool resolution defines the search space, endpoint and health data constrain that space, the scheduler selects within it, and the reliability controller manages the execution attempt.

### Backend Pool Model

The Load Balancing Module schedules within logical endpoint pools associated with the resolved logical model.

Each pool should be treated as the execution domain for a specific semantic decision. The module should not search globally across all backends once a logical model has already been selected.

The same logical model may be backed by multiple provider endpoints. This is a first-class use case, not an exception path. The purpose of the Load Balancing Module is to route among those equivalent or near-equivalent endpoints under runtime policy without reopening semantic model selection.

Each pool definition should expose, at minimum:

- the resolved logical model or logical-model alias it serves
- endpoint membership
- provider identity
- endpoint priority and weight
- backend type and protocol information
- deployment-zone or locality metadata
- any pool-specific scheduling or reliability overrides

ASE should support more than one kind of pool, including homogeneous internal pools, heterogeneous weighted pools, external provider pools, hybrid fallback pools, and compliance-scoped pools restricted by tenant or region.

### Scheduling Inputs

The scheduler operates on a structured runtime view assembled from four input classes.

| Input Class | Purpose | Representative Inputs |
| --- | --- | --- |
| Static Pool Inputs | define the dispatch domain and configured policy | selected logical model, pool membership, provider, weights, locality, endpoint priority |
| Health Inputs | determine whether an endpoint is serviceable | active health state, passive failure rate, circuit state, drain state |
| Runtime Load Inputs | describe current execution pressure | in-flight requests, queue depth, latency, token throughput, rate-limit status, provider-side quota state |
| Affinity Inputs | preserve useful locality where possible | session ID, conversation ID, tenant ID, sticky hash |

Without these inputs, endpoint choice becomes blind traffic spreading. With them, the Load Balancing Module becomes a controlled scheduling problem grounded in current system state.

### Dispatch Framework

Dispatch should be understood as a staged reduction process rather than a single opaque scheduling step.

#### Stage 1: Resolve Pool

The module maps the resolved logical model in the `model` field to the logical endpoint pool that is allowed to serve it. This establishes the execution domain for the request.

#### Stage 2: Build Candidate Set

The module retrieves all endpoints that belong to the selected pool together with their static metadata and current runtime state.

#### Stage 3: Apply Hard Exclusions

Endpoints that are down, draining, policy-ineligible, locality-ineligible, or otherwise incapable of serving the request are removed first. Scheduling must not optimize over endpoints that should never receive traffic.

#### Stage 4: Apply Affinity Preference

If stickiness is enabled, the module should prefer the previously associated endpoint when it remains healthy and eligible. Affinity is applied after hard exclusion because locality cannot override serviceability.

#### Stage 5: Select Endpoint

The scheduler chooses the best provider endpoint among the remaining candidates using the configured scheduling algorithm and available runtime metrics.

#### Stage 6: Dispatch and Observe

The module dispatches the request, classifies the outcome, and updates health, reliability, and observability state accordingly.

This staged framework is central to explainability. It allows operators to understand not only which provider endpoint was chosen, but why other endpoints were excluded or deprioritized.

### Scheduling Model

The Load Balancing Module should support a family of scheduling strategies rather than a single fixed algorithm. Different backend topologies favor different policies.

| Scheduling Strategy | Best Fit | Primary Tradeoff |
| --- | --- | --- |
| Weighted Random / Weighted Round Robin | simple or relatively homogeneous pools | low overhead, limited runtime sensitivity |
| Least Connections / Least In-Flight | variable-duration or streaming workloads | better concurrency awareness, needs current state |
| Hash / Affinity-Based Placement | cache locality or session continuity sensitive workloads | stronger locality, weaker short-term fairness |
| Priority / Weighted Failover | primary-backup or tiered backend topologies | explicit preference ordering, less even spread |
| Metrics-Aware Scheduling | rich telemetry environments | better adaptation, stronger dependency on telemetry quality |

The initial ASE deployment can start with simpler strategies such as weighted random, weighted round robin, or least in-flight and evolve toward richer metrics-aware approaches without changing the module boundary.

### Advanced Scheduling Strategies

In addition to baseline strategies, ASE should treat several LLM-specific scheduling patterns as first-class policies rather than ad hoc implementation details.

| Strategy | Required Signals | Best Fit | Primary Caveat |
| --- | --- | --- | --- |
| Consistent Hashing / Stable Affinity | session ID, conversation ID, or request-class hash | preserve KV-cache or prefix-cache locality across related requests | locality must be breakable when the preferred endpoint is unhealthy or overloaded |
| Power of Two Choices | lightweight load signal such as in-flight requests or queue depth | better short-term balance than random selection at low overhead | still depends on reasonably fresh load state |
| Least Queue / Queue-Aware | queue depth, waiting requests, or admission backlog | prefill-heavy or bursty clusters where waiting time dominates | poor queue telemetry leads to poor choices |
| Token-Aware Scheduling | prompt-token estimate, `max_tokens`, token throughput | highly variable request sizes where raw request count is misleading | estimates are approximate and should not be treated as exact cost |
| Cache-Aware / Prefix-Aware Routing | prefix hash, cache-hit probability, or session locality hint | repeated system prompts, agent workloads, or templated RAG requests | can reduce fairness if locality is over-weighted |

These strategies are composable. For example, ASE may use Power of Two Choices to reduce candidate-selection overhead, then break ties using least in-flight load or cache-locality preference.

### Health Model

Endpoint health must be modeled explicitly. Dispatch quality depends on being able to distinguish endpoints that are healthy, degraded, unavailable, or draining for administrative reasons.

Recommended endpoint states include:

- `up`
- `degraded`
- `down`
- `draining`

Health state should be informed by both active and passive signals. Active checks detect connectivity or service-level failure independently of live traffic. Passive signals react to observed outcomes such as connection failure, timeout, HTTP failure classes, or abrupt stream termination.

Recovery must also be policy-controlled. An endpoint that has just recovered should not necessarily receive full traffic immediately. Gradual re-entry or controlled reinstatement is preferable when recent instability suggests recovery may be incomplete.

Circuit-breaking state may be layered on top of endpoint health. An `open` circuit suppresses traffic to a recently failing endpoint, while a `half_open` circuit permits controlled probes before the endpoint is returned to normal service. Circuit state is an operational safety mechanism and should remain visible in dispatch telemetry.

### Reliability Model

Reliability behavior is part of the module design, not an implementation afterthought.

Retries should be finite and policy-controlled. Redispatch should occur only when the failure mode and request semantics make it safe to do so. Failover should remain bounded and observable rather than open-ended.

Safe retry scenarios usually involve failures before execution begins, such as connection failure, TLS establishment failure, or immediate admission rejection. Unsafe retry scenarios usually involve the possibility that execution already began, such as partial streamed responses, ambiguous upstream timeouts, or tool side effects.

For streaming responses, redispatch is usually unsafe once bytes or tokens have already been relayed to the caller. Unless the upstream protocol provides explicit resumable semantics, ASE should treat post-stream-start failures as terminal execution failures rather than silently replaying them on another backend.

If retry and redispatch are exhausted, the module must emit a clear infrastructure failure rather than silently collapsing the error into a semantic-routing problem.

### Affinity and Locality Model

Affinity exists to improve locality, not to weaken availability.

Useful affinity keys include session ID, conversation ID, tenant ID, and request-class hashes that approximate cache reuse or backend warm state. Affinity may improve session continuity, prefix-cache locality, and hot-state reuse, but it should always remain subordinate to endpoint eligibility and health.

Consistent placement strategies may be used where stable assignment matters, but the module must still be able to break affinity when the preferred endpoint is unhealthy, draining, or overloaded beyond policy.

### Metrics and Runtime State

The module should be able to operate correctly using generic telemetry and improve when richer backend-native telemetry exists.

Generic metrics that should work across backend types include:

- endpoint health state
- request latency
- in-flight requests
- error rate
- timeout rate
- dispatch success rate

When available, richer backend-native telemetry may also be incorporated, such as queue depth, token throughput, GPU utilization, KV cache usage, and prefix-cache hit potential. vLLM-style metrics endpoints are particularly useful in this category. See [R5].

The architectural principle is portability: ASE should benefit from rich metrics without depending on a single serving stack in order to function correctly.

### Canonical Runtime Metrics Vocabulary

ASE should normalize backend-native telemetry into a small canonical vocabulary before the scheduler consumes it. Scheduling policy should target logical metrics rather than vendor-specific metric names.

| Canonical Metric | Meaning | Example Backend-Native Signal | Primary Use |
| --- | --- | --- | --- |
| `requests_running` | active requests currently executing on an endpoint | `vllm:num_requests_running` | least in-flight and overload detection |
| `requests_waiting` | requests queued or waiting for admission | `vllm:num_requests_waiting` | least-queue and queue-aware scheduling |
| `requests_swapped` | requests displaced due to memory pressure | `vllm:num_requests_swapped` | detect degraded service and memory stress |
| `ttft_seconds` | time to first token | `vllm:time_to_first_token_seconds` | user-visible latency and endpoint scoring |
| `tpot_seconds` | average time per output token | `vllm:time_per_output_token_seconds` | decode-speed scoring for streaming workloads |
| `e2e_latency_seconds` | end-to-end request latency | `vllm:e2e_request_latency_seconds` | overall service quality and regression detection |
| `prompt_tokens_total` | prompt token throughput counter | `vllm:prompt_tokens_total` | throughput and token-aware cost estimation |
| `generation_tokens_total` | generation token throughput counter | `vllm:generation_tokens_total` | decode throughput and capacity planning |
| `gpu_kv_cache_usage_ratio` | GPU-side KV-cache pressure | `vllm:gpu_cache_usage_perc` | cache-aware scoring and overload protection |
| `cpu_kv_cache_usage_ratio` | CPU-side KV-cache pressure | `vllm:cpu_cache_usage_perc` | swapped-state diagnosis and pressure scoring |
| `prefix_cache_hit_rate` | reuse potential for repeated prompts | `vllm:prefix_cache_hit_rate` | cache-aware or prefix-aware routing |
| `gpu_utilization_ratio` | approximate accelerator saturation | backend-specific GPU metric | capacity and hotspot detection |

Not every backend will expose every metric. The normalization layer should preserve missing values explicitly so that schedulers can degrade gracefully rather than pretending absent telemetry is a healthy zero.

### Backend Compatibility and Degradation Model

Not all inference backends expose the same operational surface. Some provide rich `/metrics` and `/health` endpoints, while others provide only coarse availability signals or no scheduler-friendly telemetry at all.

ASE should therefore classify backend integrations by capability level.

| Capability Level | Typical Signals Available | Scheduling Modes That Fit |
| --- | --- | --- |
| Level 0: minimal | passive failure observation, connect success, HTTP status | weighted round robin, priority failover |
| Level 1: basic | active health, request latency, in-flight count | least in-flight, Power of Two Choices |
| Level 2: queue-aware | waiting queue, token throughput, admission signals | least queue, token-aware scheduling |
| Level 3: cache and accelerator aware | KV-cache usage, prefix-cache hit rate, GPU utilization | cache-aware, prefix-aware, richer metrics-aware scheduling |

This compatibility model is operationally important. A backend that lacks `/metrics` should not be excluded from ASE, but it should be scheduled using simpler and more conservative policies. Likewise, if `/health` is missing, ASE should fall back to passive health signals or synthetic probes rather than assuming full observability.

Provider-specific request adaptation, including auth injection or protocol shaping, may occur after endpoint selection. That work belongs to this module or its execution adapters because it depends on the chosen provider endpoint, not on semantic model selection.

### Failure Semantics

The Load Balancing Module must classify failures by dispatch stage so that operators can distinguish an endpoint-selection failure from a semantic-routing failure.

| Failure Class | Meaning | Typical Cause |
| --- | --- | --- |
| No Eligible Endpoint | the selected logical model resolves to a pool, but no endpoint is eligible to serve it | all endpoints down, draining, policy-ineligible, or filtered out |
| Dispatch Failure | an eligible endpoint was chosen, but the immediate dispatch attempt failed | connect failure, admission failure, protocol failure |
| Retry Exhaustion | a dispatch failure was retried or redispatched within policy, but all attempts failed | repeated connect failures, repeated endpoint instability |
| Pool Unavailable | the pool as a whole cannot currently serve traffic | pool-wide outage, full drain, region-wide restriction |

These categories matter because they let operators distinguish semantic success followed by infrastructure failure, isolated endpoint instability, and pool-wide unavailability.

### Observability

The Load Balancing Module must expose operationally useful telemetry because its decisions are driven by runtime state and often change rapidly under load.

Core metrics should include:

- pool-resolution count
- endpoint-selection count by endpoint, provider, and pool
- dispatch latency
- retry count
- redispatch count
- per-endpoint failure rate
- pool-unavailable count
- drain-state traffic volume

Useful trace or log fields include request ID, selected logical model, resolved endpoint pool, selected provider, selected provider endpoint, scheduling algorithm, retry count, redispatch count, and final dispatch outcome.

The minimum dispatch trace should preserve enough state to reconstruct the path from resolved logical model, to pool, to candidate filtering, to selected provider endpoint, to final dispatch result.

### Configuration Model

The module should be configured declaratively rather than through code changes. This is necessary so that scheduling and reliability policy remain reviewable and tunable as fleet conditions evolve.

The configuration surface should at least cover:

- logical-model-to-pool mapping
- endpoint registry
- provider metadata and execution-adapter binding
- health-check policy
- retry policy
- redispatch policy
- scheduling algorithm selection
- scheduler fallback policy by backend capability tier
- stickiness or affinity policy
- metrics-adapter bindings

An illustrative logical configuration is shown below.

```yaml
load_balancing:
  default_algorithm: least_inflight
  retry_policy:
    max_retries: 2
    redispatch_on_connect_failure: true
  affinity_policy:
    enabled: true
    key: session_id
    mode: preferred
  pools:
    - logical_model: code-high-capacity
      algorithm: least_inflight
      metrics_capability: level_2
      endpoints:
        - id: code-large-a
          provider: internal-vllm
          address: 10.0.0.11:8000
          weight: 100
        - id: code-large-b
          provider: internal-vllm
          address: 10.0.0.12:8000
          weight: 100
    - logical_model: general-general-purpose
      algorithm: weighted_round_robin
      metrics_capability: level_0
      endpoints:
        - id: general-small-a
          provider: external-provider-a
          address: 10.0.1.11:8000
          weight: 100
```

The exact DSL is implementation-specific. The architectural requirement is that dispatch and reliability behavior remain declarative, reviewable, and versioned.

### Security and Operational Implications

Although the Load Balancing Module is not the semantic governance layer, it is still a security- and operations-relevant module because it controls the actual execution path to backend systems.

At minimum, it should support:

- explicit endpoint allowlists
- protocol and TLS policy per backend
- controlled external fallback
- backend isolation by tenant or region when required
- safe draining for maintenance
- telemetry suitable for incident response

These controls make the dispatch layer operationally trustworthy without turning it into a semantic policy engine.

### Architectural Consequences

This design makes dispatch behavior tunable and observable, but it also imposes obligations on the implementation.

The module must maintain a stable logical-model-to-pool contract, preserve explicit health semantics, and keep retry and redispatch behavior visible to operators. It must also resist architectural drift. If prompt meaning or business policy starts to directly drive endpoint selection inside this module, or if this module silently rewrites logical-model choice during failure handling, the boundary with the Semantic Routing Module has been violated and the design has degraded.

## References

- [R1] `architecture.md`, ASE Semantic Routing and Load Balancing Architecture
- [R2] `semantic_routing_module.md`, ASE Semantic Routing Module Design
- [R3] NGINX HTTP Load Balancing Documentation, https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/
- [R4] HAProxy Reliability and Health Check Documentation, https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/
- [R5] vLLM Metrics Documentation, https://docs.vllm.ai/en/stable/design/metrics/
