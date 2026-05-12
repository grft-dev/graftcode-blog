---
title: Make Your Day new one!
coverImage: 'https://academy.graftcode.com/_next/image?url=%2Fquick-start-image.png&w=3840&q=75'
---

What is Graftcode

Graftcode is a runtime-level integration platform that allows software systems to communicate using strongly typed method calls, even when those systems run in different processes, on different machines, or in different programming languages.

Instead of designing endpoints, defining schemas, and generating clients, you expose a programming interface—classes, methods, and types—and Graftcode makes that interface callable from other applications as if it were local code.

From a developer’s point of view, a remote service becomes just another dependency in the project.

A familiar problem, repeated everywhere

When two pieces of code live in the same application, they can call each other directly.

When they don’t, we usually introduce an API.

That API then grows a surrounding ecosystem:

endpoints and routes

request and response models

client libraries

versioning rules

compatibility concerns

Over time, a significant part of the system exists not to implement business behavior, but to translate between systems.

This pattern has repeated itself for decades—from early distributed systems, through SOAP and REST, to modern gRPC. Each generation improved performance or ergonomics, but the underlying idea remained the same: communication lives outside the code.

Graftcode challenges that assumption.

A different way to think about communication

Graftcode treats communication as a programming concern, not a networking one.

If one piece of software needs to call another, you define what can be called—methods, arguments, return types—and let the runtime handle how that call happens.

The result feels closer to using a shared library than consuming an external service:

you work with real types, not schemas

you call methods, not URLs

incompatibilities show up early, not at runtime

Under the hood, the call may cross process boundaries or machines, but that detail stays out of your application code.

What Graftcode actually provides

Graftcode introduces a small set of core building blocks that work together:

Grafts – strongly typed clients generated from public method signatures

Graftcode Gateway – a runtime host that exposes those signatures and executes calls

Hypertube™ – a runtime-level bridge that connects different runtimes efficiently

Graftcode Vision – live, always-up-to-date documentation derived from real code

Together, they form a system where interfaces are discovered, clients are generated, and calls are executed automatically—without asking developers to design or maintain a separate communication layer.

What Graftcode replaces—and what it doesn’t

It’s important to be precise here.

Graftcode can replace:

REST-based service APIs

gRPC-based service communication

custom client libraries built around messaging or RPC

Graftcode does not replace:

your infrastructure

your security model

your monitoring and observability tools

your deployment strategy

It works with existing load balancers, proxies, logging systems, and tracing tools. Communication happens over standard, well-understood transports, and all execution remains under your control.

How Graftcode fits into an architecture

Graftcode sits between business logic and infrastructure, removing the need to explicitly design how services talk to each other.

During development:

public method signatures are analyzed

typed clients are generated on demand

those clients are distributed through standard package managers

At runtime:

calls are executed directly between runtimes

execution can happen in memory, over TCP/IP, or via WebSocket

routing, security, and transport are configured outside the code

This separation allows teams to evolve architecture independently of application logic.

When Graftcode is a good fit

Graftcode works particularly well when:

systems are composed of multiple services or modules

teams want strong typing across service boundaries

communication patterns change over time

integration code has become a maintenance burden

It can be introduced gradually. Existing APIs can continue to operate alongside Grafts, and adoption can start with a single service or use case.

The core idea

At its heart, Graftcode is based on a simple observation:
