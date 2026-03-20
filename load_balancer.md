# ASE Load Balancing Design

## Document Status

* **Document ID**: ASE-LLM-LOAD-BALANCING
* **Document Type**: Subsystem Design
* **Intended Audience**: Platform architects, gateway engineers, inference platform engineers, SREs, operations teams
* **Scope Level**: Detailed subsystem design
* **Related Documents**:

  * `ASE LLM Gateway Architecture Overview`
  * `ASE Semantic Routing Design`

---

## 1. Introduction

This document specifies the design of the **Load Balancing** subsystem in the ASE LLM gateway.

The role of Load Balancing is to determine **which backend instance should execute a request whose target model has already been resolved**. This subsystem operates strictly after Semantic Routing. It does not interpret prompt semantics, classify tasks, or decide which model should be used. Instead, it takes an authoritative `model` assignment from the upstream Semantic Routing layer, resolves the corresponding backend pool, and dispatches the request to an eligible instance according to runtime system state and reliability policy.

The design of this subsystem follows established load-balancer engineering practice rather than semantic decision logic. Professional systems such as NGINX document core upstream balancing methods including round robin, least connections, IP hash, and generic hash, while HAProxy documents active health checks, bounded retries, and redispatch as standard mechanisms for resilient traffic distribution. ASE uses these mature concepts as the basis for its second-layer design. ([NGINX Docs][1])

---

## 2. Scope

### 2.1 In Scope

This document covers:

* model-pool resolution
* endpoint discovery and eligibility
* instance scheduling
* health-aware request dispatch
* retry and redispatch control
* draining and recovery behavior
* affinity and stickiness policies
* runtime metrics ingestion for scheduling
* load-balancing observability

### 2.2 Out of Scope

This document does **not** define:

* semantic model selection
* prompt classification
* policy-driven model choice
* safety-based model restriction
* route hints based on task semantics
* plugin logic tied to Semantic Routing

Those concerns belong to `ASE Semantic Routing Design`.

---

## 3. Problem Definition

Load Balancing solves the following problem:

> Given a request with an already-resolved target model, and a set of backend endpoints capable of serving that model, select the best execution target while preserving availability, stability, and service efficiency.

This is an **instance-selection** problem, not a **model-selection** problem.

The subsystem must work correctly under:

* partial backend failures
* overloaded endpoints
* heterogeneous pools
* streaming requests with variable lifetimes
* uneven request sizes
* mixed internal and external serving targets
* incomplete or heterogeneous backend metrics support

This distinction is important. The vLLM metrics documentation explicitly separates server-level metrics from request-level metrics and exposes them through Prometheus-oriented `/metrics` endpoints, which is exactly the kind of runtime state a Load Balancing layer should consume after the model has already been selected. ([vLLM][2])

---

## 4. Design Goals

### 4.1 Correct Instance Selection

The subsystem should always dispatch only to endpoints that are eligible to serve the selected model.

### 4.2 Availability and Reliability

The subsystem should route around unhealthy, overloaded, or administratively drained endpoints.

### 4.3 Operational Stability

The subsystem should avoid amplifying failures through unbounded retries, unstable stickiness, or indiscriminate redispatch.

### 4.4 Low Scheduling Overhead

Instance selection should be efficient enough to sit in the request path for all traffic.

### 4.5 Clear Separation from Semantic Routing

The subsystem should never reinterpret request meaning under normal operation.

### 4.6 Backend Heterogeneity Tolerance

The subsystem should support a mix of internal inference servers, external APIs, and backends with different observability capabilities.

---

## 5. Non-Goals

The subsystem is not intended to:

* infer task intent from prompts
* choose between model families based on semantics
* replace semantic routing
* enforce high-level business policy that belongs in Layer 1
* perform prompt-level reasoning
* decide quality-versus-cost trade-offs across different models under normal operation

---

## 6. Architectural Position

Within the ASE gateway, Load Balancing is the **second decision layer** in the request path.

**Client Request -> Semantic Routing -> Request Enrichment (`model=...`) -> Load Balancing -> Selected Backend Instance**

