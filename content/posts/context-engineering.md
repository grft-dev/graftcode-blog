---
title: Software Architecture Is Context Engineering for AI Coding Agents
slug: context-engineering
author: Adam Wasielewski
category: General
readingTime: 12
---

## TLDR

* **Context engineering is the design of everything an AI agent can see**: files, tool definitions, service interfaces, and retrieved state. Agents fail not because models can't reason but because they burn their context window on irrelevant information before reaching what matters.
* **Software architecture is the most important context engineering decision**; it determines what the agent reads, how accurately that reflects system behavior, and what error signals it gets when generated code is wrong.
* **Cross-service integration code creates three specific problems for agents**: it buries business intent in plumbing, makes schema drift invisible until runtime, and forces agents to parse transport-layer artifacts to infer a callable interface.
* **Graftcode addresses all three:** agents read a method signature instead of multiple integration files; the contract is semantically accurate rather than inferred from HTTP client code; contract changes surface as compile-time errors rather than silent runtime failures; and every hosted service automatically exposes an MCP endpoint so agents can call it directly.
* **An agent-ready codebase looks like a well-architected human codebase**: typed interfaces, compile-time enforcement, minimal integration surface, clear boundaries, consistent conventions. Agent error rates and exploration overhead are now measurable feedback on how well the architecture embodies these properties.

Context engineering is the practice of deliberately designing everything a model receives, not just the prompt, but the files, tool definitions, service interfaces, and retrieved state that surround every agent action. For AI coding agents like Claude Code, Cursor, and Copilot Workspace, the context window is the working environment. What the agent can do is bound by what it can read. And what it can read is determined, more than anything else, by how the codebase is structured.

Most teams miss this connection: software architecture is already context engineering, just for human developers. The same properties that make a codebase readable for engineers also make it legible and efficient for AI agents. Clear module boundaries, enforced interfaces, and minimal integration overhead reduce context cost for both. And the properties that make codebases painful for humans, implicit coupling, DTO sprawl, and undocumented integration contracts, make them expensive and unreliable for agents for the same reasons. This guide covers what context engineering is, why architecture is the most important lever for it, where context gets wasted in typical microservices codebases, and how reducing the integration surface improves agent quality, reliability, and efficiency as agentic workflows scale.

## What is Context Engineering

Context engineering is the deliberate practice of designing the inputs an LLM receives to enable it to reason accurately and act reliably. It goes beyond prompt engineering, which treats the model input as a string to optimize, to treat the model's entire operating environment as a system to design.

For AI coding agents, that environment includes:

* The files and code snippets retrieved for the current task
* Tool definitions: what the agent can call, and how
* Service interfaces: what methods exist, and what they accept
* Conversation and task history
* Architectural documentation, if it exists and is machine-readable

The binding constraint is the context window. Everything the agent reasons over must fit. Anything that doesn't fit or crowds out what matters degrades output quality. Agents don't fail because frontier models can't reason. They fail because the agent burns its context window reading irrelevant information before it gets to the part that matters.

Research on coding agent behavior confirms this: read operations dominate agent token consumption at over 76%, and agents spend an excessive amount of their budget repeatedly exploring the codebase. This is why retrieval strategies, memory systems, and prompt formatting, the usual focus of context engineering advice, only go so far.

| Dimensions | Prompt engineering                 | Context engineering                       |
| ---------- | ---------------------------------- | ----------------------------------------- |
| Focus      | The instruction given to the model | Everything the model can see when it acts |
| Scope      | Single input string                | Full operating environment                |
| Levers     | Wording, examples, format          | Files, interfaces, architecture, memory   |
| Bottleneck | Model understanding                | Context window quality                    |

They operate downstream of a more fundamental constraint: the structure of the codebase itself.

## Why Software Architecture Is the Most Important Context Engineering Decision

![](/uploads/context-engineering/image4.png)

An agent navigating a well-structured codebase accomplishes more per read operation. It reaches the logic it needs to change without first tracing through layers of integration infrastructure. It sees typed interfaces that accurately represent what services do rather than inferring behavior from HTTP client code. And when contracts change, it gets a concrete error signal at compile time rather than discovering the mismatch at runtime.

An agent navigating a tightly coupled codebase with implicit contracts, DTO sprawl, and undocumented integration code faces the opposite situation. It has to read more files to understand less. It may need to trace five files to understand what one function does. It may need to parse an OpenAPI spec, an HTTP client implementation, and a DTO definition just to understand how two services communicate, and even after all of that, it may still infer the wrong behavior because the integration code doesn't accurately represent the contract.

Four architectural properties directly improve agent context quality and reliability:

