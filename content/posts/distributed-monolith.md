---
title: 'Distributed Monolith vs Microservices: Why Most Migrations Go Wrong'
slug: distributed-monolith
author: Adam Wasielewski
category: General
readingTime: 13
coverImage: /uploads/distributed-monolith/image1.png
---

## TLDR

* **A distributed monolith is a system that's physically distributed but behaviorally coupled:** services may run in separate processes or containers with separate pipelines, but can't be deployed independently; failures cascade across boundaries; and schema changes ripple through the whole system.
* **Four patterns create it**: lift-and-shift decomposition that mirrors code structure instead of domain boundaries, shared databases that make every schema change a system-wide event, shared libraries that become hidden monoliths, and integration code between services that recreates the same coupling that extraction was supposed to remove.
* **Three layers of coupling require three separate fixes**: domain boundaries require decomposing around business capabilities; database coupling requires per-service data ownership with dual-write or CDC during the transition; and the integration layer requires typed contracts enforced at compile time rather than discovered at runtime.
* **The integration layer is where most fixes stop short**; even with clean domain boundaries and separate databases, ad hoc HTTP clients, DTOs, and serialization code between services recreate tight coupling invisibly, and that surface grows quadratically with every new service pair added.
* **Graftcode reduces integration-layer maintenance** by replacing hand-written HTTP clients and DTOs with strongly typed Grafts installed via a package manager, and using GraftConfig to control whether calls run in-process locally or remotely in production via a single environment variable. Interface changes surface as package updates at compile time rather than as silent runtime failures.

Most teams that migrate to microservices are chasing the same outcomes: independent deployments, autonomous teams, and the ability to change one service without touching everything else. The decision to split the monolith feels like the right architectural move. The outcome, often, isn't what anyone expected.

Services end up in separate containers and Kubernetes deployments, sometimes in separate repos, sometimes not. But releases still require coordinating five teams. A schema change still ripples through the entire system. A single service going down still cascades into failures everywhere. The coupling didn't go away. It moved across a network.

This is the distributed monolith, one of the most common yet least discussed outcomes of a microservices migration. This guide covers what it is, the four specific ways teams build one without realizing it, how to diagnose whether you're in one, and how to fix the coupling at each layer, including the integration layer, which is where most fixes stop short.

## What Is a Distributed Monolith

A distributed monolith is what you get when a team splits a monolith into separate services without actually decoupling them. The services are physically separate, running in different containers, often with separate CI/CD pipelines, but they behave as a single tightly coupled system at runtime. Separate repos are more characteristic of true microservices than of distributed monoliths; in practice, distributed monolith services frequently share a monorepo. The physical separation is real. The behavioral independence is not.

The term was coined to describe a specific failure mode in microservices migrations: teams perform the structural work of decomposition without addressing the underlying coupling. The result is a system that's harder to operate than the original monolith (more infrastructure, more network hops, more failure points) while retaining all the coupling that made the monolith painful to change in the first place.

It's worth distinguishing from two adjacent concepts:

* A **modular monolith** is a single deployment with clean internal module boundaries, where coupling is contained, testable, and intentional
* **True microservices** have independent deployability, isolated failure domains, and enforced contracts between services
* A distributed monolith has the deployment complexity of microservices with the coupling of a monolith, the operational overhead without the independence.

## What a Distributed Monolith Looks Like in Practice

![](/uploads/distributed-monolith/image4.png)

A distributed monolith is a system distributed in form but monolithic in behavior. Services may run in separate containers, separate repos, or separate deployment pipelines, or they may share some of those. Still, regardless of the physical setup, they can't be developed, tested, or deployed independently. The reason is coupling: services share databases, call each other synchronously without fault tolerance, or depend on shared libraries that force coordinated releases. The architectural independence that microservices promised never materialized because the underlying coupling was never addressed; only the deployment topology changed.

In practice, a distributed monolith surfaces as a set of observable symptoms:

* A change to one service requires coordinating deploys across three others simultaneously
* A database schema change ripples through five services that all read the same tables
* A single container failure cascades into downstream failures across services that "shouldn't" depend on it
* Releases require lockstep coordination, even though every service has its own CI/CD pipeline

The coupling is still there. It's just invisible, spread across network calls, shared databases, and manually maintained integration contracts.

A distributed monolith isn't a choice teams make intentionally; it's what forms when a migration addresses the structural work of decomposition without addressing the underlying coupling. The result is a system that carries the operational overhead of microservices: extra infrastructure, extra network hops, extra failure points, while retaining the same tight coupling that made the original monolith painful to change.