The subsystem assumes that the upstream Semantic Routing layer has already produced an authoritative model assignment.

Its contract is therefore:

* input: request with resolved `model`
* output: selected endpoint capable of serving that model

The subsystem must preserve the following rule:

> Load Balancing may choose the serving instance, but it may not change the semantic model assignment under normal operation.

---

## 7. Design Principles

### 7.1 Instance-Centric, Not Model-Centric

This subsystem schedules instances inside a model pool. It does not choose the model.

### 7.2 Health Before Throughput

Unavailable or unstable endpoints must be excluded before optimization among healthy candidates.

### 7.3 Constraint-Then-Scheduling

The scheduler should first determine which endpoints are eligible, then select among them.

### 7.4 Bounded Recovery Semantics

Retries, redispatches, and failover must be policy-controlled and finite. HAProxy documents retries and redispatch as configurable mechanisms rather than open-ended behavior, which aligns with this principle. ([HAProxy Technologies][3])

### 7.5 Affinity as a Preference, Not an Absolute

Stickiness should improve locality where useful, but should not override availability.

### 7.6 Metrics-Aware but Metrics-Portable

The subsystem should use rich backend metrics when available, but must still function when only basic health and latency signals are exposed.

---

## 8. Internal Architecture

The Load Balancing subsystem is composed of six logical components.

### 8.1 Pool Resolver

Maps the resolved `model` field to a backend pool definition.

Responsibilities:

* model-to-pool lookup
* pool membership retrieval
* endpoint metadata loading
* priority and weight loading

### 8.2 Endpoint Registry

Maintains the set of known endpoints and their metadata.

Responsibilities:

* endpoint identity
* endpoint address and protocol
* model affinity
* deployment type
* configured weight
* locality metadata
* drain status

### 8.3 Health Manager

Tracks operational state for each endpoint.

Responsibilities:

* active health-check results
* passive failure observations
* down/degraded/up state
* drain state
* recovery transition control

HAProxy’s reliability documentation describes active health checks as periodic tests that can remove failing servers from rotation and restore them after sufficient successful checks, which is the right foundation for ASE’s health manager. ([HAProxy Technologies][4])

### 8.4 Scheduler

Selects one eligible endpoint for a request.

Responsibilities:

* apply scheduling policy
* enforce affinity if appropriate
* select endpoint among healthy candidates
* support fallback ordering

NGINX documents a standard set of upstream scheduling primitives—round robin, least connections, IP hash, and generic hash—which serve as the natural baseline scheduling vocabulary for ASE. ([NGINX Docs][1])

### 8.5 Reliability Controller

Applies dispatch-time reliability policy.

Responsibilities:

* connect-time retry control
* redispatch control
* failure classification
* failover policy
* recovery suppression for unstable endpoints

HAProxy documents retries and redispatches as mechanisms to reconnect after failed connections or resend after failed HTTP requests, but as explicitly configured behavior rather than implicit unlimited recovery. ASE adopts the same principle. ([HAProxy Technologies][3])

### 8.6 Metrics Adapter

Normalizes runtime signals from different backend types.

Responsibilities:

* ingest generic health and latency telemetry
* ingest backend-native metrics if available
* normalize metrics for scheduling decisions
* expose scheduler-facing runtime state

vLLM documents `/metrics` on the OpenAI-compatible API server and categorizes telemetry into server-level and request-level metrics, which is directly relevant to this component. ([vLLM][2])

---

## 9. Backend Pool Model

### 9.1 Logical Pooling

Each routable model maps to a **logical backend pool**.

Examples:

* `general-small` -> Pool A
* `code-small` -> Pool B
* `code-large` -> Pool C
* `reasoning-large` -> Pool D

The Load Balancing subsystem may only schedule a request to an endpoint belonging to the pool associated with the resolved `model`.

### 9.2 Pool Membership

Each pool entry should define at least:

* endpoint ID
* network address
* protocol type
* backend type
* assigned model or model family
* weight
* priority tier
* drain flag
* locality zone
* health state

### 9.3 Pool Types

ASE should support at least three pool types:

* **homogeneous internal pool**
  multiple replicas of the same internal serving stack