**Clear module boundaries.** When each module has a well-defined interface and owns its domain, an agent can work within it without needing to understand the entire codebase. The boundary contains the context. An agent scoped to an `OrderService` module doesn't need to read the inventory system to understand what the order module does. Exploration stays bounded.

**Enforced contracts between services.** When service-to-service contracts are explicit, via typed interfaces, not ad hoc HTTP clients, the agent sees a method signature instead of an HTTP client implementation. The callable surface is smaller, more readable, and semantically accurate. More importantly, when contracts change, that mismatch fails before production once the updated Graft is applied, at build time in typed languages, or as an import/attribute error in dynamic ones, rather than generating code that fails silently at runtime with no signal at all.

**Minimal integration boilerplate.** Every line of serialization code, DTO definition, and HTTP client setup that the agent reads is a read operation spent on plumbing rather than business logic. The more boilerplate exists between services, the more the agent has to read to understand each service interaction, and agents spend the majority of their token budget on read operations. Reducing what there is to read directly reduces what the agent has to read.

**Consistent conventions.** Agents generalize from patterns. A codebase with consistent error handling, dependency injection, and naming conventions enables an agent to transfer what it learns from one module to the next. Inconsistency forces reading more code to understand each new pattern encountered.

| Property                | Agent impact                                                                                                                                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Clear module boundaries | Exploration stays bounded                                                                                                                                                      |
| Enforced contracts      | Fails before production once the updated Graft is applied, at build time in typed languages, or as an import/attribute error (or via an optional type checker) in dynamic ones |
| Minimal boilerplate     | Less to read, fewer wrong inferences                                                                                                                                           |
| Consistent conventions  | Patterns generalize across modules                                                                                                                                             |

## Where Context Gets Wasted in Typical Microservices Codebases

![](/uploads/context-engineering/image3.png)

Microservices architectures, if structured without attention to integration overhead, create specific, observable problems for AI coding agents.

**Cross-service integration code obscures business intent.** When Service A calls Service B via HTTP, the codebase includes an HTTP client in Service A's language, a DTO class that matches Service B's expected schema, serialization and deserialization logic, error handling for HTTP failures, and often a shared client library that wraps it all. An agent who is asked to make a change to this call must listen to the entire call to understand what's happening. None of that code describes business intent; it describes the integration plumbing. The agent is reading the infrastructure to understand a business operation.

> *Agent impact: reads infrastructure to infer a business operation.*

```python
# What an agent reads to understand one cross-service call, without typed contracts
import requests

def call_order_processor(order_id: str, customer_id: str):
    response = requests.post(
        "http://order-processor:8080/api/v1/process",
        json={"orderId": order_id, "customerId": customer_id},
        headers={"Content-Type": "application/json"},
        timeout=5.0
    )
    result = response.json()
    return result
```

To fully understand this call, the agent also reads the DTO definition for the request body, the DTO for the response, any OpenAPI spec or endpoint documentation, and error-handling wrappers. That's multiple files for one service interaction, and agents are already spending over 76% of their token budget on file reads.

**Schema drift is invisible to agents until it breaks.** When a service renames a field, downstream HTTP clients break at runtime. An agent generating code for those callers has no compile-time signal to act on; it produces plausible-looking code that fails in production. The agent can't distinguish a correct call from a stale one by reading the code alone.

> *Agent impact: generates code against a stale contract with no prior warning.*

**OpenAPI specs are a wide surface area with low semantic density.** When agents read service interfaces to understand what a service can do, they're parsing endpoint paths, HTTP verbs, request body schemas, response schemas, and error schemas. Most of that information describes the transport layer rather than the business logic. A typed method signature communicates the same callable surface in a fraction of the space and with higher semantic accuracy.

*Agent impact: infers intent from transport artifacts rather than the actual callable surface.*

**Distributed monolith patterns force cross-boundary exploration.** In a codebase where services are physically separate but implicitly coupled through shared databases, shared libraries, or undocumented integration contracts, an agent must trace dependencies across service boundaries to understand the impact of a single change. The coupling is invisible in the code structure but present at runtime. The agent discovers it only when something breaks, after it has already generated code based on an incomplete picture.

## How Graftcode Improves Context Quality for AI Coding Agents

The integration layer between services is where agent context quality degrades fastest. It's where business logic is buried under plumbing, where contracts are implicit, and where the agent reads the most to understand the least. Graftcode addresses this at the source by replacing that integration layer with typed interfaces that are accurate, minimal, and compile-time-enforced. The result for AI coding agents is threefold: less to read per service interaction, a more accurate representation of what each service actually does, and a deterministic error signal when contracts change rather than a silent runtime failure.

