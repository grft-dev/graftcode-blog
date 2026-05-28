---
title: Start grafting with GraftCode
author: Adam Wasielewski
category: General
readingTime: 7
coverImage: /uploads/quick-start-image.webp
---

# 🌱 Start Grafting with Graftcode

*(Beginner’s Guide to Multi-Language Grafting)*

Software teams spend too much time wiring systems together: REST endpoints, DTOs, client libraries, SDK updates, serialization bugs, versioning issues, and integration meetings.

Graftcode changes that.

Instead of building and maintaining integration layers, you expose public methods, install them as Grafts through your regular package manager, and call them from another application as if they were local code.

That means less integration code, fewer moving parts, and faster delivery across polyglot systems.

💡*Fun fact: The word “grafting” comes from gardening: joining two plants so they grow as one. Graftcode applies the same idea to software: separate services, modules, and languages can work together as one connected system.*

***

## 🚀 Why Graftcode Exists

Modern applications rarely live in one language or one runtime anymore.

You might have:

* A React, Vue, or Angular frontend
* A Node.js or .NET backend
* A Python AI module
* A Java service
* A legacy system that still runs something business-critical
* A growing need to expose services to AI agents through MCP

Traditionally, every connection requires extra integration work:

* REST or gRPC endpoints
* DTOs and serialization models
* Client SDKs
* API documentation
* Versioning and compatibility handling
* Tests for code that is not really business logic

Graftcode removes much of that integration layer.

With Graftcode, you can:

* **Expose public methods** from your service
* **Run them through Graftcode Gateway**
* **Install a strongly typed Graft** in another project with just **one command**
* **Call remote methods like local functions**
* **Keep IDE autocompletion and type safety**
* **Switch between local and remote execution without rewriting integration code**

***

## ⚡ How It Works

The traditional way looks like this:

**Old way:**\
Write API → define DTOs → generate SDK/client → handle serialization → maintain versions → repeat

![](/uploads/upload_9d40e513206547d9a9ea3bc4f1d10266.png)

**New way with Graftcode:**\
Expose public methods → install a Graft → call methods directly

![](/uploads/2.png)

Under the hood, Graftcode uses **Hypertube™**, a native runtime communication layer based on binary messaging. It allows applications written in different languages to communicate directly, without forcing developers to build REST or gRPC layers first.

The developer experience is simple:

1. Write normal code in your language.
2. Expose the methods you want to make available.
3. Run the service with Graftcode Gateway.
4. Install the generated Graft in another application.
5. Call the exposed methods like local code.

***

## 🌍 Polyglot Superpower

Graftcode is especially useful when your stack is not limited to one language.

For example, you can call a Python AI module from a Node.js service without building a REST API or writing a custom client.

### Python: AI Module

```python
# ai_model.py
from graftcode import expose

@expose
def predict_days(features: dict) -> int:
    complexity = features.get("complexity", 1)
    return complexity * 2
```

The exposed method becomes available to other applications through Graftcode with just one simple installation command:

```bash
npm install --registry https://grft.dev/<graft_registered_url> @graft/pypi-ai-example
```

And then you can call it from your Node.js application:

### Node.js: Calling the AI Python Method

```javascript
import { importGraft } from "graftcode";

async function main() {
    const ai = await importGraft("python-ai-module");

    const days = await ai.predict_days({ complexity: 3 });
    console.log(`Predicted days: ${days}`);
}

main();
```

From the Node.js side, the Python function behaves like a regular imported method.

No REST controller.\
No JSON payload debugging.\
No hand-written client.\
No duplicated DTOs.

![](/uploads/2.5.jpg)

***

## 🏛️ Evolutionary Architecture Without the Rewrite

A common problem in software architecture is choosing too early between a monolith and microservices.

Start with a monolith, and scaling later can be painful.\
Start with microservices, and you may pay the operational cost before you need it.

Graftcode supports a more evolutionary approach.

You can:

* Start with modules running close together
* Keep clear boundaries between business capabilities
* Move selected modules into separate services when needed
* Change how components are connected through configuration
* Avoid rewriting the integration layer every time your architecture evolves

That makes Graftcode useful for modular monoliths, microservices, legacy modernization, and AI-ready systems.

![](/uploads/3.png)

***

## 🤖 AI-Ready by Design

AI coding tools are getting better at writing business logic, but integration is still where many projects slow down.

APIs, clients, wrappers, schemas, and middleware increase complexity. They also make systems harder for AI agents to understand and safely orchestrate.

Graftcode helps by exposing clean, strongly typed methods directly.

Even better, services exposed through Graftcode can also become available through **MCP**, making it easier for AI agents to call existing business logic without building another integration layer.

This means your existing backend can become more AI-ready without rebuilding it around a new protocol from scratch.

***

## ⚡ Performance and Efficiency

As applications scale, service communication becomes expensive.

REST and gRPC are proven technologies, but they still introduce layers: HTTP, serialization, generated clients, payload transformation, and extra code paths. That adds latency, CPU usage, and maintenance cost.

Graftcode’s Hypertube™ approach reduces that overhead by using a native runtime bridge and binary communication.

The result is:

* Faster service-to-service communication
* Less integration code
* Lower CPU overhead
* Cleaner application boundaries
* Fewer generated or hand-written clients to maintain

Performance will always depend on the workload, deployment model, and infrastructure, but the direction is clear: removing unnecessary layers makes systems simpler and more efficient.

![image](https://hackmd.io/_uploads/H1OA5dWTxl.png)

![](/uploads/5.png)

***

## 🛤️ Easy Migration Path

You do not need to rebuild your whole system at once.

A practical adoption path looks like this:

1. Pick one internal service or module.
2. Expose a few public methods with Graftcode.
3. Consume them from another app using a Graft.
4. Compare the integration effort with your current REST or gRPC approach.
5. Expand gradually where it makes sense.

You can start with new features, wrap existing services over time, and remove legacy integration code only when the team is ready.

![](/uploads/6.png)

***

## 📚 Learn Graftcode in the Academy

The easiest way to start is through **Graftcode Academy**:

👉 [Graftcode Academy](https://academy.graftcode.com/)

There you will find tutorials, articles, and code samples that help you get started with Graftcode.

If you want to jump straight into building, start here:

👉 [Quick Start](https://academy.graftcode.com/quick-start)

The Quick Start path shows how to use Graftcode with supported technologies and common integration scenarios, including frontend-to-backend communication, backend service exposure, microservice connections, cross-language modules, MCP exposure, and switching between monolith and microservice setups.

For deeper technical details, concepts, and reference material, use:

👉 [Documentation](https://academy.graftcode.com/documentation)

***

## 🚀 Your Next Step

Pick one small integration in your project.

Expose a method.\
Install a Graft.\
Call it like local code.

That is the fastest way to understand what Graftcode changes.

Less integration code.\
Cleaner architecture.\
More shipping.
