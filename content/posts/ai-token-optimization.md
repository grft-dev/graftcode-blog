---
title: 'AI Token Optimization: How to Reduce Context Waste in AI Coding'
slug: ai-token-optimization
author: Adam Wasielewski
category: General
readingTime: 7
coverImage: /uploads/ai-token-optimization/image2.png
---

## TL;DR

* Tokens are the billing and processing unit for LLMs. Every request has an input cost and an output cost, and both are charged per token, including the code, docs, and API specs an AI coding tool reads before it writes a single line back.
* Context that fits inside the window isn't free. Some providers tier pricing higher once a request exceeds a length threshold, and every extra token in a long request adds to the bill regardless of whether it ultimately matters.
* Longer context isn't just more expensive; it can be less accurate. Research on "context rot" found model accuracy degrading well before the context window actually filled up, and separate research on "lost in the middle" attention found models read the start and end of a context far more reliably than the middle.
* Integration boilerplate, HTTP clients, DTOs, serialization, and error handling make up a disproportionate share of what an AI coding tool has to read per service call; none of it is business logic.
* Graftcode reduces that specific surface with strongly typed Grafts, shrinking what has to be read (and paid for) per service interaction.

Most conversations about making AI coding tools "more efficient" focus on the prompt: shorter instructions, better examples, tighter formatting. That's real, but it's a small lever compared to the one nobody adjusts: how much of a codebase's actual content is integration plumbing that has nothing to do with the task at hand and gets re-read, re-parsed, and re-paid for every time a tool touches it.

This guide starts with what a token actually is and what it costs, then works through why bigger context windows don't solve the problem the way they're marketed to, and ends with the specific architectural pattern, reducing the integration surface between services, that cuts token usage at the source rather than trying to compress or summarize it away after the fact.

## What a Token Actually Is and Why It's the Unit Thats Important

A token is the unit an LLM actually processes and is billed for, not a word or a character, but a chunk somewhere in between. For English text and most programming languages, a common rule of thumb is roughly four characters per token. However, this varies by tokenizer and by how repetitive or structured the text is.

Every request to an LLM has two separate costs: an **input cost** for every token sent to the model (the prompt, the retrieved files, the conversation history) and an **output cost** for every token the model generates in response. Both are billed per token, and for most providers, the per-token rate for output is higher than for input.

This matters for AI coding tools specifically because most of what they do is read before they write. Whether it's a coding agent exploring a codebase to understand a function, or a chat interface pulling in file context to answer a question, the tokens spent reading integration code, API specs, and boilerplate are billed the same as tokens spent reading the business logic that actually matters.

The assumption worth challenging is that just because something fits within a model's context window, including it is free. It has a direct cost on the input side, and, as the next two sections cover, it can also have an accuracy cost that doesn't show up on the bill at all.

## Why Bigger Context Windows Don't Mean Free Context

Context windows have grown dramatically; several frontier models now advertise windows of 1 million tokens or more. That growth gets marketed as solving the context problem outright: just put everything in.

Two things complicate that framing. First, cost. Even without tiered pricing, Claude Sonnet 4.6 and later now bill the full 1M context window at standard rates, volume still accumulates fast. At $3 per million input tokens, a single request that fills half a 1M context window costs $1.50 in input alone, before any output is generated. For an AI coding tool that repeatedly pulls integration boilerplate into context across a session, that cost compounds with every touch of every service boundary in the codebase.

Second, and more consequential, a bigger window doesn't mean the model reasons equally well across the entire window. That's the subject of the next section, and it's a bigger problem than the pricing tiers.

## Context Rot: Why More Tokens Can Mean Worse Answers, Not Just Higher Bills