### **What an agent reads without Graftcode**

To understand how Service A calls Service B, the agent reads the HTTP client implementation, DTO definitions, endpoint documentation, and error handling. Each service interaction requires retrieving and parsing multiple files before the agent can reason about the underlying business logic.

```python
# Agent must read all of this to understand one service interaction
import requests

def call_order_processor(order_id: str, customer_id: str):
    response = requests.post(
        "http://order-processor:8080/api/v1/process",
        json={"orderId": order_id, "customerId": customer_id},
        headers={"Content-Type": "application/json"},
        timeout=5.0
    )
    result = response.json()
    # No compile-time warning if order-processor renames "customerId" on its side --
    # the request field goes unrecognized, and the failure surfaces as a
    # server-side validation error, not a warning the agent can see in advance
    return result
```

The agent reads infrastructure to infer business intent. And the inferred contract is fragile: one field rename, and the code the agent generates has no compile-time signal to catch it before it reaches production.

### **What an agent reads with Graftcode**

```python
// The entire interface: one method signature
const result = await OrderProcessor.processOrder(orderId, customerId);
// If order-processor renames "customerId", this line fails at compile time
```

A real extraction shows the same pattern at a larger scale. A service that inlines Reddit API auth and fetch logic directly in a route handler looks like this before Graftcode:

```python
# Before — Reddit auth and fetch logic inline in main.py
import praw
import requests

def get_reddit_access_token_from_refresh():
    response = requests.post("https://www.reddit.com/api/v1/access_token", ...)
    return response.json()["access_token"]

@app.post("/generate_comment")
async def generate_comment(request):
    token = get_reddit_access_token_from_refresh()
    headers = {"Authorization": f"bearer {token}", ...}
    response = requests.get(
        f"https://oauth.reddit.com/comments/{post_id}",
        headers=headers
    )
    # parse raw JSON, handle pagination, error handling...
```

To understand what `/generate_comment` does, an agent has to read token refresh logic, header construction, raw JSON parsing, and pagination handling, none of which is about generating a comment. After extraction into a Graftcode service:

```python
# Install the Graft
python -m pip install --extra-index-url https://grft.dev/simple/<project-id>__free graft-nuget-redditservice==1.0.0 
```

```python
# After -- one typed method call via Graftcode
import os
from graft_nuget_redditservice.graft.nuget.RedditService import GraftConfig
from graft_nuget_redditservice.redditservice import RedditService

GraftConfig.host = "wss://reddit-service-host/ws"

@app.post("/generate_comment")
async def generate_comment(request):
    post = RedditService.get_submission(url=reddit_url)
```

The agent reads one line to understand what the route does. Auth, pagination, and response parsing move into `reddit_service/reddit_service.py`, where they belong, and the agent only needs to read that file if it's actually working on Reddit integration logic, not every time it touches this route.

The callable surface is the method signature. No endpoint path, no HTTP verb, no request body schema, no status code handling. The agent reads what the service does, not how it communicates. And if the contract changes, the agent gets a compile-time error rather than a silent runtime failure, a concrete signal it can act on before generating downstream code.

### **The three Graftcode components and why they are important for agents**

Graftcode Gateway runs the provider service; it spins up a runtime for the configured modules, exposes their methods, and is not positioned between services. By default, it exposes public methods automatically once a module is hosted; developers can narrow that surface with access policies if they want to control which methods are actually reachable, but nothing needs to be explicitly configured just to expose a module's public interface. A Gateway can host one or multiple modules depending on how the service is structured; there's no central router and no shared configuration for the agent to trace across services.

```python
# Start the Gateway, which runs the order processor service
# --projectKey accepts either a raw JWT token or a username:jwt_token / env:jwt_token reference
./gg --projectKey <username>:<jwt_token> --modules <path-to-your-modules>
```

The Gateway serves on port 80 (calls) and port 81 (Graftcode Vision, live documentation of all exposed methods, their signatures, and generated install commands for every supported language). It also exposes a Model Context Protocol (MCP) endpoint automatically for every hosted service, meaning an AI agent can call a service's methods directly through MCP, not just through an installed Graft, without any additional setup.

**Graft** is a strongly typed interface generated inside the calling service, installed via a standard package manager. The install command is generated by Graftcode Vision at `http://<provider-host>:81/GV`:

```bash
npm install --registry https://grft.dev/<project-id>__free @graft/nuget-orderprocessor@1.0.0
```

The Graft reflects the provider's public interface exactly: method names, argument types, return types. IDE autocompletion works on every method. When the provider changes a method signature, that change is immediately visible as a package update; teams choose when to apply it. Once applied, any incompatibility surfaces at compile time in the calling service, before it reaches production and before an agent generates downstream code against a stale contract.

