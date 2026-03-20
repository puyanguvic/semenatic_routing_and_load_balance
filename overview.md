# ASE LLM Gateway Architecture Overview

## Document Status

- **Document ID**: ASE-LLM-ARCH-OVERVIEW
- **Document Type**: Architecture Overview
- **Intended Audience**: Platform architects, AI infrastructure engineers, gateway engineers, security engineers, operations teams
- **Scope Level**: High-level system architecture
- **Related Documents**:
  - `ASE Semantic Routing Design`
  - `ASE Load Balancing Design`

---

## 1. Introduction

Enterprise LLM access can no longer be treated as a simple proxying problem. Modern AI applications invoke multiple models with different capability, latency, cost, deployment boundary, and compliance characteristics. As a result, the gateway in front of these models must make two fundamentally different decisions for every request.

The first decision is **semantic model selection**: determining which model is most appropriate for the request based on task intent, complexity, context requirements, policy constraints, and user preferences. The second decision is **infrastructure-level dispatch**: determining which backend instance should execute the request once the target model has already been selected.

To address this problem, ASE adopts a **two-layer request processing architecture**:

1. **Semantic Routing Layer**  
   Resolves the target model for the request.

2. **Load Balancing Layer**  
   Resolves the serving instance within the selected model pool.

This document describes the overall architecture of the ASE LLM gateway, the system boundaries, the request lifecycle, and the responsibility split between these two layers. Detailed designs of each layer are intentionally described in separate subsystem documents.

---

## 2. Purpose and Scope

This document provides the **high-level architecture** of the ASE LLM gateway. It focuses on:

- the architectural motivation for the gateway
- the decomposition of the system into two layers
- the contract between the two layers
- the major components in the end-to-end request path
- the overall processing lifecycle for an LLM request
- the relationship between this overview document and the subsystem design documents

This document does **not** define:

- detailed semantic signal extraction logic
- detailed routing policy rules
- detailed instance scheduling algorithms
- detailed health-check, retry, or failover parameters
- detailed request/response API fields
- detailed configuration schema

Those topics belong in the subsystem documents.

---

## 3. Problem Statement

An enterprise LLM gateway must solve a routing problem that is more complex than traditional HTTP traffic distribution.

In a conventional load balancer, the primary question is: **which backend server should receive the request**. In an LLM system, that is only half of the problem. A production gateway must answer two questions in sequence:

1. **Which model should serve this request?**
2. **Which serving instance should execute that model request?**

These questions should not be collapsed into a single opaque decision because they operate on different inputs, different constraints, and different optimization objectives.

### 3.1 Why a Single-Layer Router Is Not Enough

A single-layer “smart router” quickly becomes difficult to reason about because it mixes:

- semantic understanding of prompt intent
- policy and governance enforcement
- model capability matching
- cost and quality trade-offs
- backend availability
- queue and connection pressure
- retry and failover behavior

This creates poor separation of concerns. It also makes the system harder to explain, harder to operate, and harder to evolve.

### 3.2 Architectural Requirement

ASE therefore requires an architecture that:

- separates **model selection** from **instance scheduling**
- preserves a clean control boundary between policy logic and traffic engineering
- supports heterogeneous model fleets
- supports heterogeneous serving backends
- supports enterprise governance and observability
- remains operationally understandable under failure and scale

---

## 4. Design Goals

The ASE LLM gateway is designed to satisfy the following goals.

### 4.1 Correct Model Selection

The system should route each request to a model that is appropriate for the task, capability requirement, policy boundary, and business objective.

### 4.2 Efficient Instance Scheduling

Once a model is selected, the system should forward the request to a backend instance that can serve it with good latency, availability, and operational stability.

### 4.3 Policy and Governance Enforcement

The gateway should provide a control point for enterprise requirements such as authorization, tenant restrictions, compliance boundaries, and safety-related routing controls.

### 4.4 Operational Reliability

The system should remain resilient under backend failures, overload, degraded service, and partial recovery.

### 4.5 Explainability and Debuggability

Routing should be explainable as a sequence of decisions rather than a single opaque outcome.

### 4.6 Extensibility

The architecture should allow the semantic layer and the load-balancing layer to evolve independently.

---

## 5. Design Principles

The architecture is based on the following principles.

### 5.1 Separation of Concerns

Semantic routing and load balancing solve different problems and therefore must be implemented as separate layers.

### 5.2 Layered Contract

The output of the Semantic Routing layer is the authoritative model assignment. The Load Balancing layer must honor that assignment and perform scheduling only within the selected model pool.

### 5.3 Policy Before Expensive Execution

