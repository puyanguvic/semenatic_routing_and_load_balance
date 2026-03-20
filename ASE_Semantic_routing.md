# ASE Semantic Routing Design

## Document Status

* **Document ID**: ASE-LLM-SEMANTIC-ROUTING
* **Document Type**: Subsystem Design
* **Intended Audience**: Platform architects, gateway engineers, AI infrastructure engineers, security engineers
* **Scope Level**: Detailed subsystem design
* **Related Documents**:

  * `ASE LLM Gateway Architecture Overview`
  * `ASE Load Balancing Design`

---

## 1. Introduction

This document specifies the design of the **Semantic Routing** subsystem in the ASE LLM gateway.

The role of Semantic Routing is to determine **which model should serve a request**. It operates before infrastructure-level dispatch and is responsible for converting an incoming request into a policy-compliant, explainable, and operationally useful **model selection decision**. Its output is a request enriched with an authoritative `model` field and optional routing metadata.

This design is informed by the signal-driven routing architecture proposed in **vLLM Semantic Router**, which treats routing as a structured decision problem built from heterogeneous signals, Boolean decision logic, and per-decision plugin execution. That paper presents a three-layer architecture—**signal extraction**, **decision evaluation**, and **plugin chains**—and explicitly positions semantic routing as a request-level **Mixture-of-Models** problem rather than a server-level scheduling problem. ([arXiv][1])

In ASE, we adopt that design direction for the model-selection layer, while keeping backend instance scheduling explicitly out of scope for this document.

---

## 2. Scope

### 2.1 In Scope

This document covers:

* semantic model selection
* routing signal extraction
* model eligibility filtering
* policy-aware decision evaluation
* request enrichment with resolved model metadata
* routing explainability and debug outputs
* session-level routing continuity
* semantic failure handling
* routing-related observability

### 2.2 Out of Scope

This document does **not** define:

* backend instance scheduling
* endpoint health checks
* retry policies
* failover across serving instances
* pool-level traffic distribution
* load-balancer algorithm selection

Those concerns belong to `ASE Load Balancing Design`.

---

## 3. Problem Definition

Semantic Routing solves the following problem:

> Given an incoming LLM request, a heterogeneous set of candidate models, and a set of deployment, policy, and business constraints, determine the most appropriate target model for the request.

This is not equivalent to simple prompt classification. A production semantic router must simultaneously reason about:

* task and domain
* model capability fit
* context-window requirements
* cost and latency preference
* privacy and safety constraints
* deployment restrictions
* multi-turn session consistency

The vLLM Semantic Router paper makes this explicit. It argues that production routing must consider multi-dimensional signals including domain, modality, complexity, language, identity, privacy/safety, deployment diversity, and multi-turn consistency, rather than only “difficulty routing” between a small number of models. ([arXiv][2])

---

## 4. Design Goals

The Semantic Routing subsystem is designed to satisfy the following goals.

### 4.1 Correctness of Model Choice

The subsystem should select a model that is semantically appropriate and policy-compliant for the request.

### 4.2 Explicit Separation from Load Balancing

The subsystem should determine the **model**, not the **serving instance**.

### 4.3 Policy-First Decision Making

Governance constraints should be enforced before expensive model invocation.

### 4.4 Explainability

The subsystem should be able to explain why a model was selected or why no model was eligible.

### 4.5 Extensibility

New signals, new policies, and new model families should be introducible without redesigning the subsystem.

### 4.6 Low Decision Latency

Semantic routing should add bounded overhead and should support selective signal computation so that expensive signals are evaluated only when needed. The vLLM Semantic Router paper emphasizes heterogeneous signal extraction, including both sub-millisecond heuristic signals and more expensive learned classifiers, which is a useful template for ASE. ([arXiv][2])

---

## 5. Non-Goals

The subsystem is not intended to:

* optimize GPU utilization directly
* monitor per-instance queue depth
* retry failed upstream requests
* rebalance across backend replicas
* replace load balancing
* implement generic API gateway concerns unrelated to model selection

---

## 6. Architectural Position

Within the ASE gateway, Semantic Routing is the **first decision layer** in the request path.

**Client Request -> Semantic Routing -> Request Enrichment (`model=...`) -> Load Balancing**

The contract is strict:

* Semantic Routing resolves the target model.
* Load Balancing receives that resolved model and schedules only within the corresponding backend pool.

This separation is central to the overall ASE architecture.

---

## 7. Design Principles

### 7.1 Model-Centric, Not Server-Centric

Semantic Routing is responsible for model selection only.