Graftcode supports 20 programming languages across 10 package managers, which is especially important for agents working in polyglot codebases. The calling pattern, install a Graft, read a typed method signature, remains consistent regardless of whether the provider service is written in Python, Java, Go, or Rust. An agent doesn't need a different mental model per language pair.

**GraftConfig** is an internal class of each Graft, controlling whether that Graft's call runs in-process or remotely. Set via environment variable, config file, or directly on the Graft, not a code change:

```python
const { GraftConfig, OrderProcessor } = require("@graft/nuget-orderprocessor");

// Local development: runs in-process, no network call
GraftConfig.host = "inMemory";

// Production: points to the order processor's Gateway
GraftConfig.host = "wss://order-processor:9000/ws";

// Same method call in both environments
const result = await OrderProcessor.processOrder(orderId, customerId);
```

For agents specifically, this consistency removes an entire class of context noise. When service calls branch based on environment, different URLs, different auth headers, and different client initialization, the agent has to read and reason about that branching to understand what the code does in production. GraftConfig externalizes that concern entirely. The agent reads one method call and knows it behaves the same way everywhere.

### **Why the improvement compounds across agentic workflows**

AI coding agents in production don't make one service call. They orchestrate multi-step workflows: read the order, call the fulfillment service, notify the customer, update inventory. Each cross-service interaction in a REST-based architecture requires the agent to read multiple files before it can reason about the interaction. In a Graftcode-based architecture, each interaction is a method signature.

The improvement compounds in three ways:

**Fewer files to retrieve per workflow.** Agents spend the majority of their token budget on file reads, and a large share of that budget goes to repeatedly exploring the codebase. When each service interaction is a typed method call rather than a chain of HTTP client, DTO, and endpoint spec, the agent retrieves fewer files to accomplish the same task.

**More accurate understanding per read.** A typed method signature accurately and completely communicates the service's callable surface. An HTTP client implementation communicates with the transport layer; the agent has to infer the actual contract from how the plumbing is wired. Inference introduces error; typed interfaces reduce it.

**Deterministic error signals instead of runtime surprises.** When an agent generates code that calls a service with the wrong field name, a REST-based architecture surfaces that as a runtime failure after deployment. A Graftcode-based architecture surfaces it as a compile-time error before the agent generates downstream code based on the wrong assumption. Compile-time enforcement makes the agent's feedback loop tighter and its output more reliable.

Across backend teams, roughly 30–40% of engineering time is spent on API plumbing, DTO maintenance, and SDK versioning. For AI coding agents, that same surface is context overhead and a source of unreliable inference. Graftcode reduces it for both. Graftcode also runs up to 70% faster than conventional web services with one-eighth the CPU consumption, overhead reductions that compound alongside the context reduction agents experience.

## Practical Context Engineering Principles for Teams Using AI Coding Agents

**Design module interfaces as agent entry points.** When an agent is given a task, its first action is typically to read the module's public interface to understand what it can do. Make those interfaces explicit, typed, and minimal. An agent that reads `OrderService.getOrder()`, `OrderService.createOrder()`, and `OrderService.updateStatus()` has a complete and accurate map of the module's capabilities without reading a single implementation file.

**Enforce contracts at compile time.** When service contracts are enforced at compile time, through typed Grafts, gRPC stubs, or strict TypeScript interfaces, agent-generated code that violates the contract fails before production. When contracts are enforced at runtime, the agent produces plausible-looking code that only fails when it runs. Compile-time enforcement makes the contract visible, accurate, and enforceable in the same step.

**Minimize integration boilerplate.** Every HTTP client, DTO class, and serialization wrapper that the agent doesn't have to read is a saved read operation. This is the most direct architectural lever for improving the quality of agent context. Graftcode reduces this layer significantly for cross-service calls, but even without it, keeping integration code thin and co-located reduces the surface an agent has to explore.

**Write architectural documentation for machines.** Codified context infrastructure, structured artifacts whose primary audience is an AI agent, is becoming a distinct engineering discipline. `ARCHITECTURE.md` files, module ownership documents, and interface contracts that are kept current and machine-readable give agents the architectural context they need without requiring them to infer it from implementation code.

**Treat agent output quality as a codebase metric.** Agent error rates and the amount of exploration an agent does per task are direct measures of how legible the codebase is to it. Codebases where agents repeatedly explore the same files, generate code that fails at runtime, or produce plausible-looking but incorrect implementations are telling you something about context quality, not just about the model.

## What Makes a Codebase Agent-Ready

![](/uploads/context-engineering/image1.png)

An agent-ready codebase shares most properties of a well-architected human codebase, with the context window and agent reliability as explicit design constraints:

**Typed interfaces between modules and services.** Agents read interfaces to understand what a service does. A typed interface is the smallest accurate representation of a service's callable surface. An HTTP client implementation, plus DTO definitions, is the largest and least semantically accurate.

**Compile-time contract enforcement.** Agent-generated code that violates a contract should fail before production. This requires enforced contracts, typed Grafts, gRPC stubs, and TypeScript interfaces, not inferred ones from HTTP client code.

**Minimal integration surface between services.** The less integration boilerplate there is, the less the agent has to read to understand each service interaction, and the fewer opportunities there are for the agent to infer an incorrect contract.

**Clear module boundaries.** Agents work best when they can be scoped to a domain without needing to understand adjacent systems. Clean module boundaries make that scoping possible and keep exploration bounded.

**Consistent conventions.** Agents generalize from patterns. Consistency means reading one example is enough to understand the next. Inconsistency forces you to read more code to understand each new pattern.

Explicit rather than implicit coupling. Agents can't discover hidden coupling until it surfaces at runtime. Explicit coupling, visible in the code the agent reads, is discoverable before generation, not after execution.

These properties aren't new. They're what good software architecture has always required. What AI coding agents add is a concrete, measurable feedback signal: agent error rates, exploration overhead, and runtime failures in agent-generated code are direct measures of how well a codebase embodies these principles.

## Conclusion: Architecture Decisions Are Context Engineering Decisions

Context engineering's most impactful decisions aren't retrieval strategies or prompt formats; they're architectural decisions made long before an agent is involved.

Codebases with typed service interfaces, minimal integration overhead, and compile-time contract enforcement give agents accurate context, bounded exploration, and reliable feedback signals. Codebases with implicit coupling, DTO sprawl, and undocumented integration contracts force agents to read more, infer more, and fail more, not because the models are incapable, but because the context they're working with is noisy, incomplete, and unreliable.

The architecture decisions teams make today are context engineering decisions for the AI agents they'll be working with tomorrow. Audit your service boundaries, enforce contracts at compile time, and reduce the integration layer that forces agents to read plumbing rather than logic. Graftcode handles the last part directly.

## FAQs

### **1. What is context engineering and how does it differ from prompt engineering?**

Prompt engineering focuses on optimizing the instructions and examples given to a model in a single input. Context engineering is broader; it covers the entire environment in which an agent operates: the files it retrieves, the tool definitions it reads, the service interfaces it calls, and the architectural state it navigates. Prompt engineering is about what you ask. Context engineering is about everything the model can see when it tries to act.

### **2. How does software architecture affect AI coding agent performance in practice?**

Architecture determines what the agent reads, how accurately that reflects the system's actual behavior, and what error signals it receives when its generated code is wrong. A well-structured codebase with typed interfaces and minimal integration overhead enables an agent to access the relevant logic with fewer read operations and to obtain accurate contract information. A poorly structured codebase forces the agent to explore more files, infer contracts from plumbing code, and discover errors at runtime rather than compile time.

### **3. What makes a codebase "agent-ready" in practice?**

Typed interfaces between modules and services, compile-time contract enforcement, minimal integration boilerplate, clear module boundaries, consistent conventions, and explicit rather than implicit coupling. These are the same properties that make a codebase maintainable for human engineers, with the additional dimension that they directly improve agent accuracy, reduce exploration overhead, and make contract violations detectable before production.

### **4. Why is compile-time contract enforcement specifically important for AI agents?**

When an agent generates code that calls a service with the wrong method signature or field name, the feedback loop matters. In a REST-based codebase, the error surfaces at runtime, after the agent has potentially generated substantial downstream code based on the wrong assumption. In a codebase with compile-time enforcement, the error surfaces immediately as a type error the agent can act on before generating anything else. Tighter feedback loops produce more reliable agent output. Graftcode provides this enforcement for cross-service calls; incompatibilities surface when the team applies the Graft package update before any downstream code is generated against the wrong contract.

### **5. How does Graftcode specifically improve context quality for AI agents?**

Graftcode replaces the integration code between services, HTTP clients, DTOs, and serialization logic with typed Grafts installed via a package manager. For an agent, this means reading a method signature rather than retrieving and parsing multiple integration files to understand a single service call. It also means the callable surface is semantically accurate; a method signature communicates what a service does, not how it communicates, and contract changes surface as compile-time errors rather than silent runtime failures. In agentic workflows that orchestrate calls across multiple services, these improvements compound with each service interaction.
