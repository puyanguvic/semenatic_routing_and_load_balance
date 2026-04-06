ASE Semantic Router

Author: Pu Yang/84399478， Rui Zhou/8400631

[[_TOC_]]

---

# Introduction

Traditional API gateways are designed for conventional application traffic. They typically front diverse microservices and focus on northbound request handling, authentication, security policy, protocol mediation, and service routing. AI gateway behavior is different. It must broker access to heterogeneous external LLM providers and self-hosted model stacks, where prompt meaning, data sensitivity, modality, context size, and governance constraints may determine the legal and appropriate model path before inference begins.

Within that access path, semantic routing and load balancing are separate functions. Semantic routing selects a legal capability path, model family, or target pool from normalized request context and governance constraints. Load balancing selects a healthy backend endpoint within the selected pool. ASE Semantic Router therefore operates on request semantics rather than transport metadata and MUST emit an explainable and auditable `RouteDecision` for ASE LLM Load Balancer and any downstream serving stack.

This boundary also defines feature ownership. Request understanding, provider connectivity, policy enforcement, semantic classification, model-family selection, and route explanation belong to ASE Semantic Router. Inference optimization, replica scheduling, KV-cache efficiency, batching, and endpoint-level failover belong to ASE LLM Load Balancer or to downstream self-hosted serving systems.

# Background

Current AI gateway implementations generally develop from two directions. One direction extends traditional API gateways, such as Kong AI Gateway or Cloudflare AI Gateway, to recognize LLM traffic and route across providers or model classes. The other direction starts from self-hosted serving systems, such as Ollama, vLLM, or SGLang, that are optimized for execution and scheduling within a deployed model environment. These approaches address different parts of the request path.

ASE separates those concerns explicitly. Semantic routing determines whether a request should go to an external provider, an internal provider pool, or a self-hosted model family by using normalized request context, semantic signals, and governance policy. Downstream load balancing and serving components then optimize execution within the selected pool or deployment. Those downstream components may use engine-native metrics, queue state, batch formation, and cache locality, but those signals are runtime scheduling inputs rather than semantic route inputs.

Many existing LLM routing systems interleave semantic interpretation, policy enforcement, and backend dispatch within one component. That coupling makes route selection harder to explain, govern, and audit. ASE instead treats semantic routing as a separate request pipeline whose output is a pool-level decision rather than an endpoint choice. Future enhancements SHOULD therefore be classified first by ownership: semantic routing and governance, or downstream execution and inference optimization.

## Conventions and Terminology

The uppercase keywords `MUST`, `SHOULD`, and `MAY` in this document indicate normative requirements. When those words appear in lowercase, they are used in their ordinary descriptive sense.

In this document, Semantic Router refers to the stage that selects a legal capability path and target pool. Load Balancer refers to the downstream stage that selects a concrete backend endpoint within that pool. A target pool is therefore a semantic dispatch domain rather than an individual provider endpoint or replica.

## Scope

# System Architecture

## ASE Semantic Router Block Diagram

The semantic router processing logic as below:

<div align="center">
<img src="./assets/ase_semantic_router.svg" alt="ASE Semantic Router Block Diagram" width="1000"/>
</div>

The following flows describe the stages owned by ASE Semantic Router. 

## Semantic Router Request Path

```mermaid
flowchart LR
    A[Receive Request] --> B{Request Valid?}
    B -- No --> E1[Return Validation Error]

    B -- Yes --> N[Normalize Request]
    N --> C[Build Routing Context]
    C --> O{Override Present?}

    O -- Yes --> P{Override Authorized and Compatible?}
    P -- No --> E2[Return Policy Error]
    P -- Yes --> PL[Apply Route Plugins]

    O -- No --> S[Extract Routing Signals]
    S --> R[Evaluate Decision Rules]
    R --> T[Build Candidate Pool Set]
    T --> G{Target Pool Selected?}
    G -- No --> E3[Return No Route Error]
    G -- Yes --> PL

    PL --> D[Emit RouteDecision]
    D --> H[Handoff to ASE LLM Load Balancer]
```