Governance, authorization, and semantic eligibility decisions should occur before requests reach costly inference backends.

### 5.4 Infrastructure Scheduling After Semantic Resolution

Backend selection should be based on operational state only after model resolution is complete.

### 5.5 Explicit Failure Semantics

Failure handling should preserve layer boundaries. Semantic failure and infrastructure failure are different failure classes and should be observable as such.

### 5.6 Deployment Flexibility

The gateway should support different deployment modes, including internal model clusters, external APIs, and hybrid serving topologies.

---

## 6. System Context

ASE sits between enterprise applications and a heterogeneous LLM serving environment.

At the northbound side, ASE exposes a unified request interface to clients and internal services. At the southbound side, ASE connects to one or more model-serving environments, which may include:

- internally hosted inference clusters
- vendor-hosted external model APIs
- model-specific serving pools
- region-specific or compliance-specific deployments
- mixed-capability fleets with different performance and cost profiles

ASE therefore functions as both:

- an **enterprise LLM gateway**
- a **control point for policy-aware request steering**

---

## 7. Two-Layer Architecture

### 7.1 Layer 1: Semantic Routing

The Semantic Routing layer determines **which model should serve the request**.

This layer is responsible for:

- request understanding
- routing signal extraction
- model eligibility filtering
- policy-aware model decision
- request enrichment with the selected `model`

This layer is **model-centric**. It is not responsible for choosing a machine.

### 7.2 Layer 2: Load Balancing

The Load Balancing layer determines **which backend instance should execute the already-selected model request**.

This layer is responsible for:

- model-pool resolution
- endpoint eligibility evaluation
- health-aware instance scheduling
- retry and failover control
- upstream traffic distribution

This layer is **instance-centric**. It is not responsible for reinterpreting prompt semantics.

### 7.3 Core Architectural Rule

The two layers are connected by a strict interface:

> The Semantic Routing layer resolves the target model.  
> The Load Balancing layer resolves the serving instance within that model’s backend pool.

This rule is the central architectural contract of the ASE LLM gateway.

---

## 8. High-Level Component Model

The ASE LLM gateway can be described as the following set of logical components:

### 8.1 Northbound Interface

Receives requests from clients and internal applications.

Responsibilities include:

- request ingress
- protocol adaptation
- authentication and identity propagation
- request normalization
- tracing context initialization

### 8.2 Semantic Routing Subsystem

Determines the target model for the request.

Responsibilities include:

- semantic signal extraction
- policy evaluation
- model selection
- routing metadata generation
- optional governance plugins

### 8.3 Request Enrichment Boundary

Defines the output contract from Semantic Routing to Load Balancing.

Typical data passed forward may include:

- resolved `model`
- session identifier
- tenant identifier
- request priority
- routing reason code
- policy tags

### 8.4 Load Balancing Subsystem

Schedules the request onto an eligible backend instance.

Responsibilities include:

- model-pool lookup
- endpoint state tracking
- instance selection
- retry/failover handling
- upstream dispatch

### 8.5 Backend Registry and Health State

Maintains dynamic knowledge of available serving endpoints and their operational state.

Responsibilities include:

- endpoint discovery
- pool membership
- health status
- drain status
- scheduling eligibility

### 8.6 Southbound Model Endpoints

The actual execution targets that serve requests.

These may be:

- internal model-serving instances
- vendor APIs
- regional deployments
- dedicated tenant pools
- compliance-scoped model clusters

### 8.7 Observability and Control Functions

Provide operational visibility and control over the system.

Responsibilities include:

- request tracing
- metrics emission
- routing audit logs
- error classification
- control-plane policy distribution

---

## 9. End-to-End Request Lifecycle

A typical request lifecycle in ASE is described below.

### Step 1: Request Ingress

A client sends a request to ASE, typically with a logical model target such as `model=auto` or another abstract routing form.

### Step 2: Request Normalization

ASE normalizes the incoming request into an internal representation suitable for gateway processing.

### Step 3: Semantic Evaluation

The Semantic Routing layer inspects the request and evaluates routing signals, policy constraints, and model eligibility.

### Step 4: Model Resolution

The Semantic Routing layer selects the target model and attaches it to the request.

At this point, semantic model selection is complete.

### Step 5: Pool Resolution

The Load Balancing layer resolves the backend pool corresponding to the selected model.

### Step 6: Endpoint Filtering

The Load Balancing layer removes endpoints that are unavailable, draining, or otherwise ineligible.

### Step 7: Instance Scheduling

The Load Balancing layer selects a serving instance using the configured balancing strategy.

### Step 8: Request Dispatch

