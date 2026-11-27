---
title: 'Best Tools to Reduce Integration Boilerplate: 8 Options for Generating Clients and Stubs'
slug: integration-boilerplate-tools
readingTime: 20
coverImage: /uploads/integration-boilerplate-tools/01-banner.webp
seo:
  metaTitle: 8 Best Tools to Reduce Integration Boilerplate (2026)
  metaDescription: 'Compare 8 tools that generate API clients and stubs, including Graftcode, OpenAPI Generator, Kiota, gRPC, Refit and tRPC, with pros, cons and pricing.'
---

## TL;DR

* Integration boilerplate is the client classes, data transfer objects, serialization code, and transport wiring that surround a service call without containing business logic, and the same code is repeated for every pair of services that communicate.
* Graftcode leads this list because it generates the caller-side package from the provider's own module, so the client wrapper, models, and invocation code are produced instead of written, and it can run alongside existing REST APIs for callers that can install a generated package.
* The other seven tools fall into three groups: spec-first generators such as [OpenAPI Generator](https://github.com/OpenAPITools/openapi-generator) and Kiota, schema generators for gRPC and GraphQL, and declarative or inferred clients in Refit, Spring HTTP Service Clients, and tRPC.
* Seven of the eight tools are open source under Apache 2.0 or MIT licenses, and Graftcode offers a free standalone Gateway, though its paid plans are not published.
* Language scope narrows the choice quickly: Refit is for .NET, Spring HTTP Service Clients for Java with Spring, tRPC for TypeScript monorepos, and GraphQL Code Generator for GraphQL APIs, while OpenAPI Generator, Kiota, gRPC, and Graftcode work across several languages.

Every service that calls another service carries code that has little to do with the problem the service solves. A developer writes a client class, maps responses into models, serializes arguments, attaches credentials, handles timeouts, and keeps all of it in step with the provider as the provider changes. Each of these tasks is small, but they repeat for every pair of services, and the repetition grows as a system adds services, languages, and teams. Tooling for this problem has existed for years, and the options differ in where the contract lives and in who writes what.

This guide compares eight tools that reduce integration boilerplate. Each entry covers what the tool does, its strengths and weaknesses, and its pricing, with a comparison table and a decision guide near the end. Graftcode comes first because it generates the caller side from the provider's own module rather than from a separate description. The remaining seven are ordered by approach, because the best fit depends on where a team's contract already lives.

## What Is Integration Boilerplate in Service-to-Service Code?

Integration boilerplate is the repetitive code a service needs in order to call another service or API, apart from the business logic of the call itself. When an orders service needs a figure from a metering service, the business question takes one line: ask for the result. The surrounding code is the client class, the request and response models, the serialization and deserialization, the transport configuration, the authentication headers, and the error handling. That surrounding code is the boilerplate.

Boilerplate differs from the integration itself. The integration is the decision that two services communicate, together with the contract they agree on. Boilerplate is the mechanical code that expresses that contract in each caller's language and framework. Because the provider already defines the contract once, tools can often produce the mechanical part from it, and that observation is the basis of every approach in this guide.

One small example runs through the guide: a metering service that exposes a method taking two numbers and returning a number, and a usage service written in Python that calls it. The example stays small so that the surrounding code remains visible. The next diagram shows the layers around that one call, and the section after it walks through them.

![Six layers of boilerplate around a single service call](/uploads/integration-boilerplate-tools/02-boilerplate-layers.webp)

## Where Integration Boilerplate Accumulates in a Single Service Call

Six layers account for most of the boilerplate in a typical call between two services. Each layer is small on its own, and each one is repeated for every remote operation a caller uses.

* **Client wrapper:** a class around the HTTP or RPC library with one method per remote operation.
* **Models:** request and response types that mirror the provider's data shape.
* **Serialization:** conversion to JSON, Protobuf, or another wire format and back.
* **Transport and credentials:** base URLs, sessions, TLS settings, and authentication tokens.
* **Resilience:** timeouts, retries, and mapping failures to exceptions.
* **Contract tracking:** updating the five layers above whenever the provider changes a field or method.

The Python sketch below shows the first five layers for one method of the hypothetical metering service, written against a REST-style provider. The base URL and field names are illustrative.

```python
import requests
from dataclasses import dataclass

@dataclass
class NetConsumption:
    value: float

class MeteringClient:
    def __init__(self, base_url: str, token: str, timeout: float = 5.0):
        self.base_url = base_url
        self.timeout = timeout
        self.session = requests.Session()
        self.session.headers["Authorization"] = f"Bearer {token}"

    def net_consumption(self, first: float, second: float) -> NetConsumption:
        response = self.session.post(
            f"{self.base_url}/meters/net-consumption",
            json={"first": first, "second": second},
            timeout=self.timeout,
        )
        response.raise_for_status()
        return NetConsumption(value=response.json()["value"])
```

The class contains one line of business meaning and about twenty lines of plumbing, and the plumbing repeats for every additional method. The sixth layer has no lines in the sketch because it is the work of noticing when the provider changes. If the provider renames the response field, the lookup of the value key raises a KeyError at runtime, and no compile-time warning comes first. The next section explains how the tools in this guide were chosen and how they attack these layers from different directions.

## How These Integration Boilerplate Tools Were Chosen

Each tool in this guide was checked against its own documentation or repository. The criteria were where the contract lives, which languages the tool supports, how many of the six layers it generates, and what it costs. Where a source did not state a detail, the entry leaves that detail out.

The tools start from four different sources of truth, so the next diagram groups them into four approaches. Spec-first generation reads a machine-readable description of an HTTP API. Schema and IDL generation reads a Protocol Buffers file or a GraphQL schema. Declarative or inferred clients rely on an interface that a developer writes by hand or on types that a compiler infers. Runtime-generated packages analyze the provider's own module and generate a package for each caller ecosystem.

![Four approaches to generating clients and stubs](/uploads/integration-boilerplate-tools/03-four-approaches.webp)

The question that separates the approaches is where the contract lives. A contract kept in a separate file, such as an OpenAPI description or a proto file, gives generators a stable input and gives teams one more artifact to maintain. A contract kept in code, such as a TypeScript type or a hosted module, removes the separate artifact and ties the tool to the language or runtime that holds the code. The entries below start with Graftcode and then move through the generators and typed clients.

## Best Integration Boilerplate Tools in 2026

### 1. Graftcode

![Graftcode website](/uploads/integration-boilerplate-tools/04-graftcode.webp)

Graftcode is a runtime integration system for generated, package-based method calls between modules or services. It starts from the provider's own code. A module is hosted by the Graftcode Gateway, which the Gateway repository describes as a native gateway that hosts modules behind WebSocket, HTTP, optional TCP, and optional HTTP/2 servers. The Gateway captures the module's callable surface, and the Graftcode Engine generates a package for the caller's ecosystem.

The documentation defines a [Graft](https://docs.graftcode.com/core-concepts/what-is-a-graft) as a generated package for calling a module through Graftcode. A Graft is not the hosted implementation and not a copy of the Receiver's code, and the page rules out a manually maintained HTTP SDK. The wrapper, the configuration class, and the invocation plumbing are generated, so the first three layers are generated rather than written.

Contract tracking, the sixth layer, follows the package generation model. Generation happens once per contract version rather than on every call, generated code is not edited as the source of truth, and a change to the Receiver surface is followed by regeneration. In dynamic languages such as Python, an updated Graft that no longer matches a call site surfaces as an import or attribute error, or through an optional type checker. In typed languages it surfaces at build time. The Python connect microservices tutorial shows the caller side of the metering example with a sample service, using the module and method names from that tutorial:

```python
from graft_nuget_energypriceservice.graft_config import GraftConfig
from graft_nuget_energypriceservice.meter_logic import MeterLogic

GraftConfig.host = "wss://metering.internal/ws"

consumption = MeterLogic.net_consumption_k_wh(1000, 1150)
```

The caller imports the generated classes, sets the host setting in GraftConfig to a WebSocket address ending in /ws, and calls the method. The execution modes page explains that generated Graft templates default the host to inmemory, which runs the Receiver in the same process with no network hop. In that mode the Caller must be able to resolve the Receiver module locally, through the module setting in GraftConfig or the installed package layout, and whether a generated package already includes the Receiver depends on the ecosystem. A remote address sends the same call to another process or machine and needs no module path. The diagram below shows the two modes side by side.

![Graftcode in-memory and remote execution modes](/uploads/integration-boilerplate-tools/05-graftcode-execution-modes.webp)

Graftcode is designed to coexist with REST. The [alongside REST guide](https://docs.graftcode.com/how-to-guides/use-graftcode-alongside-an-existing-rest-api) advises keeping REST for external clients that need a public HTTP contract and for callers that cannot install a generated Graft, and adding Graftcode for internal or controlled callers. It also notes that the Gateway can host existing REST workloads and a Receiver surface from one deployment, and that performance depends on the runtime pair, execution mode, payload, transport, and workload.

**Best for:** Teams whose providers are modules they control, whose callers use several languages, or whose modules may move between in-process and remote execution.

**Pros**

* Generated packages cover the wrapper, the configuration class, and the invocation plumbing, so callers import typed classes instead of writing a client for each remote operation.
* The host setting in GraftConfig switches a call between inmemory and a remote address without changing the call site.
* It is designed to sit alongside REST, so existing REST APIs stay in place for external clients while internal callers use generated packages.
* The fit page lists .NET, Node.js and TypeScript, Java and JVM, Python, PHP, and Ruby as supported for both hosting code with the Gateway and generating Grafts.

**Cons**

* Callers have to be able to install a generated Graft, so callers that cannot do that still need REST.
* The Gateway sits in the call path and has to be operated and scaled with the service, and TLS, authentication, load balancing, health checks, and telemetry remain the team's work.
* Everything exposed becomes part of the generated Graft that callers depend on, and the callable surface is an exposure boundary, not an authorization boundary.
* In-memory calls still represent and transfer arguments and results, and remote calls add latency, authentication, routing, and partial-failure concerns that generated typing does not remove.

**Pricing:** The documentation states that a free standalone Gateway exists, and the Graftcode site invites visitors to create a free account. Paid plans are not published on the site, so confirm terms with Graftcode for production use.

### 2. OpenAPI Generator

![OpenAPI Generator website](/uploads/integration-boilerplate-tools/06-openapi-generator.webp)

OpenAPI Generator is the broad option among spec-first tools. The OpenAPI Specification defines a standard, programming-language-agnostic interface description for HTTP APIs, letting people and software discover a service's capabilities without access to source code. Client generators read that description and emit the client wrapper, models, and serialization code, which covers layers one to three of the six listed above.

The generators list sorts generators into client, server, documentation, schema, and config categories, so one tool can produce clients for many languages and also server stubs from the same description. In the running example, the metering team publishes an OpenAPI description of its net consumption endpoint, and the usage service runs a generator on it instead of maintaining the MeteringClient class by hand.

**Best for:** Teams that keep an OpenAPI description for each service and have callers in several languages.

**Pros**

* One tool produces client libraries, server stubs, documentation, and configuration from the same description.
* The generators list is broad, with entries in client, server, documentation, schema, and config categories.
* Generation can run in the build pipeline, which the SSW rule on generating API clients recommends, so clients follow the description.
* It is open source under the Apache 2.0 license.

**Cons**

* The contract is only as accurate as the OpenAPI description, so the description has to be kept current on the provider side.
* The SSW rule advises keeping generated code in a separate project or folder and customizing through partial classes or wrapper services, so generated files are never edited by hand.
* The generators list tags some generators as beta, experimental, or deprecated, so the tag for the target language is worth checking before adoption.
* It supports only HTTP APIs described by OpenAPI, and credentials and resilience settings stay with the team.

**Pricing:** Free and open source under Apache 2.0.

### 3. Kiota

![Kiota documentation](/uploads/integration-boilerplate-tools/07-kiota.webp)

Kiota is Microsoft's client generator for OpenAPI-described APIs. Its overview page describes a command-line tool for generating an API client for any OpenAPI-described API, with support for C#, Go, Java, PHP, Python, Ruby, and TypeScript. It can limit generation to a specified subset of an API description when a caller needs only a few operations.

In the running example, a usage service that needs only the net consumption operation can generate just that part of a large description instead of a client for every endpoint the metering service offers.

**Best for:** Teams that consume a few operations from a large OpenAPI description.

**Pros**

* Generation can be limited to a specified subset of an API description, which keeps the generated client small.
* It supports C#, Go, Java, PHP, Python, Ruby, and TypeScript.
* The repository marks generation for C#, Go, Java, PHP, and Python as stable.
* It is released under the MIT license.

**Cons**

* The repository marks generation for TypeScript and Ruby as in preview.
* It needs an OpenAPI description, so the same upkeep burden applies as with other spec-first generators.
* Generated clients need the same care as other generated code: kept apart from hand-written code and regenerated when the description changes.

**Pricing:** Free and open source under the MIT license.

### 4. gRPC

![gRPC website](/uploads/integration-boilerplate-tools/08-grpc.webp)

gRPC moves the contract into a Protocol Buffers file. The [gRPC introduction](https://grpc.io/docs/what-is-grpc/introduction/) states that gRPC uses protoc with a special gRPC plugin to generate code from a proto file, which yields generated client and server code, and that clients and servers can be written in different supported languages, for example, a Java server with clients in Go, Python, or Ruby.

In the running example, the metering service defines the net consumption method in a proto file, and the usage service builds against the generated stub. The generated stub replaces the client wrapper, models, and serialization layers.

**Best for:** Teams that already run gRPC or want one typed contract shared across several languages.

**Pros**

* Client and server code come from the same proto file, so both sides share one contract.
* Clients and servers can be written in different supported languages.
* The generated stub removes the hand-written wrapper, model, and serialization code from each caller.
* The repository uses the Apache 2.0 license.

**Cons**

* The proto file is a separate artifact, so teams maintain it and run generation as part of their builds.
* It applies to services that speak gRPC and does not address a plain HTTP API.
* Credentials, timeouts, and retry settings still need to be configured by the team.

**Pricing:** Free and open source under Apache 2.0.

### 5. GraphQL Code Generator

![GraphQL Code Generator website](/uploads/integration-boilerplate-tools/09-graphql-code-generator.webp)

GraphQL Code Generator applies the same idea to GraphQL. It is a plugin-based tool that produces typed queries, mutations, and subscriptions for front-end stacks such as React, Vue, Angular, Next.js, and Svelte, as well as typed resolvers for Node.js or Java servers. Its documentation lists less boilerplate through full code generation, such as React hooks generation, and notes that developers no longer maintain TypeScript types by hand because the types come from the schema.

In the running example, a metering service that exposed a GraphQL API would let the usage service generate typed operations from the schema instead of writing the request and response types itself.

**Best for:** Front-end and Node.js teams whose services expose GraphQL APIs.

**Pros**

* Types come from the schema, so developers no longer maintain TypeScript types by hand.
* It generates typed queries, mutations, and subscriptions, and it can also generate React hooks.
* It produces typed resolvers for Node.js or Java servers as well as client-side code.
* The repository is licensed under the MIT license.

**Cons**

* It works with GraphQL schemas only, so it does not help with plain HTTP or RPC services.
* The GraphQL schema is a separate artifact that the team has to maintain.
* It generates types and operations, so credentials and resilience settings stay with the team.

**Pricing:** Free and open source under the MIT license.

### 6. Refit

![Refit repository](/uploads/integration-boilerplate-tools/10-refit.webp)

Refit is a type-safe REST library for .NET. A developer declares an interface whose methods carry HTTP attributes such as a GET route, and the RestService class generates an implementation that uses HttpClient to make the calls.

In the running example, a .NET version of the usage service would replace the client class with one attributed interface method per remote operation.

**Best for:** .NET teams that call REST APIs and want to keep the client declaration short.

**Pros**

* The client wrapper shrinks to an interface, so there is far less hand-written code than in a full client class.
* The interface holds the contract, so no separate description file is needed.
* The generated implementation uses HttpClient, which fits into existing .NET services.
* It is released under the MIT license.

**Cons**

* It supports .NET only.
* A developer still writes one interface method per remote operation.
* Contract tracking stays manual: when the provider changes a field or route, the interface has to be updated by hand.

**Pricing:** Free and open source under the MIT license.

### 7. Spring HTTP Service Clients

![Spring HTTP Service Clients documentation](/uploads/integration-boilerplate-tools/11-spring-http-service-clients.webp)

Spring HTTP Service Clients bring the declarative pattern to Java: an interface with @HttpExchange methods, a proxy created by HttpServiceProxyFactory, and a choice of RestClient, WebClient, or RestTemplate behind it. A controller can implement the same interface on the server side.

Spring Cloud OpenFeign followed the same model, in which a developer creates an interface and annotates it. Its reference documentation now states that the project is considered feature-complete and suggests migrating to Spring HTTP Service Clients, so new Spring projects start from the latter.

**Best for:** Java teams on Spring that call HTTP services.

**Pros**

* An annotated Java interface replaces the hand-written client wrapper.
* RestClient, WebClient, or RestTemplate can sit behind the same interface.
* A controller can implement the interface on the server side, which keeps both sides on one declaration when both are Java.
* It is part of the Spring Framework, which is released under version 2.0 of the Apache License.

**Cons**

* It supports Java with Spring only.
* A developer still writes one interface method per remote operation.
* Contract tracking stays manual, because the interface is updated by hand when the provider changes.

**Pricing:** Free as part of the open source Spring Framework, under Apache 2.0.

### 8. tRPC

![tRPC website](/uploads/integration-boilerplate-tools/12-trpc.webp)

tRPC builds typesafe APIs without schemas or code generation by using TypeScript's type inference. The concepts page describes it as designed for TypeScript monorepos, where functions defined on the server are called from the client as ordinary functions. A change to the server function then surfaces as a type error on the client at build time.

In the running example, tRPC would need the metering and usage services to be written in TypeScript in a setup that shares types, so the Python usage service would not qualify.

**Best for:** TypeScript monorepos where client and server share types.

**Pros**

* There is no schema file and no generation step, so nothing separate has to be kept in sync.
* A change to a server function shows up as a type error on the client at build time, which addresses contract tracking directly.
* Client code calls server functions as ordinary functions, with no hand-written wrapper.
* The repository uses the MIT license.

**Cons**

* Both sides need to be TypeScript in a setup that shares types, so it does not cross language boundaries.
* Credentials, timeouts, and retry settings remain the team's responsibility.

**Pricing:** Free and open source under the MIT license.

## Integration Boilerplate Tools Compared in One Table

The table lists the source of truth, output, language scope, and pricing for all eight tools, in the same order as the entries above.

| Tool                        | Source of truth                      | What it produces                                             | Languages and scope                                                          | Pricing and license                               |
| :-------------------------- | :----------------------------------- | :----------------------------------------------------------- | :--------------------------------------------------------------------------- | :------------------------------------------------ |
| Graftcode                   | Callable surface of a hosted module  | Generated package per caller ecosystem                       | Hosting and Grafts for .NET, Node.js and TypeScript, Java, Python, PHP, Ruby | Free standalone Gateway; paid plans not published |
| OpenAPI Generator           | OpenAPI description                  | Client libraries, server stubs, documentation, configuration | Many generators, listed by language or framework                             | Free, Apache 2.0                                  |
| Kiota                       | OpenAPI description                  | API clients, optionally for a subset of operations           | C#, Go, Java, PHP, Python, Ruby, TypeScript                                  | Free, MIT                                         |
| gRPC                        | Proto file                           | Client and server code through protoc plugins                | Any supported gRPC language                                                  | Free, Apache 2.0                                  |
| GraphQL Code Generator      | GraphQL schema                       | Typed queries, mutations, subscriptions, and resolvers       | Front-end frameworks, Node.js and Java servers                               | Free, MIT                                         |
| Refit                       | Interface with HTTP attributes       | Client implementation through RestService                    | .NET only                                                                    | Free, MIT                                         |
| Spring HTTP Service Clients | Interface with @HttpExchange methods | Client proxy from HttpServiceProxyFactory                    | Java with Spring                                                             | Free, Apache 2.0                                  |
| tRPC                        | Server-side TypeScript types         | Typed client calls with no generation step                   | TypeScript monorepos                                                         | Free, MIT                                         |

## How to Choose the Right Integration Boilerplate Tool

The most reliable starting point is the place where the contract already lives. A team with an OpenAPI description for each service can generate clients without changing anything on the provider side. A team that has standardized on gRPC or GraphQL already has the schema that its generators need. A TypeScript monorepo can use tRPC, and a team inside one framework can use the declarative client that matches it, such as Refit on .NET or HTTP Service Clients on Spring. Graftcode fits the case where the providers are modules the team controls, the callers use several languages, or a module may move between in-process and remote execution. The diagram below maps common situations to a first tool.

![Decision guide mapping situations to a first tool](/uploads/integration-boilerplate-tools/13-how-to-choose.webp)

Combinations are common. A service can keep a REST API with a generated OpenAPI client for external consumers and use generated Graftcode packages for internal callers that can install them. The decision criteria are the same in each case: where the contract lives, which languages the callers use, and how much of the surrounding operational work the team already covers with other tools.

## Where Should You Start With Reducing Integration Boilerplate

Integration boilerplate consists of six layers, from the client wrapper to contract tracking, and each tool in this guide removes a different subset of them. Graftcode generates the caller side from the provider's module, OpenAPI Generator and Kiota generate it from an OpenAPI description, gRPC and GraphQL Code Generator generate it from schemas, and Refit, Spring HTTP Service Clients, and tRPC reduce the wrapper to an interface or to type inference within one framework or language. Credentials, resilience, and operations remain with the team in every approach.

Begin with the service pair that has the most hand-written client code and pick the tool that matches where its contract already lives. For calls between modules that the team controls, in several languages or across the in-process and remote boundary, follow the [Graftcode quick start](https://docs.graftcode.com/quick-start), run a first call in memory, and then point the host setting at a remote Gateway to compare the two modes on one real service.

## FAQ

### Can code generation remove all integration boilerplate?

No. Generators produce the client wrapper, models, and serialization code, but credentials, resilience settings, and the process of keeping the contract current remain with the team. Generated code is also best kept separate and extended through wrappers, so some hand-written code stays around every generated client.

### Do OpenAPI Generator and Kiota do the same job?

No, not quite. Both generate API clients from an OpenAPI description. OpenAPI Generator also produces server stubs, documentation, and configuration, while Kiota focuses on API clients and can generate code for a subset of an API description.

### Do you need gRPC to avoid hand-written HTTP clients?

No. gRPC generates clients from proto files, but an OpenAPI description can drive generated clients for an HTTP API, and declarative libraries such as Refit and Spring HTTP Service Clients reduce a client to an interface. The right choice depends on the protocol the services already use.

### Can Graftcode work alongside existing REST APIs?

Yes. The Graftcode documentation advises keeping REST for external clients and for callers that cannot install a generated Graft, and adding Graftcode for internal or controlled callers. The Gateway can host existing REST workloads and a Receiver surface from one deployment.

### Does a generated package make a remote call as cheap as a local call?

No. In-memory calls still represent and transfer arguments and results, and remote calls add latency, authentication, routing, and partial-failure concerns. Generated typing reduces hand-written code, and the runtime characteristics of the chosen execution mode still need to be measured for each workload.