The request path MUST normalize the inbound payload into a canonical routing context before semantic selection begins. When an explicit model or route override is present, the override MAY bypass rule evaluation, but it MUST still pass authorization, capability, and policy compatibility checks. When no override applies, the router MUST extract routing signals, evaluate decision rules, derive a legal candidate pool set, and emit a single explainable `RouteDecision` for downstream scheduling.

## Semantic Router Response and Post-Processing Path

```mermaid
flowchart LR
    A[Receive Downstream Response] --> B{Request Context Correlated?}
    B -- No --> X[Drop or Apply Error Policy]

    B -- Yes --> P{Post-Response Plugins Enabled?}
    P -- No --> R[Return Response]
    P -- Yes --> S[Update Session Metadata]
    S --> T[Write Audit Trace]
    T --> C[Update Semantic Cache]
    C --> R[Return Response]
```

The response path begins only after the downstream response has been correlated with the original request context. If no valid correlation exists, the router MUST drop the response or apply implementation-specific error policy; it MUST NOT attach post-processing state to an unrelated request. Post-response plugins MAY update session continuity metadata, audit trace state, and semantic cache entries before the response is returned to the caller.

# Design Modules

## LLM Listener

Please refer to the corresponding section of [ASE semantic load balancer](./ase_semantic_load_balancer.md).

## Ports

Please refer to the corresponding section of [ASE semantic load balancer](./ase_semantic_load_balancer.md).

## LLM REST API Service

Please refer to the corresponding section of [ASE semantic load balancer](./ase_semantic_load_balancer.md).

## Semantic Router

### Core Data Structures

#### Routing Context

A routing context is the canonical object used by semantic routing.

| Routing Context Class           | Major fields                                                                                                  | Purpose                                                    |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Request Content                 | messages, prompt text, system instructions, tool requirements, multimodal metadata, output format requirement | Describe what the request is asking for                    |
| Control Metadata                | `model`, `routing_hint`, `route_override`, `preference`, `input_tokens_estimate`, debug flags                 | Express caller routing intent or optimization hints        |
| Identity and Governance Context | tenant identity, user class, authorization scope, privacy tags, compliance tags, provider restrictions        | Constrain what the caller is allowed to use                |
| Session Context                 | `session_id`, previous route class or target pool, continuity preference, escalation history                  | Preserve continuity across multiple turns when appropriate |

#### Model Card

A model card is the route-visible model-family definition used by semantic routing to derive a pool decision.

Each routable semantic entry or model family SHOULD expose an identifier, capability class, target-pool mapping, routing tags, supported capabilities and modalities, context-window limits, optional quality/latency/cost attributes, optional reasoning or LoRA variants, and any governance or tenant restrictions relevant to route selection.

A semantic entry or model family is not a concrete backend endpoint. A target pool MAY map to multiple provider models and downstream replicas, while endpoint health and queue metrics remain outside the ownership of model cards.

#### Decision Rule

A decision rule is the semantic route rule defined in `routing.decisions`.

Each decision rule SHOULD contain a name, priority, typed conditions with logical composition, a candidate `modelRefs` set, a selection algorithm, and optional plugins. Decision rules define the legal route space for a request; they do not select the final backend endpoint.

#### Major Interface Objects

The module boundary consists of four major objects:

| Object            | Owned by          | Major fields                                                                                                           | Purpose                                    |
| ----------------- | ----------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Request           | Client or gateway | prompt, messages, metadata, identity, session                                                                          | Original request entering semantic routing |
| RouteDecision     | Semantic Router   | `route_class`, `target_pool`, `model_family`, `safety_profile`, `cache_policy`, `routing_confidence`, `fallback_pools` | Formal SR to LB handoff contract           |
| SchedulingContext | Load Balancer     | pool members, health, load, latency, locality, admission status                                                        | Runtime scheduling state owned by LB only  |
| DispatchResult    | Load Balancer     | selected endpoint, replica, region, dispatch reason                                                                    | Final execution result after scheduling    |

#### Signal Set

A signal is a typed routing feature extracted from the request context.

Depending on deployment requirements, ASE Semantic Router MAY use keyword, embedding, domain, language, complexity, context, modality, preference, authorization, jailbreak, and PII signals. Not every deployment requires every signal family, and not every request requires every extractor to run.

#### Session Continuity Metadata

