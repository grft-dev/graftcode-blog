---
title: 'Protocol-Agnostic Microservices: Switching Between TCP, WebSocket, and Broker Transport Without Rewriting Code'
slug: protocol-agnostic
author: Adam Wasielewski
category: General
readingTime: 10
coverImage: /uploads/protocol-agnostic/image1.png
---

## TL;DR

* Every service-to-service call has two separate layers: what you're asking for (the interface) and how the request physically travels (the transport: REST, gRPC, RabbitMQ, a Kafka topic, raw TCP). Most codebases accidentally collapse these into one thing.
* Switching a call from synchronous REST to an async event, or from one broker to another, usually means rewriting the client, the DTOs, the serialization, and the error handling, not because the business logic changed, but because the protocol was never separated from it.
* Being protocol-agnostic means a service's interface remains stable regardless of which transport carries a given call; switching transport should be a configuration change, not a rewrite.
* Graftcode does this concretely: `GraftConfig` determines whether a Graft call runs in-memory, over a direct connection, or through a broker-mediated plugin, and the calling code never changes.
* This piece stays scoped to what's actually confirmed: a direct-connection transport swap and a RabbitMQ-mediated transport are both real, documented mechanisms. An equivalent to Kafka has not been confirmed to exist, and this piece won't claim one.

Most teams don't set out to build a system that mixes protocols. It happens gradually: REST for the public API, because that's what every client expects; an event queue for anything that can happen asynchronously; maybe a direct RPC-style call for the one internal path that needs to be fast. Each choice makes sense on its own. The problem shows up later, when one of those choices needs to change, a synchronous call creates a bottleneck, or the team migrates from one broker to another, and changing it affects far more of the codebase than anyone expected.

This guide starts with what a protocol is at the level of a single service call, explains why changing one usually cascades into a rewrite, and ends with what it takes to make transport a configuration decision instead of an architectural one.

## What Is a Protocol in a Microservices Call

Every call from one service to another involves two things happening at once, even though they are usually written as a single block of code. There's the **interface**, what you're actually asking for: "get this order's status," "reserve this item," "charge this payment." And there's the **transport**, the mechanism that physically carries that request from one process to another: an HTTP request over REST, a message published to a RabbitMQ queue, an event written to a Kafka topic, a raw socket connection, a gRPC call over HTTP/2.

In most codebases, these two things are written as if they're the same thing. A REST client doesn't just describe "get order status"; it describes an HTTP GET request to a specific path, with a specific header format, and expects a specific JSON shape in response. The interface and the transport are fused into a single function.

This is a completely reasonable way to build a system, and most systems end up using more than one protocol on purpose, not by accident:

* **REST or gRPC** for calls that need an immediate response, a user-facing request that has to complete before the page renders
* **Events over a broker** (RabbitMQ, Kafka) for anything that doesn't need an immediate response and benefits from decoupling: a purchase triggering a notification, an inventory update triggering a re-order check
* **Direct RPC-style calls** for internal, latency-sensitive paths where the overhead of HTTP or a broker round-trip actually matters

None of this is a mistake. Different calls have different requirements, and using one protocol for everything is usually the wrong call. The problem isn't having multiple protocols in a system; it's that the interface and the transport are usually so tightly fused that changing one requires touching the other.

## Why Does Switching Protocols Usually Require a Rewrite

Consider a simple call: checking whether an item is in stock. Here's the same intent implemented three different ways: a REST call, a RabbitMQ-published event with a reply, and a direct RPC-style call. All three are asking the exact same question. None of them are interchangeable as written.

```python
# REST
import requests

def check_stock_rest(sku: str) -> dict:
    response = requests.get(f"http://inventory-service:8080/api/v1/stock/{sku}")
    return response.json()
```

```python
# RabbitMQ: request/reply over a queue
import pika, json, uuid

def check_stock_rabbitmq(sku: str) -> dict:
    connection = pika.BlockingConnection(pika.ConnectionParameters("rabbitmq-host"))
    channel = connection.channel()
    reply_queue = channel.queue_declare(queue="", exclusive=True).method.queue
    correlation_id = str(uuid.uuid4())

    channel.basic_publish(
        exchange="",
        routing_key="stock_check_queue",
        properties=pika.BasicProperties(reply_to=reply_queue, correlation_id=correlation_id),
        body=json.dumps({"sku": sku})
    )
    # ... consume from reply_queue, match correlation_id, parse response
```

