---
title: Calling Services Over Azure Service Bus Plugin Without Writing the Client Layer
slug: azure-service-bus-plugin
date: 2026-10-04T05:04:45.404Z
author: Adam Wasielewski
category: General
readingTime: 10
coverImage: /uploads/azure-service-bus-plugin/Azure Service Bus Messaging Flow.png
---

## TL;DR

* Two queues and a session filter are the minimum anyone needs to get request/reply working safely over Service Bus, and most teams underbuild it until a reply routes to the wrong caller under load.
* Errors don't normally survive a trip through a broker cleanly; a dropped message just looks like a timeout, so most hand-written consumers can't actually tell "it failed" apart from "it's slow."
* Graftcode's Service Bus plugin handles both jobs for you: request/reply over queues when a method returns something, one-way over a topic when it doesn't, chosen entirely by config, not by writing two different clients.
* The session and correlation logic, plus a one-byte error signal on the response, live inside the plugin, so a Graft call fails the same way a local function call fails, not the way a broker timeout usually does.
* Nothing about Service Bus itself changes; durability, ordering, and dead-lettering all stay exactly where they are. This only removes the client code sitting on top of it.

Every team that puts Azure Service Bus in front of a service does so for the same reason: reliable delivery without running the infrastructure themselves. Service Bus doesn't hand you the code that turns a delivered message into a method call with a typed result; that part stays with the consuming team, and it's easy to underestimate until a reply goes to the wrong client or a failure disappears into a timeout with no clear cause. This piece walks through what that work actually involves, where it gets expensive, and how things change once a plugin handles it instead.

## What Is Azure Service Bus

Azure Service Bus is a managed message broker; it sits between services, accepting messages from a producer and delivering them to a consumer, without either side needing to be online or reachable at the same moment. It supports two core patterns: **queues**, where each message goes to exactly one consumer, and **topics with subscriptions**, where each message can go to multiple independent consumers, each with their own subscription.

Teams reach for it for the same reason they choose any managed broker: reliable delivery, durability, and platform-handled ordering, without running that infrastructure themselves. What it doesn't hand you is the code that turns a delivered message into an actual method call with a typed result; the consuming team still has to write that.

## What a Correct Service Bus Call Requires

Getting a request/reply call working correctly over Service Bus means solving three separate problems, and it's easy to get the first two working in a demo and only discover the third one is broken once two callers hit the service at the same time. A request has to reach the right queue. A reply has to find its way back to the specific client that asked for it, not just whichever client happens to be listening on the reply queue. And a server-side failure has to reach the caller as something recognizable as a failure, not as a message that quietly never shows up.

That middle problem is where people go wrong. Without a session strategy, two clients sharing a reply queue can end up reading each other's responses, or one client blocks while waiting for a reply that has already been delivered elsewhere. Here's roughly what solving it by hand looks like:

```python
# Everything below exists only to get a reply back to the right caller; none of it is business logic
from azure.servicebus import ServiceBusClient, ServiceBusMessage
import uuid

def call_physics_calculator(payload: bytes) -> bytes:
    session_id = str(uuid.uuid4())
    with ServiceBusClient.from_connection_string(CONN_STR) as client:
        sender = client.get_queue_sender(queue_name="myqueue")
        receiver = client.get_queue_receiver(
            queue_name="myqueue.reply", session_id=session_id
        )
        message = ServiceBusMessage(payload)
        message.reply_to_session_id = session_id
        sender.send_messages(message)

        reply = receiver.receive_messages(max_wait_time=30)
        if not reply:
            raise TimeoutError("no response within timeout")
        return b"".join(reply[0].body)
```

None of this describes what the physics calculation actually does. It's session setup, a reply filter, and a timeout, three things a team has to get right before any real logic runs, and get wrong silently if they don't.

## Where the Integration Work Piles Up

The shape of this work changes depending on how a service consumes Service Bus, but it doesn't get smaller.