Understanding why it forms is more useful than recognizing it after the fact.

## The Four Causes That Create a Distributed Monolith

Most distributed monoliths form gradually, not all at once. Four patterns appear repeatedly across migrations that end up in this state.

### **Lift-and-shift decomposition**

The most common cause. Teams split the monolith along existing code boundaries; a catalog module becomes a catalog service, an orders module becomes an orders service, rather than around domain boundaries. The services mirror the monolith's internal structure and inherit its coupling.

A catalog service that still depends on the same shared tables as the orders service hasn't been decoupled. It's just the same dependency with a network hop in between. The coupling is structural, not incidental. Moving code into containers doesn't redesign the domain.

### **Shared databases**

When multiple services read from and write to the same database tables, the database becomes the hidden coupling layer. Renaming a field in the orders table requires coordinated changes across every service that reads it, regardless of how cleanly those services are otherwise structured.

Schema changes become system-wide events. Teams coordinate releases around database migrations. The database owns the system's behavior in ways that the service boundaries don't reflect. No amount of service extraction fixes this; it has to be addressed at the data ownership level.

### **Shared libraries that become hidden monoliths**

A common serialization library, a shared domain model, and a validation package are used across all services. Any update to that library forces all dependent services to rebuild and redeploy simultaneously. The library serves as the coupling mechanism, a monolith hidden within the distributed system.

Teams often introduce shared libraries to avoid code duplication. The cost only becomes visible when the library needs to change, and every service team has to coordinate the update at the same time.

### **Integration code recreating the coupling**

This is the least-discussed cause and the one that compounds fastest. Every cross-service call in a different language requires:

* An HTTP endpoint defined on the provider service
* An HTTP client written in the caller's language
* DTO definitions matching the wire format on both sides
* Serialization and deserialization logic in both services
* Manual versioning when either side changes

That integration code is tightly coupled by design. A field rename in the provider breaks the caller at runtime with no compile-time warning. The integration layer recreates exactly the coupling that extraction was supposed to remove, just invisibly, across a network boundary. And as the number of service pairs grows, the integration surface grows quadratically.

## How to Diagnose a Distributed Monolith

![](/uploads/distributed-monolith/image2.png)

The following symptoms are observable indicators, not theoretical warning signs. Each one points to a specific type of coupling.

* **Lockstep deployments.** A change to Service A requires deploying Services B and C simultaneously. Indicates runtime coupling between services that should be independent.
* **Cascading failures.** Service A going down takes Service B with it, even though there's no explicit dependency. Indicates behavioral coupling or shared infrastructure.
* **Schema changes ripple everywhere.** Renaming a database column requires changes across multiple services. Indicates shared database coupling.
* **Growing inter-service parameter lists.** The payload passed between two services keeps gaining fields over time. Indicates the service boundary is in the wrong place.
* **Shared boilerplate across services.** The same serialization or domain logic is shared across multiple service codebases. Indicates a hidden shared library dependency.
* **Runtime field mismatches.** One service renames a field; another raises a Null or KeyError at runtime, with no compile-time warning. Indicates uncontrolled integration contracts.
* **Integration code is growing with every new service.** Each new service pair requires a new HTTP client, new DTO classes, and new serialization code. Indicates that the integration layer is growing ad hoc without systematic contract enforcement.

The last two are the most diagnostic for the integration layer problem specifically. If runtime field mismatches and growing integration code are present, the coupling is living in the service-to-service calls, and fixing domain boundaries alone won't resolve it.

The diagnosis points to three distinct layers of coupling, each requiring a different fix.

## Fixing the Distributed Monolith: Three Layers of Coupling

![](/uploads/distributed-monolith/image3.png)

The fixes for a distributed monolith are layered. Getting Layer 1 right without addressing Layer 3 still leaves the coupling in place. Each layer is a separate problem with a separate solution.

### **Layer 1: Domain boundary coupling**

**Fix:** Decompose around business capabilities, not code structure. Each service owns its domain completely: logic, data, and interface. No service directly reaches into another's domain model or database.

Conway's Law applies directly here: systems reflect the communication structures of the teams that build them. If the team structure doesn't align with the service boundaries, coupling reappears regardless of architectural decisions. Domain boundary fixes and org structure fixes have to happen together.

**When to do it:** Before any extraction. If the bounded context isn't clear enough to draw without ambiguity, the extraction will inherit the same coupling it's trying to escape.

### **Layer 2: Database coupling**

**Fix:** Each service owns its data. Use dual-write or Change Data Capture (CDC) during the transition to keep legacy and new schemas in sync. Once the new service's data is validated, cut over and stop shared table access.