* **heterogeneous internal pool**
  same logical model served by different inference platforms

* **external provider pool**
  multiple remote endpoints or provider regions serving the same logical model contract

This keeps the subsystem deployment-agnostic.

---

## 10. Scheduling Inputs

The scheduler consumes four classes of inputs.

### 10.1 Static Inputs

Examples:

* resolved model
* endpoint weight
* endpoint priority
* deployment zone
* affinity configuration
* drain flag

### 10.2 Health Inputs

Examples:

* active health-check result
* recent connect failures
* recent timeout failures
* backend disabled state

### 10.3 Runtime Load Inputs

Examples:

* active in-flight requests
* queue depth if exposed
* observed latency
* timeout rate
* recent success/error ratio

### 10.4 Affinity Inputs

Examples:

* session ID
* conversation ID
* sticky key
* locality hint
* prefix-affinity tag

---

## 11. Scheduling Pipeline

ASE Load Balancing should execute the following pipeline.

### Step 1: Resolve Pool

Read the authoritative `model` field and identify the corresponding backend pool.

### Step 2: Build Candidate Set

Load the set of endpoints registered for that model.

### Step 3: Apply Hard Exclusions

Remove endpoints that are:

* down
* administratively disabled
* draining
* protocol-incompatible
* explicitly excluded by pool policy

### Step 4: Apply Affinity Preference

If stickiness is enabled and a preferred endpoint is healthy, treat it as a preferred candidate.

### Step 5: Apply Scheduling Algorithm

Select the endpoint according to the configured algorithm.

### Step 6: Dispatch

Forward the request to the selected endpoint.

### Step 7: Observe Outcome

Record latency, success/failure, retry behavior, and any redispatch events.

### Step 8: Update Runtime State

Feed the dispatch outcome back into endpoint health and scheduling state.

---

## 12. Supported Scheduling Algorithms

ASE should support a small, professional set of scheduling modes.

### 12.1 Round Robin / Weighted Round Robin

Round robin is the baseline algorithm for homogeneous pools. NGINX documents round robin as the default balancing method and supports weights for uneven capacity. ([NGINX Docs][1])

Recommended use:

* homogeneous replicas
* low-variance request sizes
* simple internal pools
* initial deployments

### 12.2 Least Connections / Least In-Flight

NGINX documents least connections as a standard method that selects the server with the fewest active connections. In ASE, this should be interpreted more generally as **least in-flight requests** for long-lived inference traffic, especially streaming responses. ([NGINX Docs][1])

Recommended use:

* chat completions
* streaming responses
* variable-length request lifetimes
* interactive inference

### 12.3 Hash / Affinity-Based Routing

NGINX documents IP hash and generic hash as affinity-oriented methods. ASE should adapt this idea to application-level keys such as `session_id`, `conversation_id`, or other stable request affinity keys rather than relying on raw client IP. ([NGINX Docs][1])

Recommended use:

* multi-turn sessions
* locality-sensitive workloads
* deployments where continuity matters
* cache-friendly serving environments

### 12.4 Priority and Weighted Failover

ASE should support primary and backup endpoint classes or weighted preference tiers. This is especially important for mixed deployments where some endpoints are premium, internal, lower-latency, or policy-preferred.

Recommended use:

* private-first routing with external fallback
* regional preference
* premium hardware preference
* controlled degradation modes

### 12.5 Metrics-Aware Scheduling

When backends expose sufficient telemetry, ASE may layer runtime-aware policies on top of the baseline algorithms.

Examples:

* least queue depth
* lowest recent latency
* queue-and-weight hybrid
* overload-avoidance scheduling

This is an ASE design extension rather than a direct requirement from traditional LB documentation.

---

## 13. Health Model

### 13.1 Endpoint States

Each endpoint should be represented by one of the following states:

* **UP**
  Eligible for normal scheduling

* **DEGRADED**
  Eligible but weight-reduced

* **DRAINING**
  No new requests accepted; existing traffic allowed to complete

* **DOWN**
  Removed from scheduling

### 13.2 Active Health Checks

The subsystem should support active probes for:

* TCP connect success
* HTTP readiness endpoint
* model-serving API readiness
* optional lightweight inference probe for high-assurance deployments

HAProxy’s documentation describes active health checks with failure and recovery thresholds, which matches the operational behavior ASE should adopt. ([HAProxy Technologies][4])

### 13.3 Passive Health Signals

The subsystem should also downgrade endpoints based on observed runtime failures, such as:

* repeated connect failures
* repeated timeout failures
* repeated 5xx responses
* sustained elevated latency

### 13.4 Recovery Policy

Recovered endpoints should not necessarily receive full traffic immediately. ASE should support staged re-entry through reduced weight or temporary degraded status, even if the specific control knob differs by deployment.

---

## 14. Retry, Redispatch, and Failover

### 14.1 Retry Principle

Retries must be **bounded**, **failure-class-aware**, and **safe for the request state**.

HAProxy documents retries as a controlled feature and describes redispatch as a way to send a failed request attempt to another server when appropriate. ([HAProxy Technologies][3])

### 14.2 Safe Retry Cases

ASE should allow retry primarily for:

* connection establishment failure
* early transport failure
* backend unreachable before request execution
* request not yet materially streamed

### 14.3 Unsafe Retry Cases

ASE should avoid blind retry for:

* partially streamed responses
* requests with uncertain backend execution state
* non-idempotent upstream side effects
* repeated application-layer failures that indicate semantic or backend correctness problems

### 14.4 Redispatch Policy

If the selected endpoint fails in a retryable way before execution becomes committed, ASE may redispatch to another healthy endpoint in the same pool.

### 14.5 Escalation to Higher-Level Failure

If retries are exhausted or no healthy candidates remain, the subsystem should emit an infrastructure failure rather than silently mutating the model decision.

---

## 15. Affinity and Stickiness

### 15.1 Purpose

Affinity improves continuity and locality. In ASE, affinity is used for:

* multi-turn session continuity
* operational locality
* stable request placement
* optional cache locality

### 15.2 Affinity Keys

Recommended keys:

* `session_id`
* `conversation_id`
* `tenant_id + session_id`
* request-specific stable routing key

### 15.3 Affinity Rule

Affinity should be treated as **preferred placement**, not an unconditional requirement.

If the sticky endpoint is:

* healthy and schedulable -> prefer it
* degraded but still usable -> optionally reduce preference
* down or draining -> ignore stickiness and reschedule elsewhere

### 15.4 Consistent Placement

When consistent placement is desired, ASE may use consistent-hash-style routing over the affinity key so that endpoint membership changes minimize remapping. NGINX documents generic hash and consistent hash configuration in its broader load-balancing materials, which provides a sound implementation reference. ([NGINX Docs][5])

---

## 16. Metrics and Runtime State

### 16.1 Generic Metrics

The subsystem should ingest:

* request rate
* active request count
* success rate
* timeout rate
* error rate
* observed latency
* recent dispatch failures

### 16.2 Backend-Native Metrics

When available, ASE should also ingest backend-specific metrics. vLLM documents `/metrics` and distinguishes between:

* **server-level metrics** for engine state and performance
* **request-level metrics** for timing and request characteristics ([vLLM][2])

Examples of relevant backend-native signals include:

* running requests
* waiting requests
* request latency histograms
* token-throughput indicators
* backend resource pressure

### 16.3 Metrics Portability Principle

Not all backends expose the same telemetry model. Therefore, ASE should normalize metrics into two classes:

* **required scheduler signals**
  health, recent latency, failure rate, active requests

* **optional enhanced scheduler signals**
  queue depth, backend-native request state, token-processing counters

This keeps Load Balancing portable across serving stacks.

---

## 17. Failure Semantics

Load Balancing failures should be clearly separated from Semantic Routing failures.

### 17.1 No Eligible Endpoint

A model was selected successfully, but no endpoint in the corresponding pool is eligible.

### 17.2 Dispatch Failure

An endpoint was selected, but request dispatch failed before stable execution.

### 17.3 Retry Exhaustion

Retryable failures occurred, but recovery attempts were exhausted.

### 17.4 Pool Unavailable

The target model pool exists, but all endpoints are down or draining.