```python
# Direct RPC-style call
import grpc
from inventory_pb2_grpc import InventoryStub
from inventory_pb2 import StockRequest

def check_stock_grpc(sku: str):
    channel = grpc.insecure_channel("inventory-service:50051")
    stub = InventoryStub(channel)
    return stub.GetStock(StockRequest(sku=sku))
```

Three completely different implementations, three different sets of dependencies, three different error-handling patterns, three different response shapes to parse. If a team decides to move this call from REST to RabbitMQ because the inventory service is now under enough load that synchronous polling is a problem, every caller of `check_stock_rest` must be identified and rewritten to use the RabbitMQ version instead. The business logic asking the question didn't change at all. Everything else did.

This is the actual cost of protocol coupling: it's not that any one of these implementations is wrong; it's that none of them can become another one without a rewrite, because the protocol was never a separate, swappable layer to begin with.

## How Can a Service Stay Independent of Its Transport

The fix is architectural, not a specific technology choice: separate what a service exposes from how a call reaches it. The "check stock for this SKU" interface should remain stable regardless of whether the request arrives via REST, a broker, or a direct connection. The transport should be a detail configured underneath that interface, not baked into every caller that uses it.

This is genuinely hard to retrofit into an existing system for a straightforward reason: writing the transport directly into the calling code makes it faster to ship initially. Nobody sets out to couple an interface to REST specifically; they write a function that calls an HTTP endpoint because that's the fastest way to get the feature working, and the coupling happens as a side effect of shipping quickly.

Done well, changing a call's transport should look like changing a configuration value, not finding every caller, rewriting the client, redefining the request and response shapes, and re-testing the error handling. The interface a caller depends on stays exactly the same; only what's configured underneath it changes.

## How Does Graftcode Make the Transport Swappable Without Touching Calling Code

Graftcode is a cross-runtime communication layer. Services call each other's public methods directly through automatically generated interfaces called Grafts, installed as typed packages via standard package managers, without hand-written integration code, DTOs, client libraries, or serialization boilerplate.

**Graftcode Gateway** runs the provider service itself; it spins up a runtime for the configured modules and hosts them. It is not positioned between services and does not route calls between them.

```bash
./gg --projectKey <your-project-key> --modules ./inventory_service
```

**Graft** is a strongly typed interface, installed via a package manager, that mirrors the provider's public methods exactly: names, argument types, and return types. The calling code is written once, against this interface.

**GraftConfig** is an internal class of each Graft, and it's the single place transport gets configured. It doesn't change what method is being called, only how the call physically reaches the target service.

```python
const { GraftConfig, InventoryService } = require("@graft/nuget-inventoryservice");

// If GraftConfig.host is never set, it defaults to in-memory automatically
// Local development: in-process, no network call at all 
GraftConfig.host = "inmemory";

// Direct connection: explicitly configured, no broker involved
GraftConfig.host = "wss://inventory-service:9000/ws";
```

The calling code is identical in both cases:

```python
const stock = await InventoryService.getStock(sku);
```

Graftcode also ships an official RabbitMQ transport plugin, letting a Graft call route *through* RabbitMQ instead of over a direct connection; the confirmed reference syntax for this is C#/.NET:

```python
// Broker-mediated, routed through RabbitMQ via Graftcode's RabbitMQ plugin

string configSource = """
{
  "configurations": {
    "graft.nuget.InventoryService": {
      "runtime": "netcore",
      "host": "rabbitmq-host:5672",
      "stateless": true,
      "plugin": {
        "name": "RabbitmqPlugin",
        "queue": "inventory_check",
        "replyQueue": "inventory_check.reply",
        "user": "guest",
        "password": "guest",
        "vhost": "/",
        "rpcTimeoutMs": 30000
      }
    }
  }
}
""";
graft.nuget.InventoryService.GraftConfig.SetConfig(configSource);

// The calling code stays identical to the direct-connection version:
var stock = await InventoryService.GetStock(sku);
```

Whether a call runs in-process during local development, over a direct WebSocket connection in production, or through a RabbitMQ request/reply queue because the team decided to route it through the broker instead, the caller reads the same method call. The transport lives entirely in configuration.

![](/uploads/protocol-agnostic/image2.png)

## The Transport Modes Graftcode Supports Right Now

It's worth being precise about what "protocol-agnostic" means here, rather than letting the claim outpace what we actually support. We've confirmed and documented several transport modes. One notable option isn't supported yet, and we won't claim otherwise.