This is the hardest fix operationally; data migrations are high-risk and slow. But it's the only real fix for database coupling. Services that share tables are not independent, regardless of how they're deployed.

**When to do it:** After the domain boundaries are clear. Database ownership follows domain ownership; you can't fix one without the other.

### **Layer 3: Integration layer coupling**

This is where most teams stop short. Even with clean domain boundaries and separate databases, integration code between services can recreate tight coupling if it isn't addressed deliberately.

The integration layer problem is both an architecture problem and a tooling problem. The pattern choices, Strangler Fig, Branch by Abstraction, and Anti-Corruption Layer, handle Layers 1 and 2 by defining where boundaries should sit and how to migrate toward them incrementally. But even with the right boundaries in place, the tooling used to implement cross-service calls determines whether those boundaries stay clean or quietly accumulate the same coupling that extraction was supposed to remove. Layer 3 needs both: the right boundary decisions and the right contract enforcement mechanism.

Three options exist:

**gRPC**: enforced contracts via `.proto` files, with compile-time interface validation through generated stubs. Protocol Buffers support backward-compatible field evolution; adding optional fields doesn't break existing consumers, and code generation is typically automated as part of the build pipeline rather than a manual step per change. The maintenance overhead is real but manageable: teams maintain `.proto` definitions as the source of truth, and interface drift surfaces at compile time rather than at runtime. The tradeoff is that proto files are a separate artifact to maintain alongside the service code, and in systems with many service pairs across multiple languages, keeping that surface consistent requires discipline.

**Event-driven**: async decoupling via a broker removes synchronous dependency entirely. Works well for fan-out, audit trails, and workloads that don't need an immediate response. Adds broker infrastructure overhead and doesn't suit request-reply patterns where the caller needs an answer before proceeding.

**Graftcode**: installs a strongly typed Graft via the package manager. No `.proto` files, no manual code generation step. The Graft mirrors the provider service's public interface exactly. Interface changes surface as the package updates at compile time.

## How Graftcode Reduces the Integration Layer Coupling and Contract Drift

Graftcode is a cross-runtime communication layer. Instead of writing HTTP clients, defining DTOs, and maintaining integration code across service pairs, a service installs a strongly-typed Graft via its standard package manager. The Graft is a generated interface that mirrors the provider service's public methods, method names, argument types, and return types, all expressed in the calling service's native type system.