### 17.5 Why This Matters

This failure model allows ASE to expose precise infrastructure outcomes such as:

* `model_resolved_no_healthy_backend`
* `dispatch_failed_connect_timeout`
* `retry_exhausted_same_pool`
* `pool_unavailable_all_draining`

These are operationally far more useful than a generic “routing failed”.

---

## 18. Observability

### 18.1 Core Metrics

The Load Balancing subsystem should emit at least:

* requests dispatched total
* dispatches by model pool
* dispatches by endpoint
* scheduler latency
* retries total
* redispatches total
* health-state transitions
* pool unavailable events
* no-eligible-endpoint events

### 18.2 Recommended Dashboards

Operations should be able to view:

* per-pool request volume
* per-endpoint health state
* per-endpoint latency
* retry and redispatch rates
* error distribution by backend
* drain state distribution

### 18.3 Trace Fields

Recommended trace fields:

* request ID
* resolved model
* resolved pool
* selected endpoint
* scheduling algorithm
* retry count
* redispatch count
* final dispatch outcome

HAProxy’s monitoring materials explicitly surface retries and redispatches as meaningful operational counters, reinforcing that these should be first-class observability fields in ASE as well. ([HAProxy Technologies][6])

---

## 19. Configuration Model

The subsystem should be configured declaratively.

### 19.1 Configuration Domains

* model-to-pool mapping
* endpoint registry
* health-check policy
* retry policy
* redispatch policy
* scheduling algorithm
* stickiness policy
* metrics adapters

### 19.2 Example Logical Structure

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
    - model: code-large
      algorithm: least_inflight
      endpoints:
        - id: code-large-a
          address: 10.0.0.11:8000
          weight: 100
        - id: code-large-b
          address: 10.0.0.12:8000
          weight: 100
    - model: general-small
      algorithm: weighted_round_robin
      endpoints:
        - id: general-small-a
          address: 10.0.1.11:8000
          weight: 100
```

This is intentionally logical rather than implementation-specific.

---

## 20. Security and Operational Considerations

Load Balancing is not the primary semantic governance layer, but it still has important security and operational responsibilities.

It should support:

* explicit endpoint allowlists
* protocol and TLS policy per backend
* controlled external fallback
* backend isolation by tenant or region where required
* safe draining for maintenance
* observability suitable for incident response

This ensures the subsystem remains operationally trustworthy even though its core role is traffic engineering rather than semantic policy.

---

## 21. Summary

ASE Load Balancing is the subsystem responsible for **instance-level dispatch after model selection has already been completed**.

It operates as the second decision layer in the ASE gateway and is responsible for:

* resolving the backend pool for the selected model,
* determining endpoint eligibility,
* selecting an execution target using professional load-balancing policy,
* managing health, retries, redispatch, and failover,
* and exposing operationally meaningful runtime state.

Its design is intentionally grounded in mature load-balancer practice. NGINX’s upstream scheduling methods provide the baseline algorithm vocabulary, HAProxy’s health checks and retry/redispatch model provide the reliability foundation, and backend-native telemetry such as vLLM’s `/metrics` can be incorporated where available to improve runtime-aware scheduling. ([NGINX Docs][1])

Together with `ASE Semantic Routing Design`, this subsystem completes the two-layer ASE LLM gateway architecture:

* **Semantic Routing** decides **what model should run**
* **Load Balancing** decides **where that model should run**

[1]: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/?utm_source=chatgpt.com "HTTP Load Balancing | NGINX Documentation"
[2]: https://docs.vllm.ai/en/stable/design/metrics/?utm_source=chatgpt.com "Metrics - vLLM"
[3]: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/retries/?utm_source=chatgpt.com "Retries and redispatches | HAProxy config tutorials"
[4]: https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/?utm_source=chatgpt.com "Configuration Manual"
[5]: https://docs.nginx.com/nginx-gateway-fabric/reference/api/?utm_source=chatgpt.com "API reference | NGINX Documentation"
[6]: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/alerts-and-monitoring/statistics/?utm_source=chatgpt.com "Statistics dashboard | HAProxy config tutorials"