| Transport                                               | Status                                                                                                                                                                |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| In-memory (no `GraftConfig.host` set)                   | Confirmed, the default when no host is configured, used throughout our documentation                                                                                  |
| Direct connection, WebSocket                            | Confirmed, verified across multiple languages                                                                                                                         |
| Direct connection, TCP                                  | Confirmed                                                                                                                                                             |
| Direct connection, HTTP/2                               | Confirmed                                                                                                                                                             |
| RabbitMQ-mediated, via the official RabbitMQ plugin     | Confirmed, a documented plugin (`graftcode-plugins/rabbitmq`) with a real request/reply queue mechanism, verified syntax in C#/.NET                                   |
| ServiceBus-mediated, via our official ServiceBus plugin | Confirmed. Supports request/reply over queues (session-based correlation) as well as one-way fire-and-forget over topics for void methods. Verified syntax in C#/.NET |

This matters because a "protocol-agnostic" claim is easy to overstate into "works with any broker," which isn't accurate today. What's true is narrower and still genuinely useful: a Graft call can move between in-process execution, a direct connection, and a RabbitMQ-mediated queue without the calling code changing. Extending that to other brokers is a reasonable direction, not a current fact.

## What Does a Protocol-Agnostic Architecture Look Like End to End

Pulling this together, a system where transport is genuinely decoupled from interface has:

* **Interface and transport as separate concerns**: a service's callable methods don't reference HTTP verbs, queue names, or wire formats anywhere in their calling.
* **Transport as configuration, not code**: moving a call from a direct connection to a broker-mediated one is a config change, not a search-and-replace across every caller.
* **New transports that don't require touching existing callers**: adding a new way to reach a service shouldn't mean rewriting the services that already call it successfully.
* **Local development that doesn't require production infrastructure**: testing a call against an in-memory target shouldn't require setting up a broker, a message queue, or any other transport-specific infrastructure just to validate business logic.

None of this requires abandoning REST, gRPC, or any broker already in use; it's a question of where the transport decision lives, not which transport is correct.

## Conclusion: The Transport Shouldn't Decide Your Architecture

Protocol choice is a real engineering decision; REST, events, and direct calls each solve different problems, and using the right one for a given call is good design, not a compromise. The problem isn't the choice itself. It's that most systems bake that choice directly into the calling code, so a decision that should be reversible, because load patterns change, because a team migrates brokers, because a synchronous call becomes an async one, ends up requiring a rewrite instead of a configuration change.

Separating a service's interface from its transport is what makes that decision reversible. Graftcode does this through GraftConfig: the calling code stays identical whether a call runs in-process, over a direct connection, or through a RabbitMQ-mediated queue. What's actually confirmed today is scoped to those two specific transport modes, not a blanket claim across all brokers.

To see how Graftcode handles transport configuration for your services, explore [Graftcode](https://www.graftcode.com) or go straight to the [Graftcode Academy](https://academy.graftcode.com) to get started.

## FAQs

### **1. What is the difference between a service's interface and its transport protocol?**

The interface is what a service exposes: the methods it offers and what they do, independent of how a request reaches them. The transport is the mechanism that carries the request: REST over HTTP, a message on a broker, or a direct socket connection. Most codebases couple the two by writing transport details directly into the calling code, which makes changing the transport later expensive.

### **2. Does switching from REST to an event-driven architecture always require rewriting service calls?**

In a typical codebase, yes; the HTTP client, DTOs, and response handling are usually REST-specific and don't translate directly to a message-broker pattern. If the interface and transport are architecturally separated, moving a call to an event-driven pattern means reconfiguring how that call is routed rather than rewriting the call itself.

### **3. How does GraftConfig let a service call switch transport without code changes?**

GraftConfig is an internal class for each Graft, and it's the single place where transport is configured: via an environment variable, a config file, or a value set directly on the Graft. It determines whether a call runs in-process, over a direct connection, or through a broker-mediated plugin. The calling code always uses the same method call, regardless of what's configured underneath.

### **4. Does Graftcode support switching transport to Kafka the same way it does for RabbitMQ?**

Not currently confirmed. Graftcode has a documented, official plugin for RabbitMQ that routes a Graft call through a request/reply queue pair. There is no equivalent documented plugin for Kafka, and Kafka's architecture, a pull-based log without native request/reply semantics, means a RabbitMQ-style plugin can't be assumed to work the same way for Kafka without separate confirmation.

### **5. What are the practical benefits of decoupling a service's interface from its transport?**

The main benefit is that transport decisions become reversible. A team can develop against an in-process target locally, deploy against a direct connection in production, and later move a specific call through a broker if load patterns change, all without touching the code that makes the call. It also means testing business logic doesn't require standing up the actual transport infrastructure (a broker, a message queue) just to validate a call locally.