Session continuity metadata MAY include `session_id`, the previous route class or target pool, the last escalation reason, a continuity preference, and conversation history. It is an optimization input rather than a hard override and MUST NOT bypass hard capability or policy constraints.

### Request Normalization

The request normalizer converts inbound OpenAI-compatible API traffic into a canonical routing object consumed by later stages.

ASE Semantic Router SHOULD support the following semantic-routing-aware controls:

| Field                    | Purpose                                                                         | Constraint                                                                            |
| ------------------------ | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `model=auto`             | Request semantic route selection                                                | Default path for routed traffic                                                       |
| `model=<explicit-model>` | Request a specific semantic entry model or model family directly                | Still subject to capability and policy validation, then mapped to a legal target pool |
| `routing_hint`           | Provide a coarse semantic hint such as `code`, `reasoning`, `extract`, `vision` | Advisory only; MUST NOT bypass policy                                                 |
| `route_override`         | Request a specific capability path or target-pool alias                         | Restricted to authorized callers                                                      |
| `preference`             | Express latency, cost or quality bias                                           | Optimization input only                                                               |
| `input_tokens_estimate`  | Provide a caller-side prompt-size estimate                                      | Advisory signal only                                                                  |
| `session_id`             | Preserve multi-turn continuity context                                          | Optional unless continuity policy requires it                                         |
| `debug` or `explain`     | Request routing diagnostics                                                     | Restricted and redacted for trusted callers only                                      |

The precedence order MUST be explicit. Hard capability and policy constraints are evaluated first, authorized explicit model requests or route overrides second, and continuity or optimization preferences only after eligibility is established. ASE SHOULD route at request granularity rather than pinning an entire session to one pool.

### Signal Extraction Layer

The signal extraction layer SHOULD compute cheap signals first, invoke expensive extractors only when they materially affect the decision, keep outputs explicit and typed, and avoid hidden heuristic logic in the request path. Supporting modules MAY include an embedding service, classifier service, token estimator, jailbreak detector, PII detector, tool catalog, and semantic cache. These services are subordinate to the routing pipeline; the explicit signal set remains the source of truth for semantic decisions.

Within `routing.decisions`, `modelRefs` remains the canonical upstream configuration term. In ASE split mode, `modelRefs` is an input to semantic route selection, while `RouteDecision.target_pool` is the primary output consumed by ASE LLM Load Balancer.

### Hard Constraint and Policy Filter

Before any optimization among candidate pools or model families, the request MUST pass hard capability and policy checks.

The filter evaluates request validity, explicit-override authorization, capability and modality matching, context-length and token-limit constraints, tenant restrictions, provider allowlists or denylists, privacy and compliance tags, jailbreak policy, and PII-sensitive routing restrictions.

The hard-filter stage is eliminative rather than score-based. A candidate that fails any hard constraint MUST be removed from the legal route space before optimization, preference handling, or plugin execution begins.

```mermaid
flowchart LR
    A[Evaluate Candidate Pool or Model Family] --> B{Capabilities and Modalities Compatible?}
    B -- No --> R[Reject Candidate]

    B -- Yes --> C{Context and Token Limits Satisfied?}
    C -- No --> R

    C -- Yes --> D{Policy and Governance Allowed?}
    D -- No --> R
    D -- Yes --> P[Mark Candidate Eligible]
```

### Decision Engine and Pool Selection

The route-decision algorithm MUST use normalized request context, extracted signals, configured decision rules and candidate pools or `modelRefs`, logical model capabilities and pool mappings, and any applicable continuity or caller preferences. Unlike ASE LLM Load Balancer, semantic routing MUST NOT use backend queue depth, endpoint health, or connection-pool runtime state to choose the target pool.

The decision engine performs constrained selection over the legal candidate set that survives hard filtering. Its purpose is to explain how ASE chooses one semantic target pool or model family, not to expose internal helper-function structure.

```mermaid
flowchart LR
    A[Normalized Request Context] --> I[Assemble Decision Inputs]
    B[Extracted Signal Set] --> I
    C[Matched Routing Rule] --> I
    D[Eligible Candidate Pools or modelRefs] --> I
    E[Caller and Session Preferences] --> I

    I --> S[Score and Rank Legal Candidates]
    S --> T[Apply Continuity and Tie-Break Rules]
    T --> G{Target Pool Selected?}
    G -- No --> X[Return No Route Error]
    G -- Yes --> R[Emit RouteDecision]
```