### 7.2 Constraint-Then-Optimization

The subsystem should first apply hard eligibility and policy constraints, then optimize among remaining candidates.

### 7.3 Composable Signal Orchestration

Routing should be built from multiple heterogeneous signals, not a single monolithic classifier. This is the central architectural idea in vLLM Semantic Router, which describes “composable signal orchestration” as the core contribution of the system. ([arXiv][1])

### 7.4 Configuration-Driven Policy

Routing policy should be configurable rather than hardcoded.

### 7.5 Explainable Outputs

Each final routing decision should have a recoverable reason trail.

### 7.6 Safety Before Invocation

Safety, privacy, and governance controls should be evaluated before the request reaches a model backend.

---

## 8. Internal Architecture

The Semantic Routing subsystem is composed of five logical components.

### 8.1 Request Normalizer

Transforms inbound API requests into a canonical internal routing representation.

Responsibilities:

* normalize request shape
* extract control parameters
* extract session and tenant metadata
* canonicalize message content for downstream signal extraction

### 8.2 Signal Extraction Engine

Produces a structured routing context from the request.

Responsibilities:

* heuristic signal extraction
* ML-based signal extraction
* optional cached feature reuse
* selective signal computation

The vLLM Semantic Router paper describes a signal-extraction layer spanning heuristic signals such as keyword patterns, language detection, context length, and authorization, as well as neural signals such as domain classification, embedding similarity, factual grounding, modality, and safety-related detection. ([arXiv][1])

### 8.3 Eligibility and Constraint Filter

Removes models that are impossible or disallowed for the request.

Responsibilities:

* context window filtering
* modality support filtering
* policy boundary filtering
* tenant policy filtering
* compliance-zone filtering

### 8.4 Decision Engine

Resolves a target model from the eligible candidate set.

Responsibilities:

* apply routing policy
* score or rank candidates
* select final model
* produce routing rationale

The paper’s decision layer is Boolean and policy-driven rather than a single opaque predictor, which is a useful design template for ASE. ([arXiv][2])

### 8.5 Policy Plugin Chain

Executes per-decision controls associated with routing outcomes.

Responsibilities:

* safety gating
* PII handling
* audit tagging
* request annotation
* optional semantic cache lookups
* optional retrieval or augmentation triggers

vLLM Semantic Router explicitly models per-decision plugin chains as part of the routing architecture and uses them for safety, caching, and augmentation behaviors. ([arXiv][2])

---

## 9. Routing Inputs

Semantic Routing consumes four classes of input.

### 9.1 Request Content

Examples:

* messages
* prompt text
* system prompt
* multimodal payload metadata
* requested output structure

### 9.2 Request Control Metadata

Examples:

* requested model or `model=auto`
* user preference hints
* debug flags
* route override requests
* expected response style or structured-output hints

### 9.3 Identity and Governance Context

Examples:

* tenant identifier
* caller identity
* authorization context
* compliance region
* internal-only / external-allowed policy

### 9.4 Session Context

Examples:

* session ID
* conversation ID
* prior selected model
* continuity requirement
* routing memory tags

The vLLM Semantic Router paper states that the system supports stateful multi-turn conversations and conversation-consistent model assignment through OpenAI-style APIs, which supports including session continuity as a first-class routing input. ([arXiv][1])

---

## 10. Routing Signals

ASE should classify routing signals into five groups.

### 10.1 Semantic Signals

Capture what the request is about.

Examples:

* domain: code, chat, reasoning, extraction, summarization
* modality: text, code, image-enabled, multimodal
* intent: generation, transformation, analysis, structured extraction

### 10.2 Complexity Signals

Capture how hard the request appears to be.

Examples:

* shallow factual
* moderate analytical
* deep reasoning
* long-form synthesis

### 10.3 Capability-Requirement Signals

Capture what the model must support.

Examples:

* long-context requirement
* tool-calling requirement
* structured JSON output requirement
* multimodal support requirement

### 10.4 Safety and Policy Signals

Capture what controls apply.

Examples:

* PII presence
* jailbreak risk
* hallucination-sensitive request class
* regulated-domain indicator
* tenant restriction

The paper describes privacy, safety, PII, jailbreak handling, and a gated hallucination-detection pipeline as part of the broader routing-time control framework. ([arXiv][1])

### 10.5 Preference Signals

Capture business or caller objectives.

Examples:

* prefer low cost
* prefer low latency
* prefer high quality
* prefer private deployment
* prefer continuity with prior model

---

## 11. Candidate Model Registry

The Semantic Routing layer requires a **model capability registry** that describes the routable model universe.

Each model entry should define at least:

* logical model ID
* model family
* supported modalities
* maximum context window
* structured-output support
* reasoning tier
* deployment boundary
* compliance tags
* cost tier
* latency tier
* availability policy

This registry is the authoritative source for model eligibility filtering.

---

## 12. Routing Pipeline

ASE Semantic Routing should execute the following pipeline.

### Step 1: Normalize Request

Convert the inbound request to the canonical internal format.

### Step 2: Build Routing Context

Collect content, metadata, session, and policy context into a single routing object.

### Step 3: Extract Signals

Run heuristic and ML-based signal extraction to produce a structured signal vector.

### Step 4: Filter Ineligible Models

Remove models that cannot satisfy hard requirements.

Examples:

* insufficient context window
* unsupported modality
* prohibited deployment boundary
* disallowed by tenant policy

### Step 5: Apply Policy Gates

Apply policy rules that may reject the request, constrain the candidate set, or attach required plugins.

### Step 6: Optimize Among Eligible Candidates

Choose the target model according to configured objective.

Possible objectives:

* minimum cost subject to capability and quality floor
* minimum latency subject to policy and quality floor
* maximum quality within budget envelope
* continuity-preserving choice under session constraints

### Step 7: Execute Routing Plugins

Run per-decision plugins such as annotation, redaction, audit tagging, or semantic cache lookup.

### Step 8: Enrich Request

Attach the authoritative `model` field and optional routing metadata.

### Step 9: Emit Routing Trace

Record reason codes and decision artifacts for debugging and auditability.

---

## 13. Decision Model

A professional semantic router should distinguish between three classes of logic.

### 13.1 Hard Constraints

These eliminate models from consideration.

Examples:

* unsupported modality
* insufficient context window
* deployment prohibition
* compliance mismatch
* missing required feature support

### 13.2 Policy Constraints

These enforce organizational rules.

Examples:

* regulated traffic must stay in private deployment
* route override allowed only for privileged callers
* certain tenants may not use certain providers
* unsafe prompts may require guarded execution path

### 13.3 Optimization Criteria

These select among the remaining candidates.

Examples:

* cheapest acceptable model
* fastest acceptable model
* best-quality model within budget
* continuity-preserving model for an active session

This separation is much cleaner than a single flat classifier.

---

## 14. Session and Multi-Turn Semantics

Semantic Routing must support multi-turn behavior.

### 14.1 Session Continuity Goal

Where appropriate, consecutive turns in the same session should preserve model continuity unless there is a strong reason to change the model.

### 14.2 Allowed Escalation

The subsystem may escalate a session from a smaller model to a stronger model if:

* complexity increases materially
* modality changes
* required capabilities change
* policy constraints change

### 14.3 Downgrade Policy

A downgrade to a weaker model should be conservative and preferably occur only at a clean request boundary.

### 14.4 Session Metadata

The subsystem should accept at least:

* `session_id`
* `conversation_id`
* `previous_model`
* continuity hint

The vLLM Semantic Router paper explicitly treats multi-turn statefulness and conversation-consistent model assignment as part of the routing system rather than an external concern. ([arXiv][2])

---

## 15. Output Contract

The Semantic Routing subsystem produces a **routing result**.

### 15.1 Required Output

* `model`: resolved target model identifier

### 15.2 Optional Output

* `route_reason`
* `route_policy`
* `session_id`
* `tenant_id`
* `priority`
* `policy_tags`
* `debug_trace_id`

### 15.3 Example

```json
{
  "model": "code-large",
  "route_reason": "domain=code;complexity=high;policy=default",
  "policy_tags": ["internal-ok", "safe"],
  "messages": [
    {
      "role": "user",
      "content": "Write a C epoll example with edge-triggered I/O."
    }
  ]
}
```

This output is the contract passed to `ASE Load Balancing Design`.

---

## 16. Explainability and Debuggability

Semantic Routing must be explainable.

### 16.1 Required Debug Artifacts

The subsystem should be able to reconstruct:

* extracted signals
* candidate model set before filtering
* filtered candidate set after constraints
* applied policy rules
* selected optimization objective
* final model choice

### 16.2 Reason Codes

Every successful route should emit a concise reason code.

Examples:

* `code_high_complexity`
* `long_context_private_only`
* `structured_output_low_cost`
* `policy_denied_external_provider`

### 16.3 Debug Mode

A debug mode may return routing diagnostics in a side channel or response extension, but should not expose sensitive internal policy details by default.

