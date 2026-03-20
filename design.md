# ASE Semantic Routing and Load Balancing

## Introduction

Large Language Model (LLM) services are becoming a core component in modern AI applications. As enterprises deploy multiple LLMs with different capabilities, costs, and performance characteristics, a single static routing strategy is no longer sufficient. Some requests require strong reasoning or coding ability, while others can be handled by smaller and cheaper models. At the same time, even after a model is selected, the request still needs to be dispatched to an appropriate backend instance based on runtime conditions such as queue length, latency, and resource utilization.

To address this challenge, ASE adopts a **two-stage architecture** for LLM request processing. The first stage, **semantic routing**, analyzes the incoming request and determines the most suitable target model. After this step, the request is enriched with a `model` field that explicitly records the selected model. The second stage, **load balancing**, takes the model-selected request and dispatches it to one of the available backend instances that serve the selected model, based on infrastructure-level signals such as server load, health status, and runtime pressure.

This separation is important because **model selection** and **traffic distribution** solve two different problems. Semantic routing answers the question: **which model should serve the request**. Load balancing answers the question: **which server instance should execute the request efficiently and reliably**. By decoupling these two responsibilities, ASE can improve cost efficiency, latency, reliability, and operational clarity in production LLM deployments.

---

## Background

ASE already serves as an important boundary component between internal and external networks. Traditionally, it functions as a secure web proxy and traffic mediation layer. This makes ASE a natural control point for enterprise LLM access, where requests can be inspected, governed, routed, and distributed before reaching external or internal LLM serving systems.

However, LLM traffic differs significantly from traditional web traffic. In conventional load balancing, requests are typically treated as stateless and interchangeable. Algorithms such as round robin or least connection are often sufficient because the backend servers provide the same service and the execution cost of each request is relatively predictable. In contrast, LLM requests are **heterogeneous, state-sensitive, and highly variable in cost**. Different prompts may require different model capabilities, different context window lengths, and very different inference budgets. In addition, backend inference servers may exhibit significant performance variation depending on current queue depth, batching state, KV cache pressure, and GPU utilization.

Because of these characteristics, a traditional load balancer alone cannot solve the LLM routing problem. An enterprise LLM gateway must answer two distinct questions:

1. **Which model should handle this request?**  
   This depends on the semantic intent of the request, such as whether it is a coding task, a reasoning task, a simple factual query, or a structured extraction request.

2. **Which server should execute the selected model request?**  
   This depends on runtime infrastructure conditions, such as server health, queue length, token load, latency, cache usage, and failover availability.

Therefore, ASE should not treat semantic routing and load balancing as a single undifferentiated function. Instead, it should adopt a layered design in which **semantic routing** is performed first at the request level, followed by **model-specific load balancing** at the serving layer.

---

## System Design

### Design Overview

The ASE LLM routing architecture is composed of two logically independent stages:

1. **Semantic Routing Layer**  
   This layer inspects the incoming request and selects the most appropriate target model. The output of this stage is an updated request with an explicit `model` field.

2. **Load Balancing Layer**  
   This layer receives the model-selected request and forwards it to a specific backend server instance that serves the selected model.

This design ensures a clean separation between **application-level decision making** and **infrastructure-level scheduling**.

A simplified request flow is shown below:

**Client Request → Semantic Routing → Request Enrichment (`model=...`) → Load Balancing → Selected Model Server**

---

### 1. Semantic Routing

#### 1.1 Purpose

The semantic routing stage determines **which model should serve a request**. Its goal is not to pick a server, but to classify the request and map it to the most suitable model family or model pool.

For example:

- coding-related prompts may be routed to a code-specialized model
- simple factual questions may be routed to a cheaper general-purpose model
- complex reasoning tasks may be routed to a stronger reasoning model
- structured extraction requests may be routed to a model optimized for JSON or schema-constrained output

This stage is responsible for **request understanding and model assignment**.

#### 1.2 Input Signals

The semantic router may consider the following signals:

- **Task type / domain classification**  
  Whether the request is about coding, mathematics, legal analysis, general chat, document extraction, and so on.

- **Complexity estimation**  
  Whether the request is simple, moderate, or reasoning-intensive.

- **Context length estimation**  
  Whether the request requires a long-context model.

- **Safety and governance signals**  
  Whether the request contains sensitive information, policy-sensitive content, or possible jailbreak attempts.

- **User preference or routing hints**  
  Optional request parameters such as low-cost preference, low-latency preference, or route override.