In this flow, the hard-filter stage has already removed infeasible candidates. The decision engine then combines normalized request context, extracted signals, the matched `routing.decisions` rule, the remaining legal candidate pools or `modelRefs`, and any applicable caller or session preferences into a bounded ranking problem. Continuity handling and tie-break rules MAY influence which legal candidate is selected, but they MUST operate only within the feasible route space.

Supported route-selection strategies MAY include static priority, quality-first selection, cost-aware selection, latency-aware selection based on model-level attributes, and hybrid policy-aware ranking.

For specification and design review, the route decision SHOULD be modeled as constrained selection over legal candidate pools rather than as a chain of implementation helper functions.

Let `r` denote the normalized request context and let `P = {p1, p2, ..., pn}` denote the candidate serving pools derived from the matched routing rule. For each request-pool pair `(r, p)`, the router evaluates two hard-feasibility predicates:

- `capability_feasible(r, p)`: true only when modality, context-window, and required capability constraints are satisfied.
- `policy_feasible(r, p)`: true only when tenant, compliance, privacy, and abuse-policy constraints are satisfied.

The legal candidate set is therefore:

$$
F(r) = \{\, p \in P \mid \mathrm{capability\_feasible}(r, p) \wedge \mathrm{policy\_feasible}(r, p) \,\}
$$

For each `p in F(r)`, the router computes a bounded utility score:

$$
U(r, p) = w_1 S_{\mathrm{sem}}(r, p) + w_2 S_{\mathrm{cont}}(r, p) + w_3 S_{\mathrm{pref}}(r, p) + w_4 S_{\mathrm{pool}}(r, p)
$$

$$
S_{\mathrm{sem}}(r, p) = a_1 s_{\mathrm{keyword}}(r, p) + a_2 s_{\mathrm{embedding}}(r, p) + a_3 s_{\mathrm{domain}}(r, p) + a_4 s_{\mathrm{complexity}}(r, p)
$$

$$
S_{\mathrm{cont}}(r, p) \in \{0, 1\}
$$

`S_cont(r, p)` is `1` when a valid previous session exists and `p` remains valid; otherwise it is `0`.

$$
S_{\mathrm{pref}}(r, p) = b_1 s_{\mathrm{latency}}(r, p) + b_2 s_{\mathrm{cost}}(r, p) + b_3 s_{\mathrm{quality}}(r, p)
$$

The selected pool is the maximizer over the legal candidate set:

$$
p^{*} = \arg \max_{p \in F(r)} U(r, p)
$$

If `F(r)` is empty, the router MUST return a semantic rejection or an explicitly configured fallback outcome. Once `p*` is selected, the router constructs a `RouteDecision` containing the chosen `route_class`, `target_pool`, `model_family`, and `safety_profile`.

A non-normative reference procedure is shown below:

| Field     | Description                                                                                 |
| --------- | ------------------------------------------------------------------------------------------- |
| Algorithm | Constraint-Aware Route Selection                                                            |
| Inputs    | `r`: normalized request context; `P`: candidate pools derived from the matched routing rule |
| Output    | `RouteDecision`, or rejection/fallback outcome                                              |

```text
Procedure:
1. F <- {}
2. for each p in P do
3.     if not capability_feasible(r, p) then
4.         continue
5.     end if
6.     if not policy_feasible(r, p) then
7.         continue
8.     end if
9.     score[p] <- U(r, p)
10.    F <- F union {p}
11. end for
12. if F = {} then
13.    return reject_or_fallback(r)
14. end if
15. p* <- arg max score[p] over p in F
16. return build_route_decision(r, p*)
```

This formulation makes the module boundary explicit: hard capability and policy constraints define the feasible set first; semantic, continuity, and preference signals optimize only within that legal set; and the output of the module is a `RouteDecision` rather than a backend endpoint selection. Semantic Router selects the capability pool; Load Balancer selects the concrete serving replica.

### Plugin Chain and Handoff Contract

After route selection, the module MAY execute per-decision plugins such as safety tagging, audit annotation, semantic-cache hooks, prompt rewrite, tracing, or retrieval augmentation.