The protocol underneath is Hypertube, a binary runtime bridge that connects language runtimes directly rather than wrapping HTTP. In [Graftcode's Performance Lab](https://gc-d-ca-polc-demo-perf-lab-01.blackgrass-d2c29aae.polandcentral.azurecontainerapps.io/) large-payload benchmark, Hypertube is 22ms vs gRPC's 53ms vs REST's 1,245ms, approximately 57× faster than REST and 2.4× faster than gRPC, with one-eighth the CPU consumption. Across backend teams, roughly 30–40% of engineering time is spent on API plumbing, DTO maintenance, and SDK versioning. Graftcode reduces that overhead.

### **Why gRPC and REST still leave the integration problem unsolved**

gRPC enforces contracts via `.proto` files and catches interface drift at compile time, which is a meaningful improvement over REST. The tradeoff is that `.proto` definitions are a separate artifact to maintain. Code generation is typically automated, but the `.proto` file itself still needs to be updated when the provider's interface changes, and that update has to propagate to consuming services before they can use the new interface. The overhead is controlled and predictable; it's not a blocker, but it's ongoing maintenance that scales with the number of service pairs.

REST has no enforced schema by default. When a provider renames a field, the consumer doesn't find out at compile time. Depending on where the renamed field is used, the failure mode varies: a missing response field may produce a `KeyError` or `null` in the caller. In contrast, a renamed request field is more likely to trigger a validation error or unexpected server-side behavior. Neither surfaces at compile time. Version coordination is manual, and the window between a provider change and a consumer breakage depends entirely on test coverage and deployment timing.

Graftcode generates the typed interface directly from the provider's public methods, no `.proto` files, no separate code generation step. When the provider's interface changes, that change is immediately visible as a package update in the consuming service's package manager. Incompatibilities surface at compile time in the calling service, not as runtime errors in production.

### **What cross-service calls look like without typed contracts**

In a distributed monolith, the integration code between services typically looks like this:

```python
# Service A calling Service B, without GraftCode
import requests

def call_order_processor(order_id: str, customer_id: str):
    response = requests.post(
        "http://order-processor:8080/api/v1/process",
        json={"orderId": order_id, "customerId": customer_id},
        headers={"Content-Type": "application/json"},
        timeout=5.0
    )
    result = response.json()
    # If order-processor renames "customerId" to "customer_id" on its side,
    # this request field goes unrecognized — the server may reject it with
    # a validation error or silently ignore it. No compile-time warning either way. 
    return result
```

Every field name must match the provider's schema exactly. A rename in the provider breaks the caller at runtime. In a 10-service system with multiple language pairs, maintaining this surface is a full-time job.

The same pattern repeats regardless of language, every service pair maintaining its own HTTP client, DTOs, and manual field mapping

### **How Graftcode works: Gateway, Graft, and GraftConfig**

Graftcode Gateway runs alongside the provider service, not between services. It exposes the provider's public methods to callers. A single Gateway can host one or multiple modules depending on how the deployment is structured; teams may run one Gateway per service or consolidate multiple modules behind a single Gateway. Each Gateway serves only the methods of the modules it hosts.

```bash
# Start the Gateway alongside the order processor service
gg --modules ./order_processor.py
```

The Gateway serves on port 80 (calls) and port 81 (Graftcode Vision, live documentation of all exposed methods and their signatures).

*Graft* is a strongly typed interface generated within the calling service and installed via the standard package manager. The install command is generated by Graftcode Vision at `http://<provider-host>:81/GV`:

```bash
# Install the Graft in Service A
npm install --registry https://grft.dev/<project-id> @graft/nuget-orderprocessor@1.0.0
```

The Graft is a real installed package, not a handwritten client. It reflects the provider's public interface precisely: method names, argument types, and return types. IDE autocompletion works on every method. When the provider changes a method signature, the Graft package updates, and the incompatibility surfaces at compile time in the calling service. Teams apply the update on their own schedule; it appears as a standard package version bump.

*GraftConfig* lives within the calling service and controls whether the Graft call runs in-process (no network hop) or remotely (via the deployed Gateway). Set via environment variable, not a code change.

The calling code doesn't change between local development and production. Only `GraftConfig.host` changes, set via environment variable.

```javascript
const { GraftConfig, OrderProcessor } = require("@graft/nuget-orderprocessor");

// Local development — runs in-process, no network call
GraftConfig.host = "inMemory";

// Production: points to the order processor's Gateway
GraftConfig.host = "wss://order-processor:9000/ws";

// Strongly typed call — no HTTP client, no DTO, no JSON parsing
const result = await OrderProcessor.processOrder(orderId, customerId);
// If order-processor renames "customerId", this line fails at compile time
```

### **What disappears when you replace the integration layer**

| Without Graftcode                      | With Graftcode                               |
| -------------------------------------- | -------------------------------------------- |
| HTTP client written per language pair  | Graft installed via package manager          |
| DTO classes on both sides              | Strongly typed interface auto-generated      |
| Runtime field-name mismatches          | Compile-time incompatibility detection       |
| JSON serialization/deserialization     | Handled by Hypertube binary protocol         |
| Manual versioning coordination         | Interface changes surface as package updates |
| New HTTP client per service pair added | One Graft install per service dependency     |

### **GraftConfig as a distributed monolith prevention mechanism**

GraftConfig's in-memory-to-remote switch maps directly to the migration path. Modules run in-process within a modular monolith during development `(GraftConfig.host =` `"inMemory"`), using the same calling code that will eventually run remotely. When a module is extracted into its own service, a single environment variable change in the calling service's GraftConfig points it to the deployed Gateway with no code change required.

This means the integration contract is enforced from the start of the project, not retrofitted after the distributed monolith has already formed. Every cross-module call is a typed interface, every change surfaces at compile time, and the coupling that builds up invisibly in traditional integration code doesn't have room to form.

## Preventing a Distributed Monolith Before It Forms

Prevention is significantly cheaper than remediation. The distributed monolith forms gradually; by the time it's fully visible, it's embedded in deployment pipelines, on-call rotations, and team workflows.

**Define domain boundaries before writing extraction code.** If the bounded context isn't clear enough to draw unambiguously, the extraction will inherit the same coupling it's trying to escape. Event Storming is a practical technique for mapping domain events and service boundaries before any code moves.

**No shared databases from day one.** One service owns each table. If two services need the same data, one owns it and exposes it via a typed interface.

**Enforce integration contracts at compile time from the start.** Every cross-service call should fail at compile time when the contract changes, not at runtime in production. gRPC's `.proto` files or Graftcode's typed Grafts both provide this. Ad-hoc HTTP clients with manual JSON parsing provide neither.

**Start with a modular monolith.** Clean internal module boundaries with in-process calls give teams the architectural discipline of microservices without the operational overhead. Extract when the domain is understood, and the team is ready, not because microservices sound like the right architecture.

## When a Distributed Monolith Is Hard to Escape

Some distributed monoliths are harder to undo than others. Four scenarios make remediation significantly more expensive:

**Deeply normalized shared databases.** When tables are shared across 20 services and tightly normalized across domain boundaries, data migration is a multi-year project. There's no shortcut; the data has to be separated before the services can be independent.

**A fundamentally broken domain model.** If the original monolith's domain model was wrong, not just poorly implemented but conceptually incorrect, extraction perpetuates the bad model across services. The right move here is to design the target domain model first, then migrate to it, rather than extracting the existing structure into separate deployments.

**Team structure is misaligned with service boundaries.** Conway's Law means the coupling will reappear regardless of tooling if the teams building the services are still organized around the old monolith's structure. Org structure changes have to accompany architecture changes.

**An integration layer that has become load-bearing.** When hundreds of services call each other through shared HTTP client libraries that have grown to hundreds of thousands of lines of code, the integration layer effectively becomes a second monolith. Replacing it requires treating it as a migration project in its own right.

In these cases, the most pragmatic move is often to consolidate back toward a modular monolith, fix domain boundaries, data ownership, and integration contracts correctly, and then extract again from a clean foundation.

## Conclusion: Distributed Monolith vs Microservices

A distributed monolith doesn't form because teams made the wrong architectural decision. It forms because migrations address one or two of the three coupling mechanisms without addressing all three. Domain boundary fixes without database fixes leave shared state coupling in place. Database fixes without integration layer fixes leave runtime contract drift in place.

The integration layer is where most remediation efforts stop short. Eliminating the HTTP clients, DTOs, and ad-hoc serialization code that recreate coupling across service boundaries, and enforcing those contracts at compile time rather than discovering drift in production, is what separates true microservices from a distributed monolith.

## FAQs

### **1. What is the difference between a distributed monolith and true microservices?**

True microservices can be developed, tested, and deployed independently; a change to one service doesn't require coordinating with others. A distributed monolith has the physical structure of microservices, separate processes, containers, or deployments, but retains the tight coupling of a monolith. Changes still require coordination. Failures still cascade. Releases still need lockstep timing. The services are distributed in form but coupled in behavior.

### **2. Is a modular monolith better than a distributed monolith?**

For most teams at most stages of growth, yes. A modular monolith with clean internal boundaries, separate data ownership per module, and enforced interfaces between modules gives teams most of the architectural benefits of microservices, independent development, clear ownership, and testable boundaries, without the operational overhead of distributed systems. The distributed monolith gets the operational overhead without independence. Starting with a modular monolith and extracting when the domain is understood is a more reliable path than extracting too early.

### **3. Can a distributed monolith be fixed without starting over?**

In most cases, yes, but it requires deliberately addressing all three coupling layers. Domain boundaries can be redefined incrementally using the Strangler Fig pattern. Database coupling can be resolved using dual-write or CDC during a transition period. Integration layer coupling can be resolved by replacing ad hoc HTTP clients with typed interfaces that enforce contracts at compile time. The fixes take time and discipline, but don't require starting from scratch unless the domain model itself is fundamentally broken.

### **4. What role does Conway's Law play in creating distributed monoliths?**

Conway's Law states that systems reflect the communication structures of the teams that build them. If teams are organized around the old monolith's structure a shared platform team, a single database team, and a cross-cutting API team, the services they build will reflect that structure. Service boundaries will mirror team boundaries, not domain boundaries. The distributed monolith is often as much an organizational problem as an architectural one. Fixing the architecture without changing the team structure means the coupling reappears with the next round of changes.

### **5. How does Graftcode reduce integration-layer maintenance between services?**

Graftcode replaces hand-written HTTP clients, DTOs, and serialization logic with a strongly typed Graft installed via a package manager. The Graft reflects the provider service's current public interface. When the provider changes a method signature, the incompatibility can surface as early as the moment the Graft is updated, before the code is even compiled, and is caught at compile time at the latest if the update happens alongside a build. Either way, the mismatch shows up before the calling service ships, not as a silent null or `KeyError` in production. GraftConfig controls whether calls run in-process (local development) or remotely (production), switching via an environment variable with no code changes. The integration contract is enforced from the start, reducing the hand-written HTTP clients, DTO mapping, and serialization work that would otherwise accumulate as new services are added.