- **Business policy constraints**  
  Tenant-level restrictions, budget caps, or model availability policies.

#### 1.3 Output

The output of semantic routing is a request enriched with a target model field, for example:

```json
{
  "model": "code-model",
  "messages": [
    {
      "role": "user",
      "content": "Write a C epoll example"
    }
  ]
}
````

This is the key contract between the two stages: **the semantic routing layer decides the model, and the load balancing layer honors that decision**.

#### 1.4 Design Principles

The semantic routing stage should satisfy the following principles:

* **Model-centric, not server-centric**
  It should select the model class, not a specific machine.

* **Policy-aware**
  It should enforce safety, governance, and tenant rules before the request reaches the expensive inference backend.

* **Cost-performance aware**
  It should avoid overusing expensive models for simple requests.

* **Extensible**
  New classifiers, business rules, and routing policies should be easy to plug in.

---

### 2. Load Balancing

#### 2.1 Purpose

Once the request has been assigned a model, the load balancing stage decides **which backend server instance should serve that model request**.

For example, if semantic routing chooses `code-model`, the load balancer selects one server from the `code-model` serving pool. This stage is purely about efficient and reliable request dispatch among instances that are already capable of serving the chosen model.

#### 2.2 Inputs

The load balancing stage takes two classes of inputs:

* **Selected model**

  * the `model` field produced by the semantic routing layer
  * this field identifies the eligible backend pool

* **Backend runtime signals**

  * server health status
  * queue length
  * number of in-flight requests
  * observed latency
  * GPU utilization
  * KV cache usage
  * prefix cache locality
  * rate limit status
  * failover availability

#### 2.3 Core Logic

The load balancer first filters the backend pool by the selected model, then applies an instance selection algorithm based on live system conditions.

A typical flow is as follows:

1. Read `model=code-model`
2. Find all serving instances that host `code-model`
3. Remove unhealthy or overloaded instances
4. Rank or select instances using the configured balancing algorithm
5. Forward the request to the chosen backend

This stage does not change the selected model. It only determines the best execution target for that model.

#### 2.4 Supported Algorithms

The load balancing stage may support multiple algorithms depending on deployment scale and system goals:

* **Round Robin / Weighted Round Robin**
  Suitable for basic deployments with homogeneous instances.

* **Least Connection / Least In-Flight Requests**
  Better for handling variable request durations.

* **Least Queue Depth**
  Useful when backend queue pressure is the dominant bottleneck.

* **Token-aware Scheduling**
  Routes requests based on estimated input and output token cost.

* **KV-cache-aware or Prefix-cache-aware Scheduling**
  Improves cache reuse for repeated or similar prompts.

* **Consistent Hashing / Session Affinity**
  Helps preserve locality for multi-turn sessions.

In ASE, the default strategy can begin with a simple model-pool-based least-load policy, and then evolve toward token-aware and cache-aware scheduling as the serving stack becomes more advanced.

#### 2.5 Reliability Functions

The load balancing layer is also responsible for runtime reliability, including:

* health checks
* failover
* retry policies
* circuit breaking
* endpoint draining
* degraded-mode routing

This allows ASE to continue serving requests even when some model instances are slow, overloaded, or unavailable.

---

### 3. Separation of Responsibilities

The most important architectural principle is the separation between semantic routing and load balancing.

#### Semantic Routing answers:

**Which model should this request use?**

#### Load Balancing answers:

**Which server should execute the selected model request?**

This separation provides several benefits:

* **Clear control boundaries**
  Model policy and infrastructure scheduling can evolve independently.

* **Better explainability**
  A routing decision can be traced as: first model selected, then instance selected.

* **Simpler extensibility**
  New models can be added to the semantic router without redesigning backend balancing logic.

* **Better operations**
  Load balancing policies can be tuned using runtime metrics without changing semantic classification logic.

---

### 4. End-to-End Request Lifecycle

An end-to-end ASE request may follow the process below:

1. A client sends a request with `model=auto`
2. ASE semantic routing analyzes the request content
3. ASE selects the target model, such as `general-small`, `code-model`, or `reasoning-large`
4. ASE writes the selected value into the request `model` field
5. ASE load balancing identifies the serving pool for that model
6. ASE selects a backend instance based on health and load signals
7. The request is forwarded to the selected LLM server
8. The response, metrics, and debug traces are returned or recorded

This workflow ensures that semantic policy and runtime efficiency are both enforced in a structured way.