The module then emits a formal `RouteDecision` plus any compatibility fields required by downstream execution. This object is the handoff artifact to ASE LLM Load Balancer.

| Field                             | Requirement level | Purpose                                                                    |
| --------------------------------- | ----------------- | -------------------------------------------------------------------------- |
| `route_class`                     | Required          | Capability path chosen by semantic routing                                 |
| `target_pool`                     | Required          | Primary dispatch contract consumed by ASE LLM load balancer                |
| `model_family`                    | Optional          | Preferred model family inside the selected pool                            |
| `latency_tier`                    | Optional          | Scheduling hint for latency class                                          |
| `cost_tier`                       | Optional          | Scheduling hint for cost class                                             |
| `safety_profile`                  | Optional          | Required safety posture for downstream handling                            |
| `cache_policy`                    | Optional          | Cache and reuse policy hint                                                |
| `routing_confidence`              | Optional          | Confidence of the semantic decision                                        |
| `fallback_pools`                  | Optional          | Explicit cross-pool fallback policy allowed by semantic or gateway policy  |
| `request_id`                      | Required          | Stable request identity across routing, dispatch and observability         |
| `route_decision_status`           | Required          | Distinguish successful routing from semantic rejection                     |
| `matched_decision`                | Optional          | Identify which semantic decision rule matched                              |
| `route_reason`                    | Optional          | Preserve operator-readable routing rationale                               |
| `policy_tags`                     | Optional          | Carry governance annotations that may matter downstream                    |
| `debug_trace_id`                  | Optional          | Correlate routing decisions with trace and logs                            |
| `continuity_metadata`             | Optional          | Preserve session-related context                                           |
| `model` or projected route header | Optional          | Compatibility field only; not the sole dispatch contract in ASE split mode |

At a minimum, every emitted `RouteDecision` MUST include `route_class`, `target_pool`, `request_id`, and `route_decision_status`. Optional fields MAY be omitted when not applicable or when withheld by policy.

At this boundary, `target_pool` is the primary dispatch contract. `model_family` or a normalized `model` value are compatibility hints only. ASE LLM Load Balancer MUST schedule within the selected pool and MUST NOT reinterpret prompt semantics.

An example `RouteDecision` object is shown below.

```json
{
  "route_class": "reasoning",
  "target_pool": "reasoning_pool",
  "model_family": "qwen3-32b",
  "latency_tier": "standard",
  "cost_tier": "medium",
  "safety_profile": "default",
  "cache_policy": "allow",
  "routing_confidence": 0.91,
  "fallback_pools": ["general_large_pool", "review_pool"],
  "request_id": "req-123456",
  "route_decision_status": "ok",
  "matched_decision": "computer_science_reasoning",
  "route_reason": "domain=code;complexity=high;policy=allowed",
  "policy_tags": ["tenant:default", "privacy:standard"]
}
```

### Interaction with ASE LLM Load Balancer

The interaction with the downstream load-balancing module is intentionally narrow. ASE Semantic Router MUST emit a `RouteDecision` whose primary dispatch contract is `target_pool`. ASE LLM Load Balancer MUST consume that pool directly and perform instance-level scheduling only within the declared pool. Policy tags and route metadata MAY constrain dispatch behavior, but they MUST NOT reopen semantic route selection during normal operation.

Two deployment modes MAY be supported. In upstream-compatible integrated mode, the service MAY project route headers or destination hints for gateway integration. In ASE split mode, the semantic router emits `RouteDecision` plus routing metadata and delegates final endpoint selection to ASE LLM Load Balancer. ASE split mode is preferred.

Fallback behavior MUST preserve the same boundary. Infrastructure fallback keeps the target pool fixed while ASE LLM Load Balancer switches to another healthy replica within that pool. Cross-pool fallback is allowed only when semantic routing or gateway policy declares it explicitly, for example through `fallback_pools`.

### Semantic Failure Classes