ASE forwards the request to the selected backend instance.

### Step 9: Response Handling

The backend response is returned through ASE to the client, with optional tracing, audit tagging, and metrics recording.

### Step 10: Observability Update

Routing decisions, instance selection outcomes, latency, and failure results are recorded for operations and debugging.

---

## 10. Layer Boundary and Responsibility Split

A professional architecture requires that each layer own a distinct class of decisions.

### 10.1 Semantic Routing Owns

- semantic understanding of the request
- task and capability matching
- policy-aware model selection
- model eligibility filtering
- routing rationale
- request enrichment with `model`

### 10.2 Load Balancing Owns

- endpoint pool resolution
- health-aware scheduling
- per-instance traffic distribution
- retry behavior
- failover behavior
- drain and recovery behavior

### 10.3 Semantic Routing Does Not Own

- connection-level balancing
- backend liveness monitoring
- retry loops
- per-instance scheduling

### 10.4 Load Balancing Does Not Own

- prompt semantic interpretation
- task classification
- model capability analysis
- policy-driven model reassignment under normal operation

This explicit split prevents design drift and keeps the gateway understandable.

---

## 11. Failure Model

The two-layer design also improves failure handling by separating failure classes.

### 11.1 Semantic Failure

Semantic failure occurs when ASE cannot resolve a valid target model because of:

- invalid request shape
- missing required capabilities
- policy violation
- denied route override
- no model satisfies hard constraints

This is a **Layer 1 failure**.

### 11.2 Infrastructure Failure

Infrastructure failure occurs when a valid model has been selected, but no eligible serving instance is available because of:

- all endpoints down
- all endpoints draining
- connection failures
- upstream overload
- retry exhaustion

This is a **Layer 2 failure**.

### 11.3 Why This Matters

By separating these failure classes, ASE can expose clearer operational states and error reporting:

- “no eligible model”
- “model selected but no healthy backend”
- “request failed after dispatch”

This is more useful than a generic “routing failed” outcome.

---

## 12. Deployment View

The architecture is intentionally deployment-agnostic at the overview level. ASE may be deployed in several ways.

### 12.1 Centralized Gateway, Distributed Model Pools

A single logical ASE gateway fronts multiple model pools located in one or more backend clusters.

### 12.2 Hybrid Internal and External Serving

ASE routes some requests to internal inference clusters and others to external provider APIs.

### 12.3 Compliance-Aware Segmentation

ASE restricts certain requests to approved deployment zones while allowing other requests to route more broadly.

### 12.4 Tiered Serving Architecture

Different model pools may correspond to different service tiers, such as low-cost, high-throughput, or premium-quality serving.

The high-level two-layer design remains the same across all deployment modes.

---

## 13. Observability Model

Observability is a first-class concern in ASE because routing decisions must be explainable and operations must be able to distinguish semantic issues from infrastructure issues.

At the overview level, ASE should support observability for:

- request ingress volume
- model selection outcomes
- pool resolution outcomes
- instance selection outcomes
- dispatch latency
- backend failure classes
- retry/failover events
- semantic rejection or policy denial events

A key design objective is that operators can reconstruct the request path as:

**request received -> model selected -> pool resolved -> instance selected -> response or failure**

---

## 14. Relationship to Subsystem Documents

This overview document is the top-level document in a three-document design series.

### 14.1 Architecture Overview

Describes the overall ASE LLM gateway architecture, system boundary, request lifecycle, and the responsibility split between Semantic Routing and Load Balancing.

### 14.2 Semantic Routing Design

Describes how ASE extracts routing signals, applies semantic and policy decisions, resolves candidate models, and produces the authoritative `model` assignment.

### 14.3 Load Balancing Design

Describes how ASE maps the selected model to a backend pool and schedules requests across serving instances using professional load-balancing and reliability mechanisms.

This document intentionally avoids deep subsystem detail so that each subsystem design can evolve independently without destabilizing the top-level architecture.

---

## 15. Summary

ASE should be designed as a **two-layer enterprise LLM gateway**.

The first layer, **Semantic Routing**, determines the correct model for a request based on semantic, policy, and business context. The second layer, **Load Balancing**, determines the correct backend instance within the selected model pool based on operational state and service reliability considerations.

This separation is the key architectural principle of the system. It creates a clean contract between semantic decision-making and infrastructure scheduling, improves explainability, supports extensibility, and makes the gateway suitable for production-grade enterprise LLM deployments.

The remainder of the design series builds on this overview:

- `ASE Semantic Routing Design` specifies how model selection is performed.
- `ASE Load Balancing Design` specifies how instance selection and traffic reliability are performed.