The emphasis on structured decision composition in vLLM Semantic Router makes this style of explanation natural, because the architecture is rule- and signal-based rather than a single opaque classifier. ([arXiv][2])

---

## 17. Failure Semantics

Semantic failures should be distinguished from infrastructure failures.

### 17.1 No Eligible Model

Occurs when no model satisfies hard capability constraints.

### 17.2 Policy Denial

Occurs when the request is disallowed by governance or safety policy.

### 17.3 Invalid Routing Request

Occurs when the request is malformed or missing required routing inputs.

### 17.4 Decision Engine Failure

Occurs when internal routing execution fails unexpectedly.

### 17.5 Deferred Infrastructure Failure

If a valid model is selected but no serving instance is later available, that is **not** a Semantic Routing failure. It belongs to Load Balancing.

This distinction should be preserved in logs, metrics, and error responses.

---

## 18. Observability

The Semantic Routing subsystem should emit observability signals at the routing layer.

### 18.1 Core Metrics

* routing requests total
* routing success total
* routing denial total
* no-eligible-model total
* route distribution by model
* route distribution by tenant
* route distribution by policy
* routing decision latency

### 18.2 Optional Metrics

* signal extraction latency by signal type
* percentage of requests requiring expensive ML signals
* escalation rate across model tiers
* session model-switch rate

### 18.3 Audit Log Fields

Recommended audit fields:

* request ID
* tenant ID
* session ID
* selected model
* route reason
* policy tags
* failure code if any

---

## 19. Configuration Model

The subsystem should be configured through declarative policy rather than code changes.

### 19.1 Configuration Domains

* model registry
* signal extractors
* policy rules
* optimization objective
* session continuity policy
* plugin chain bindings
* debug verbosity

### 19.2 Example Logical Structure

```yaml
semantic_routing:
  default_mode: auto
  objectives:
    default: quality_within_budget
  session_policy:
    preserve_previous_model: true
    allow_escalation: true
    allow_downgrade: conservative
  models:
    - id: general-small
      family: general
      cost_tier: low
      latency_tier: low
      context_window: 32000
    - id: code-large
      family: code
      cost_tier: high
      latency_tier: medium
      context_window: 128000
  policies:
    - name: code_high_complexity
      when:
        domain: code
        complexity: high
      select: code-large
    - name: long_context_private
      when:
        context_requirement: long
        policy_tag: private_only
      allowed_models: [reasoning-private-long]
```

The vLLM Semantic Router paper goes further and describes a typed neural-symbolic DSL compiled to multiple deployment targets, with configuration-first adaptation across deployment scenarios. ASE does not need to copy that exact mechanism, but the underlying principle—**configuration-first routing policy**—is highly applicable. ([arXiv][2])

---

## 20. Security and Governance Considerations

Semantic Routing is one of the earliest policy enforcement points in the LLM path and therefore must be treated as a governance boundary.

It should support at least:

* authorization-aware model restrictions
* deployment-boundary restrictions
* PII-sensitive routing
* jailbreak-sensitive routing
* tenant-specific provider restrictions
* audit tagging for regulated traffic

The paper explicitly integrates authorization, privacy, safety, PII, and hallucination-related controls into the routing framework, reinforcing that this layer is not merely about convenience but about policy-safe model selection. ([arXiv][1])

---

## 21. Summary

ASE Semantic Routing is the subsystem responsible for **request-level model selection**.

It operates as the first decision layer in the ASE gateway and is responsible for:

* extracting routing-relevant signals,
* filtering models by capability and policy,
* selecting the most appropriate eligible model,
* enforcing pre-invocation policy controls,
* enriching the request with an authoritative `model` field,
* and producing explainable routing outcomes.

This subsystem is inspired by the signal-driven, composable architecture of **vLLM Semantic Router**, especially its use of heterogeneous signal extraction, Boolean decision evaluation, per-decision plugin chains, and multi-turn routing support. ASE adapts those principles into an enterprise gateway setting with a strict handoff to the downstream Load Balancing layer. ([arXiv][1])

The next document in the series, `ASE Load Balancing Design`, specifies how ASE schedules requests across serving instances **after** the Semantic Routing layer has resolved the target model.

[1]: https://arxiv.org/abs/2603.04444?utm_source=chatgpt.com "vLLM Semantic Router: Signal Driven Decision Routing for Mixture-of-Modality Models"
[2]: https://arxiv.org/pdf/2603.04444 "vLLM Semantic Router: Signal Driven Decision Routing for Mixture-of-Modality Models"