| Pattern                   | What has to be written by hand                                                                   | Where it breaks quietly                                                                                                      |
| ------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Simple queue consumer     | SDK setup and message deserialization before any downstream call                                 | The consumer's expected schema can drift from what the producer actually sends, with no build-time warning                   |
| Request/reply over queues | All of the above, plus session generation, a session-filtered receiver, and correlation matching | A missed session filter or a mismatched correlation field means a reply lands with the wrong caller, or never arrives at all |
| Topics and subscriptions  | SDK setup and deserialization repeated per subscription                                          | Every subscription reimplements the same consumer logic even when the message shape is identical across all of them          |

![](/uploads/azure-service-bus-plugin/Where-Integration-Work-Piles-Up.png)

Request/reply is the expensive one. Sessions exist so a reply queue can be shared safely across many callers at once, but that safety only holds if the session filter, the reply-to field, and the correlation match are all correct, and a mistake in any one of them doesn't throw an error. It just looks like latency.

## The Cost of Writing This by Hand

Backend teams commonly lose 30 to 40 percent of engineering time to integration code in general; HTTP clients, DTO drift, SDK versioning, and Service Bus consumers carry that same tax plus a few costs specific to the broker:

* **Session plumbing per caller**: every service that needs request/reply correlation reimplements session generation and session-filtered receivers on its own, even when three other teams are calling the same downstream service
* **Silent correlation bugs**: a session or correlation mismatch doesn't throw; it just means a caller times out waiting for a reply that already went somewhere else
* **Manual timeout and retry logic**: every hand-written client ends up with its own version of this, rarely consistent with how any other caller of the same service handles it
* **Schema drift above the broker**: nothing enforces that a consumer's expected message shape still matches what the producer sends, the same problem REST has, just moved to sit above Service Bus instead of behind an endpoint

None of this shows up on an architecture diagram. It shows up in how long it takes to debug a caller that's timing out for no visible reason, and in how many times different teams rewrite the same session-handling code for the same dependency.

## A Cleaner Approach: The Service Bus Plugin

Graftcode is a runtime-level integration platform, and its Azure Service Bus plugin lets a Graft call route through Service Bus without writing any session or correlation code by hand. The architecture underneath is the same three pieces Graftcode uses everywhere: a **Graft**, a strongly typed interface installed via a package manager that mirrors the provider's public methods; **GraftConfig**, an internal class of each Graft that decides whether a call runs in-process or through a configured transport; and the **Graftcode Gateway**, which runs and hosts the provider service itself.

On top of that, the plugin offers two modes, set purely by configuration: request/reply over queues for methods that return something, and one-way over a topic for methods that don't.

![](/uploads/azure-service-bus-plugin/Message-Flow_Queue-Replies-and-Topics.png)

Configuring the client side of a request/reply call looks like this:

```python
graft.nuget.PhysicsCalculator.GraftConfig.SetConfig("""
{
  "configurations": {
    "graft.nuget.PhysicsCalculator": {
      "runtime": "netcore",
      "stateless": true,
      "plugin": {
        "name": "ServiceBusPlugin",
        "connectionString": "Endpoint=sb://<namespace>.servicebus.windows.net/;SharedAccessKeyName=<keyName>;SharedAccessKey=<key>",
        "queue": "myqueue",
        "replyQueue": "myqueue.reply",
        "rpcTimeoutMs": 30000
      }
    }
  }
}
""");
```

Everything the earlier snippet did by hand now happens inside the plugin. Session generation reuses the request's own correlation ID, confirmed directly from the plugin's source, and the reply receiver attaches a `com.microsoft:session-filter` scoped to that session, the same filter symbol the official .NET and Java SDKs use. Matching is a little more forgiving than a single rigid field too; either the correlation ID or the message ID is accepted. A server-side failure reaches the caller the same way a local exception would: the response's first byte doubles as a sentinel; `0xFF` means everything after it is an error message, thrown client-side as a native exception; anything else means a normal response.

The one-way mode skips sessions and correlation entirely, since there's no reply expected. A client publishes to a topic and returns immediately, and the server consumes from a topic subscription without sending anything back. Multiple subscriptions on the same topic each receive their own independent copy of the message; this is native pub/sub, not something built on top of the plugin.