In July 2025, Chroma Research published a paper, "[Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://www.trychroma.com/research/context-rot)", that ran controlled experiments across 18 frontier models, including the GPT-4.1, Claude 4, Gemini 2.5, and Qwen3 families. The findings: performance degrades as input length grows even on trivially simple tasks, including replicating a sequence of repeated words, a task with no ambiguity. A single distractor measurably reduced accuracy; adding more compounded the degradation. And the counterintuitive result: models consistently scored better on shuffled haystacks, randomly reordered sentences with no logical flow, than on coherent, structured ones. Structural coherence in the surrounding context appears to make relevant content harder to isolate, not easier.

A separate, earlier body of research documents the same pattern from a different angle. Liu et al.'s "Lost in the Middle: How Language Models Use Long Contexts" (TACL, 2024), from researchers at Stanford and Meta AI, found that performance follows a U-shaped curve across the context window: models attend to the beginning and end far more reliably than to the middle. In multi-document question-answering tasks, accuracy dropped significantly when the relevant information appeared in the middle of the context rather than at the very start or end.

Liu et al. describe performance following a U-shaped curve: highest when relevant information appears at the very beginning or end of the context, dropping significantly in the middle. The exact magnitude varies by model and task, but the directional pattern is consistent across all models tested.

The practical implication for AI coding tools: every token of integration boilerplate that gets pulled into context isn't just adding to the bill; it's diluting the model's attention budget and pushing whatever actually matters further from the positions where the model reads most reliably. A codebase full of HTTP clients, DTO definitions, and serialization code doesn't just cost more to read. It makes the model measurably worse at finding the business logic buried inside it.

## Where Token Waste Actually Accumulates in a Codebase

Integration code between services is the most consistent offender. When Service A calls Service B over REST, the codebase typically includes an HTTP client, a DTO class matching the expected request and response shape, serialization and deserialization logic, and error handling for network failures, none of which describes what the services actually do for the business.

```python
# What an AI coding tool has to read to understand one cross-service call
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

To fully understand what this call does, a coding tool typically also needs the request DTO, the response DTO, any OpenAPI spec or endpoint documentation, and whatever error-handling wrapper sits around the raw call. That's several files pulled into context for a single service interaction, each billed as input tokens, and none of them contain the actual business rule the tool is trying to reason about.

Compare that to a minimal typed interface for the same interaction:

```python
# What the same interaction looks like as a single typed call
result = OrderProcessor.process_order(order_id, customer_id)
```

One line communicates the same callable surface, no endpoint path, no HTTP verb, no request schema to infer, no response shape to guess at. For a coding tool, that's the difference between reading one line to understand a service interaction and reading several files to reconstruct that understanding by inference.

This isn't just true of the example above. Graftcode reports a 60% reduction in tokens and 60% less code across typical service integrations, consistent with the idea that in a typical service call, roughly 60% of the code is integration boilerplate and the remaining 40% is the business logic that actually matters.

This compounds specifically for AI coding agents that operate across a session rather than a single query. If an agent re-reads the same integration file five times across a multi-step task, once to understand the call, again after making a change elsewhere, again to verify behavior, the token cost of that boilerplate is paid five times over, not once.

## A Framework for Measuring Token Waste

Before reaching for any specific fix, it helps to actually measure where the waste is, rather than assuming. Two practical measures:

**The ratio of business logic tokens to integration tokens, per file or per service call.** A route handler where 80% of the code is HTTP client setup, DTO mapping, and error handling, and 20% is the actual business rule, has an inverted ratio that's worth flagging regardless of what fixes it.

**Tokens per service interaction, not just tokens per session.** Session-level token totals hide where the cost is concentrated. Measuring per-interaction cost, how many tokens does understanding this one call to that one service actually take, makes it possible to see which specific integrations are disproportionately expensive, rather than treating the whole codebase as equally costly.

Neither of these requires any specific tool to apply. They're a way of finding the actual waste before deciding how to remove it.

## How Graftcode Reduces the Token Footprint of Integration Code

Graftcode is a cross-runtime communication layer. Services call each other's public methods directly through automatically generated interfaces called Grafts, installed as typed packages via standard package managers, without hand-written integration code, DTOs, client libraries, or serialization boilerplate.

**Graftcode Gateway** runs the provider service itself; it spins up a runtime for the configured modules. By default, it exposes public methods automatically once a module is hosted; developers can narrow that surface with access policies to control which methods are actually reachable, but nothing needs to be explicitly configured to expose a module's public interface. It is not positioned between services and does not route calls.

```bash
# Run the Gateway, which hosts the order processor service
./gg --projectKey <your-project-key> --modules ./order_processor.py
```

The Gateway serves calls on port 80 and Graftcode Vision (live documentation of every exposed method) on port 81, by default. Every hosted service also becomes an MCP (Model Context Protocol) endpoint automatically, at `http://<service-host>:81/mcp`, no separate server, tool registration, or schema definitions required, so an AI coding tool using MCP can call the service directly without an installed Graft.

