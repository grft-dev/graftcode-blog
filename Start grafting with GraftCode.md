---
title: Start grafting with GraftCode

---

# 🌱 Start Grafting with GraftCode — The Future of App Integration
*(Beginner’s Guide to Multi-Language Grafting)*

> **Curious what this is?**  
It’s something new, fresh, and ambitious — a technology that might redefine how developers connect apps and services. Instead of endless API boilerplate, you can connect software parts in seconds, like snapping Lego bricks together.

💡 *Fun fact*: The word **“grafting”** comes from gardening. It’s about joining plants so they grow as one. **GraftCode** does the same for your code — letting your apps share logic and communicate as if they were part of one healthy system.

---

## 🚀 Why We Built GraftCode
Modern development feels like juggling chainsaws:
- Frontends, mobile apps, backends, microservices… and then some legacy thing nobody wants to touch.
- Each connection means: REST endpoints, DTOs, clients, breaking changes, hours of sync meetings.

> Tired? So were we. That’s why we built **GraftCode** — to cut the glue code and let devs focus on actual features.

With GraftCode, you:
- **Expose public methods** — no controller jungle.
- **Install auto-generated clients** — no codegen spaghetti.
- **Call them like local code** — no debugging HTTP payloads at 2 AM.

---

## ⚡ How It Works (Simplified)
Think of it like this:

**Old way** → Write APIs → Define DTOs → Generate SDKs → Update forever.  
![image](https://hackmd.io/_uploads/SJj4OOWTxx.png)

**New way** → `public methods` + `npm install my-backend` → done.

![image](https://hackmd.io/_uploads/r1wHuOZ6lx.png)

Under the hood is **Hypertube™** — our superfast binary protocol:
- 5–10× faster than REST/gRPC
- Up to 60% less integration code
- Works across languages and platforms

---

## 🌍 Polyglot Superpower
With GraftCode, you’re no longer locked to a single language or tech stack. You can easily mix and match languages and modules in one app. Here’s a simple example: calling a Python AI module from a Node.js service, without writing REST APIs or gRPC.

**Python: AI Module**
```Python
# ai_model.py
from graftcode import expose

@expose
def predict_days(features: dict) -> int:
    # Simple model simulation
    complexity = features.get("complexity", 1)
    return complexity * 2

```
* @expose marks the function as public, making it available to other languages.

**Node.js: Calling the Python Method**
```nodejs
import { importGraft } from "graftcode";

async function main() {
    // Install the Python module as a “graft”
    const ai = await importGraft("python-ai-module");

    const days = await ai.predict_days({ complexity: 3 });
    console.log(`Predicted days: ${days}`);  // → Predicted days: 6
}

main();

```
* Node.js calls the Python function as if it were a local function.
* Hypertube™ handles cross-language communication, with no HTTP or JSON serialization needed.

**Outcome**

* You can easily combine modules from Python, Node.js, Go, or .NET.
* No more glue code or integration headaches.
* Calls between services feel instant, as if all components run in a single process.

![programmerhumor-io-programming-memes-480af10e8c2ee94](https://hackmd.io/_uploads/B1hUK_WTlx.jpg)

---

## 🏛️ Evolutionary Architecture — Without Fear
Projects start small but can grow into monsters. With GraftCode, you can:
- Start as a **monolith** for speed.
- Switch to **microservices** when scaling — no rewrites.
- Change connection type (in-process ↔ network) with a config tweak.

![image](https://hackmd.io/_uploads/BJIhKdW6lg.png)


---

## ⚡ Compare Performance

As apps scale, service communication often becomes a bottleneck. REST and gRPC add overhead — JSON or Protobuf serialization, extra controllers, DTOs, and heavy network payloads. This means slower calls, more code to maintain, and higher cloud bills.

**GraftCode solves this with its Hypertube™ technology.**  
Hypertube is a native binary channel that connects directly at runtime — no HTTP, no gRPC, no extra layers. Calls feel like local method invocations but travel across services at **5–10× the speed of REST**, with **30–60% less integration code**.

In real-world tests:  
- REST: ~12–15 ms per call  
- gRPC: ~7–8 ms  
- **GraftCode: ~2–3 ms**  
- Throughput jumps to 90k+ requests/sec (vs ~20k REST, ~40k gRPC)

This speed directly reduces compute time and costs. One Azure benchmark at **5,000 req/sec** saved **4 ms per call**, cutting about **$136k annually** and lowering energy use and CO₂ emissions.

![image](https://hackmd.io/_uploads/H1OA5dWTxl.png)
![image](https://hackmd.io/_uploads/HJKkidW6ll.png)

As you can see on
By eliminating unnecessary layers, GraftCode makes apps faster, cheaper to run, and more sustainable — while keeping your business logic clean and decoupled.



---

## 🛤️ Easy Migration Path
You don’t have to go all in on day one:
- Build **new features** with GraftCode.
- Wrap **existing services** over time.
- Drop the legacy layers when ready.

![image](https://hackmd.io/_uploads/Hy1NjObTxg.png)



---

## ✨ Why People Love It
> “We shipped weeks faster — no more waiting on API specs.”

> “Our cloud bill dropped after cutting CPU time.”

> “Finally — a clean architecture diagram that isn’t a spaghetti monster.”

---

## 🚀 Your Next Step
Install a graft, try connecting one service, and see how easy it is. Then go build something amazing.

🔥 *Less glue code, more shipping.*