| Failure class                   | Meaning                                                                                      | Typical cause                                                                      |
| ------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| No Matching Decision            | No configured semantic route matched the request signal set                                  | Missing fallback route, unsupported workload shape, insufficient signal confidence |
| No Eligible Pool                | No legal capability pool or model family satisfies hard capability or deployment constraints | Missing modality support, insufficient context window, no legal pool mapping       |
| Policy Denial                   | One or more pools or model families are technically capable, but all are forbidden by policy | Tenant restriction, provider denylist, privacy or compliance rule                  |
| Invalid Routing Request         | The request is malformed or missing required routing context                                 | Malformed payload, unsupported request shape, invalid override                     |
| Decision Engine Failure         | The module failed unexpectedly during routing                                                | Internal evaluation failure, signal extraction failure, plugin error               |
| Deferred Infrastructure Failure | Semantic routing succeeded, but downstream execution later failed                            | Endpoint unavailable, dispatch failure, retry exhaustion in ASE LLM Load Balancer  |

# Management and Discovery APIs

Please refer to the corresponding section of [ASE semantic load balancer](./ase_semantic_load_balancer.md).

# Operational Debuggability

Please refer to the corresponding section of [ASE semantic load balancer](./ase_semantic_load_balancer.md).

# Configuration

This section illustrates the major configuration surfaces required by ASE semantic router. Field names are illustrative; an implementation MAY use different names provided that it preserves equivalent semantics.

```yaml
version: v0.3

listeners:
  - name: http-8899
    address: 0.0.0.0
    port: 8899
    timeout: 300s

providers:
  defaults:
    default_model: general-small
    default_reasoning_effort: medium
  models:
    - name: general-small
      provider_model_id: general-small
      backend_refs:
        - name: primary
          endpoint: llm-gateway.internal:8000
          protocol: http2
    - name: code-large
      provider_model_id: code-large
      backend_refs:
        - name: primary
          endpoint: code-gateway.internal:8000
          protocol: http2

routing:
  modelCards:
    - name: general-small
      description: default text assistant
      capability_class: chat
      target_pool: general_fast_pool
      modality: text
      capabilities: [chat, tools]
      context_length: 32768
      quality: medium
      latency: low
      cost: low
      governance_tags: [tenant:default, privacy:standard]
    - name: code-large
      description: code and design reasoning model
      capability_class: reasoning_code
      target_pool: code_reasoning_pool
      modality: text
      capabilities: [chat, reasoning, long-context]
      context_length: 131072
      quality: high
      latency: medium
      cost: medium
      governance_tags: [tenant:default, privacy:standard]
      loras:
        - name: code-review-adapter
          description: adapter for code review and design prompts
  signals:
    keywords:
      - name: code_terms
        operator: OR
        keywords: ["code", "api", "debug", "refactor"]
    complexity:
      - name: needs_reasoning
        threshold: 0.75
        description: multi-step synthesis or design-heavy prompts
    pii:
      - name: sensitive_data
        enabled: true
    jailbreak:
      - name: abuse_guard
        enabled: true
  decisions:
    - name: computer_science_reasoning
      description: route software engineering requests to reasoning-capable models
      priority: 170
      rules:
        operator: AND
        conditions:
          - type: keyword
            name: code_terms
          - type: complexity
            name: needs_reasoning
      modelRefs:
        - model: general-small
          use_reasoning: false
          weight: 0.2
        - model: code-large
          use_reasoning: true
          reasoning_effort: high
          lora_name: code-review-adapter
          weight: 0.8
      route_output:
        route_class: reasoning
        target_pool: code_reasoning_pool
        model_family: qwen3-32b
        latency_tier: standard
        cost_tier: medium
        safety_profile: default
        cache_policy: allow
        fallback_pools: [general_large_pool, review_pool]
      algorithm:
        type: hybrid_policy_aware
      plugins:
        - type: audit
          configuration:
            enabled: true
        - type: system_prompt
          configuration:
            enabled: true
            mode: insert
            system_prompt: You are a senior software architect.

global:
  router:
    config_source: file
```

# References

[1] vLLM Semantic Router documentation https://vllm-semantic-router.com/docs/intro/

[2] vLLM Semantic Router configuration documentation https://vllm-semantic-router.com/docs/installation/configuration/

[3] vLLM Semantic Router system architecture documentation https://vllm-semantic-router.com/docs/overview/architecture/system-architecture/

[4] vLLM Semantic Router Envoy ExtProc integration documentation https://vllm-semantic-router.com/docs/overview/architecture/envoy-extproc

[5] vLLM Semantic Router GitHub repository https://github.com/vllm-project/semantic-router