**Graft** is a strongly typed interface, installed via a package manager, that mirrors the provider's public methods exactly: names, argument types, return types. Graftcode supports 20 programming languages across 10 package managers.

```bash
npm install --registry https://grft.dev/<project-id>__free @graft/nuget-orderprocessor@1.0.0
```

**GraftConfig** is an internal class of each Graft that controls whether a call runs in-process or remotely, set via an environment variable, config file, or directly on the Graft, not a code change:

```python
const { GraftConfig, OrderProcessor } = require("@graft/nuget-orderprocessor");
GraftConfig.host = "wss://order-processor:9000/ws";
```

### **A real, counted example, not an estimate**

A route handler that inlined Reddit's OAuth token refresh, header construction, and a raw HTTP call, before extraction:

```python
# Before -- Reddit auth and fetch logic inline in main.py
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

After extracting that logic into its own Graftcode service and calling it through a typed method:

```bash
# Install the Graft
python -m pip install --extra-index-url https://grft.dev/simple/<project-id>__free graft-nuget-redditservice==1.0.0
```

````python
# After -- one typed method call via Graftcode in main.py
import os
from graft_nuget_redditservice.graft.nuget.RedditService import GraftConfig
from graft_nuget_redditservice.redditservice import RedditService

GraftConfig.host = "wss://reddit-service-host/ws"

@app.post("/generate_comment")
async def generate_comment(request):
    post = RedditService.get_submission(url=reddit_url)
```"
````

The code above shows a real, concrete illustration of this shift, but it's worth being precise about what's actually being compared. The install command and the two imports in the 'after' version are a one-time cost, written once when the service is first wired up, not something an agent re-reads every time it revisits this route. What actually gets re-read on every visit is the call itself, a single line, a small fraction of the inline version it replaced. Graftcode states this reduction generally as 60% fewer tokens and 60% less code across typical service integrations, which is the figure worth anchoring to rather than a token count from any one example.

Graftcode Vision, running on port 81, generates the method signature shown in the "after" example, live documentation derived directly from the running service, not a hand-maintained OpenAPI spec that can drift out of sync with the code it's meant to describe.

![](/uploads/ai-token-optimization/image1.png)

This doesn't change how well a model reasons. It changes how much of what the model reads is integration plumbing rather than business logic, which, per the context rot and lost-in-the-middle research above, is not just a cost question but an accuracy question.

| Without Graftcode                                                     | With Graftcode                            |
| --------------------------------------------------------------------- | ----------------------------------------- |
| HTTP client, DTOs, and serialization code per service pair            | Graft installed via package manager       |
| Request/response shape inferred from an OpenAPI spec or endpoint docs | Method signature read directly            |
| Re-read in full every time the integration is touched in a session    | Same minimal signature, every time        |
| Integration surface grows with every new service pair                 | Overhead stays flat as services are added |

## What a Token-Efficient Codebase Looks Like End to End

Pulling the pieces together, a token-efficient codebase has:

* **Minimal typed interfaces between services**: a method signature communicates a callable surface in a fraction of the tokens an OpenAPI spec or hand-written client requires.
* **No redundant integration boilerplate per service pair**: the same HTTP client and DTO pattern shouldn't be rewritten (and re-read) for every service that talks to the same downstream dependency.
* **Business logic separated from transport concerns**: a route handler or business function should read as what it does, not as how it happens to communicate over the network.
* **Documentation that's machine-readable and current**: live, generated documentation avoids the token cost of a coding tool cross-referencing stale docs against the actual code to figure out which one is accurate.
* **Reviewable AI output**: when an AI tool only touches business logic instead of generating integration boilerplate alongside it, the resulting pull request is smaller and easier for a human to actually review; the token savings compound into review-time savings too.

None of these properties are unique to reducing token cost; they're the same properties that make a codebase maintainable for human engineers. What changes with AI coding tools is that the cost of not having them is now measured twice: once in engineering time, and once on the token bill.

## Conclusion: Token Efficiency Is an Architecture Decision, Not a Prompting Trick

The instinct when token costs climb is to reach for a prompting fix: shorter instructions, tighter formatting, more aggressive summarization. Those help at the margins. They don't address the actual source of the cost: a codebase where a large share of what an AI tool has to read, every single time it touches a service interaction, is integration plumbing that has nothing to do with the task.

Context rot and lost-in-the-middle research both point to the same conclusion on accuracy: unnecessary tokens aren't neutral. They cost money on the way in, and they measurably degrade the model's ability to find what actually matters once they're there. Reducing the integration surface between services, not compressing it, not summarizing it, actually removing the need to write and re-read it, is what fixes both problems at the same time.

To see how Graftcode reduces the integration surface an AI coding tool has to read per service call, explore [Graftcode](https://www.graftcode.com/) or go straight to the [Graftcode Academy](https://academy.graftcode.com/) to get started.

## FAQs

### **1. Does a larger LLM context window mean I can stop worrying about token costs?**

No. Larger windows change what's technically possible to fit into a single request, not what it costs. Even without tiered pricing, some providers previously charged more beyond certain thresholds, though Claude Sonnet 4.6 and later bill the full 1M window at standard rates, every token is still billed on the input side. A request that fills a large context window can cost several dollars in input tokens alone before any output is generated.

### **2. What is "context rot" and why does it matter for AI coding tools?**

Context rot refers to research findings, most notably from Chroma Research's 2025 study across 18 frontier models, showing that model accuracy degrades as input length grows, often well before the model's documented context limit is reached. For AI coding tools, this means padding context with unnecessary files or boilerplate doesn't just cost more; it can make the model less reliable at finding the code that actually matters.

### **3. What is the "lost in the middle" problem?**

It's a documented pattern in which LLMs attend to information at the beginning and end of a context window more reliably than to information placed in the middle. In multi-document tasks, accuracy has been shown to drop by more than 30% when relevant content appears in the middle rather than at the start or end. For a codebase read into context, this means the position of relevant code, not just its presence, affects how reliably a model uses it.

### **4. Why does integration code between services waste more tokens than it seems to at a glance?**

A single REST service call typically requires an HTTP client, request and response DTOs, serialization logic, and error handling, all spread across multiple files. An AI coding tool has to retrieve and read all of it to understand a single interaction, and if that interaction is revisited multiple times during a session, the same boilerplate gets re-read and re-billed each time.

### **5. How does Graftcode reduce token usage without changing how the AI model itself works?**

Graftcode replaces hand-written HTTP clients, DTOs, and serialization logic between services with strongly typed Grafts installed via a package manager. A coding tool reads a method signature instead of retrieving and parsing an OpenAPI spec, a client implementation, and DTO definitions across multiple files, thereby reducing both the token cost of reading that integration and the amount of unrelated text competing for the model's attention as it reasons about the surrounding code.