## How This Fits What You Already Have

This doesn't ask a team to change how they use Service Bus. Whatever namespace, sessions, and dead-lettering configuration already exists stays exactly as it is, and the plugin is adopted per Graft, usually starting with whichever service pair currently has the most hand-written session and correlation code.

It also doesn't touch anything Service Bus already does well; durability, ordering, and dead-lettering are unaffected. What it removes is the layer above that: client setup, session handling, correlation matching, and the deserialization a consumer would otherwise write by hand.

Local development has two genuinely separate options worth keeping apart. The plugin explicitly supports the official Service Bus emulator; setting `useDevelopmentEmulator: true` points it at a locally running emulator instead of a live Azure namespace, still something running locally, just not a cloud dependency. Separately, setting `GraftConfig` to in-process mode skips the broker entirely; the call never leaves the process, making it faster for pure logic iteration and removing the need for the emulator or a namespace.

## A Practical Comparison

| Without the plugin                                                            | With the plugin                                                                                    |
| ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Manual session generation and a session-filtered receiver, written per caller | Session handling lives inside the plugin                                                           |
| Manual correlation matching between request and reply                         | Handled automatically, with a documented fallback match                                            |
| A dropped or mismatched reply just looks like a timeout                       | Errors cross the wire as a defined signal and throw as a native exception                          |
| Separate consumer code per subscription, even for identical message shapes    | One-way mode consumes through the same configured plugin regardless of subscription count          |
| Local testing needs a real namespace or a manually configured emulator client | GraftConfig in-memory mode skips the broker, or the plugin's own emulator support handles the rest |

This isn't a claim that the plugin removes coupling between services; that depends on how boundaries are drawn, not on what sits underneath a call. What it removes is the session, correlation, and client code that adds overhead on top of a service boundary that's already reasonably drawn.

## Conclusion: The Broker Was Never the Hard Part

Service Bus solves delivery well; durability, ordering, and dead-lettering are the broker's job, not the consuming service's. What it never solved was the code a team writes to turn a delivered message into a typed method call, and that code carries real session and correlation risk regardless of how reliable the underlying broker is.

Graftcode's Service Bus plugin addresses that layer specifically, without touching how a team already uses Service Bus. Existing namespaces, sessions, and dead-lettering stay in place, and the plugin is adopted one Graft at a time, starting wherever hand-written session and correlation code is currently most costly. To see how it fits your setup, explore [Graftcode](https://www.graftcode.com/) or go straight to the [Graftcode Academy](https://academy.graftcode.com/) for the full configuration reference.

## FAQs

### **1. Does the plugin change any of Service Bus's own delivery guarantees?**

No. Durability, ordering, dead-lettering, and session-based delivery are all Service Bus features and stay exactly as configured. The plugin only removes the hand-written client, session, and correlation code that would otherwise sit above those guarantees.

### **2. How does a server-side exception actually reach the calling code?**

The response payload carries a one-byte sentinel; if the first byte is `0xFF`, everything after it is read as an error message and thrown client-side as a native runtime exception. Any other first byte means a normal successful response, so error handling on the calling side looks the same as it would for a local method call.

### **3. Can a service use both request/reply and one-way calls at the same time?**

Yes. The mode is chosen per Graft configuration, not system-wide. One Graft can be configured for request/reply over queues while another is configured for one-way over a topic, depending on whether each method returns a value.

### **4. Is there a message size limit to plan around?**

Yes. Azure Service Bus caps standard-tier messages at 256 KB and premium-tier messages at 1 MB, and the plugin matches that 1 MB premium limit. A method returning a large object can hit this in a way it wouldn't over a direct connection. For large payloads, either move to the premium tier or keep that call on a direct-connection transport instead.

### **5. Does adopting this plugin mean migrating everything onto Service Bus?**

No. It's built to extend an existing setup incrementally; a team can configure it for a single Graft where session and correlation overhead is highest, while every other service pair keeps running however it already does. You don't need to migrate the entire system to adopt it for a single service pair.
