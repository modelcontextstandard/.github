# Model Context Standard (MCS)

**Your LLM needs to call an API. Today that means writing a wrapper server, a custom protocol, and prompt engineering from scratch. MCS gives you a driver instead -- configure once, connect any LLM to any API.**

## The Problem You Know Too Well

You have an some API on whatever transport layer. Your LLM should use it. So you start building:

- A wrapper server that translates between your API and the LLM
- Custom authentication logic for a protocol your team has never debugged before
- Hand-tuned prompts that break when you switch models
- A deployment pipeline for yet another service that needs monitoring

Three days later, you have one integration. You need twelve.

Meanwhile, [up to 8% of MCP servers on GitHub contain potentially malicious code](https://blog.virustotal.com/2025/06/what-17845-github-repos-taught-us-about.html). Enterprise security analysts found that 88% of MCP servers require credentials, with 53% relying on static, long-lived secrets. Their conclusion: *"stdio-based MCP servers break nearly every enterprise security pattern"* [(source)](https://blog.christianposta.com/mcp-should-be-remote/).

**There has to be a better way.** And there is.


## What MCS Does Differently

Andrej Karpathy famously said that LLMs are becoming the new operating systems. If that's true, they need **drivers** -- not wrapper servers.

MCS treats LLM integration as exactly that: a driver problem. Just like your OS uses a printer driver to talk to any printer, an MCS driver lets any LLM talk to any API.

```
Today:    Your API → MCP Wrapper Server → MCP Client → LLM
With MCS: Your API → MCS Driver → LLM
```

**The crucial difference:** One REST-HTTP driver for example handles *every* REST API. Point it to a different OpenAPI spec, and it just works. No new wrapper, no new server, no new code.


## What You Get

MCS is a lightweight standard focused on what's essential for **connecting LLMs to real systems**.  It is **function calling** at its core.

### ✅ Write once, connect everything
A single driver works across all LLM applications -- ChatGPT, Claude, Llama, your custom agent. Write a REST-HTTP driver once, and every developer on every LLM platform can use it. No more reimplementing the same integration for each model or for each API.

### ✅ Your API credentials stay invisible to the agent
The driver acts as a trust boundary. Credentials live in the driver constructor -- configured by the operator, invisible to the agent. The agent calls `execute_tool()` and gets results, never secrets. No shell access to credential files, no environment variable leaks.

This is the tool execution layer that agent frameworks are missing today. An LLM that needs to read emails should see `mail.list`, `mail.read`, `mail.send` -- not your IMAP password.

### ✅ No wrapper servers, no glue code
If your API already has an spec, you're done. MCS connects directly to the spec. Zero proxy layers, zero additional servers to deploy and monitor.

### ✅ Prompts that work out of the box
Stop tuning prompts for every model. Drivers ship with refined prompts tested across models, including healing rules for common LLM output quirks. Swap in updated prompt strategies without changing a single line of code -- they're loaded from external config files, not hardcoded.

### ✅ Proven security, not reinvented security
MCS builds on HTTP, OAuth, JWT, and API keys -- standards your security team already knows and audits. No custom protocol means no new attack surface. Drivers are static modules with checksum verification, distributed over proven package managers.

### ✅ Compatible with MCP -- but without the overhead
MCP pioneered standardization, and MCS builds on the same core idea: function calling. But MCS avoids the custom protocol stack, the wrapper servers, and the STDIO security risks. Already using MCP? Wrap your existing servers with `mcs-driver-mcp` and migrate gradually.



## Project Structure

[docs](https://github.com/modelcontextstandard/docs) – The core specification (Driver Contract), in-depth guides and examples. We're optimizing this together - Join the discussion!  <br>
[python-sdk](https://github.com/modelcontextstandard/python-sdk) – Ready to use Python implementation with reference drivers. Install via pip and start building.  <br>
[typescript-sdk](https://github.com/modelcontextstandard/typescript-sdk) – TypeScript SDK (in progress – based on the Python implementation) <br>
[mcs-pkg](https://github.com/modelcontextstandard/mcs-pkg) – Centralized registry for discovering and publishing MCS drivers (planned – think apt for AI tools with autodiscovery and installation). <br>
[example-drivers] - A sample driver repo for tutorials and testing. (coming soon) <br>
[mcs-driver-rest-http](https://github.com/modelcontextstandard/mcs-driver-rest-http) – Hybrid driver for REST APIs over HTTP (OpenAPI). <br>
[mcs-driver-filesystem-localfs](https://github.com/modelcontextstandard/mcs-driver-filesystem-localfs) – Reference driver for local file system access. <br>
<br>
We recommend starting with the [docs](https://github.com/modelcontextstandard/docs) to understand the core ideas.
Once familiar, dive into the Python SDK to see how drivers are built in practice. It also serves as a blueprint for implementing SDKs in other languages.


## See It Working: The Raw Principle in 2 Minutes

Before showing the full driver architecture, here's the raw idea that MCS is built on. No SDK, no driver -- just an LLM reading an API spec and calling the endpoint.

We provide a tiny FastAPI service with a **readable OpenAPI HTML spec** and a test function (`fibonacci`) that returns `2 × Fibonacci(n)` to detect hallucinations.

> Most LLMs can currently access external content only via `GET` requests and basic HTML parsing. That's enough to prove the concept.

**Try it now** -- use the hosted demo or deploy your own:

```
https://mcs-quickstart-html.coolify.alsdienst.de
```

```bash
# or self-host:
git clone https://github.com/modelcontextstandard/python-sdk.git
cd python-sdk
docker compose -f docker/quickstart/docker-compose.yml up -d
```

**Steps:**
1. Ask your LLM to open `/openapi-html` and understand the interface
2. Ask it to get the fibonacci result for n=8
3. Correct result: **42** (the endpoint returns 2 x Fibonacci(n) -- if the LLM says 21, it hallucinated)

### Verified Results

| Model             | Result | Notes                  |
| ----------------- | ------ | ---------------------- |
| ChatGPT (Browser) | ✅      | [requires two prompts](https://chatgpt.com/c/68582012-7c70-8009-8c39-b5d05613ecd8)   |
| Claude 3 (web)    | ✅      | [two-step flow](https://claude.ai/share/57128a2d-22f8-440f-a09d-41018459d94f), restricted so it could not be done in one call          |
| Gemini            | ❌      | refuses second request |
| Grok 4            | ⚠️      | [seems to work](https://grok.com/share/bGVnYWN5_f8e10a15-65a9-47de-b43e-c72d9c004af9), but result is not readable by Grok's browser     |
| DeepSeek          | ❌      | hallucination, server call never happened         |

What you just did manually -- read the spec, call the endpoint -- is exactly what an MCS driver automates. For every LLM, every time, with optimized prompts and built-in error handling.


## How It Works: The Conversation Loop

In a real application the client, the LLM and one or more drivers interact in a loop. The driver is **stateless** -- all outcome information lives in the returned `DriverResponse`:

```
User ──► Client ──► LLM ──► process_llm_response() ──► DriverResponse
                     ▲                                      │
                     │                          response.messages
                     │                            │         │
                     │                      call_executed  no match
                     │                      or call_failed    │
                     │                            │      Final answer
                     │     messages.extend(       │
                     │       response.messages)    │
                     └────────────────────────────┘
```

1. The client sends the conversation (system prompt + message history) to the LLM.
2. The LLM responds. The client passes the response to `process_llm_response()` which returns a `DriverResponse`.
3. If `response.call_executed` is true: the client extends its history with `response.messages` (pre-formatted by the driver) and loops back to step 1.
4. If `response.call_failed` is true: the driver found a tool-call signature but could not parse or execute it. The client extends its history with `response.messages` (which includes the retry hint) and loops back.
5. If neither flag is set: `response.messages` is `null` -- the LLM output is the final answer.

The driver never talks to the LLM directly. It only provides the spec and executes calls. Because the driver holds no mutable state, it is thread-safe by design. New transports and protocols only need a new driver, not changes to the client or the LLM integration.

→ For details see the [Specification](https://github.com/modelcontextstandard/specification) and [Documentation](https://modelcontextstandard.io/).


## Why MCS exists: The Problem with Current Solutions
LLMs are becoming the core of modern software stacks, but connecting them to external systems remains unnecessarily complex. MCP deserves credit for being the first serious attempt to standardize function calling. It sparked the revolution and gave developers a protocol to build upon.

But let's be real, MCP comes with a significant overhead.

### The Protocol Overhead Problem
MCP creates a new protocol stack on top of JSON-RPC, essentially reimplementing what HTTP has solved for decades. This means:

- Reinventing proven solutions: Authentication, request handling, error management – all rebuilt from scratch
- New security vulnerabilities: Recent discoveries show the risks of custom protocol implementations  [2](https://thehackernews.com/2025/07/critical-vulnerability-in-anthropics.html), [3](https://www.oligo.security/blog/critical-rce-vulnerability-in-anthropic-mcp-inspector-cve-2025-49596), [4](https://thejournal.com/articles/2025/07/08/report-finds-agentic-ai-protocol-vulnerable-to-cyber-attacks.aspx), [5](https://noailabs.medium.com/mcp-security-issues-emerging-threats-in-2025-7460a8164030), [6](https://www.redhat.com/en/blog/model-context-protocol-mcp-understanding-security-risks-and-controls) etc.
- Additional complexity: Developers must learn new patterns instead of leveraging existing knowledge
- New Tooling: You can not simply reuse what you build with MCP
  
**The irony:** What you can accomplish with MCP, you can already do with REST over HTTP using battle-tested security, established tooling, and decades of optimization.

### The Wrapper Server Multiplication
MCP's architecture forces you to create wrapper servers for every integration:

`Your API → MCP Wrapper Server → MCP Client → App → LLM`

This creates several problems:

- Double infrastructure: Every API needs its own wrapper server running alongside the original server
- Maintenance overhead: Each wrapper must be updated, monitored, and secured independently
- Resource waste: Unnecessary processes consuming CPU and memory
- Security multiplication: More servers mean more attack surfaces

This is not just our assessment. Prominent voices in the developer community have reached the same conclusion:

- *"Stop Converting Your REST APIs to MCP"* [7](https://www.jlowin.dev/blog/stop-converting-rest-apis-to-mcp) argues that auto-wrapping REST APIs poisons AI agents with human-oriented granularity, polluting context and compounding tool-choice errors.
- *"Stop Generating MCP Servers from REST APIs!"* [8](https://www.kylestratis.com/posts/stop-generating-mcp-servers-from-rest-apis/) shows that five chained REST calls with 95% individual accuracy drop overall success to ~77%, costing roughly 7x more tokens than a single purpose-built tool.

This problem persists even with MCP frameworks that generate servers from OpenAPI specs. You're still creating an unnecessary translation layer instead of using the API directly.

### The Autostart Security Risk
MCP's STDIO-based autostart spawns processes with user privileges, creating potential security vulnerabilities:

- Untrusted code execution: Malicious MCP servers can run with full user access
- Process bloat: Background processes running even when not needed
- Privilege escalation risks: One compromised server can affect the entire system

Research suggests up to 8% of MCP servers on GitHub contain potentially malicious code [1](https://blog.virustotal.com/2025/06/what-17845-github-repos-taught-us-about.html). Enterprise security experts echo these concerns: an analysis found 88% of MCP servers require credentials, with 53% relying on static, long-lived secrets. The conclusion: *"stdio-based MCP servers break nearly every enterprise security pattern"* [9](https://blog.christianposta.com/mcp-should-be-remote/).

### The Development Burden
For every MCP client/app developer:
- Master prompt engineering for each model (again)
- Handle protocol-specific authentication with a new protocol
- Learn new toolchains for debugging, monitoring and auditing
- Implement MCP client logic for each application

For every MCP server developer:
- Write a new wrapper server for each API you want to make available to LLMs
- If you create a MCP server with valuable logic, you cannot reuse it in classic applications without also reimplementing the MCP client
- Maintain separate codebases for MCP and non-MCP integrations

This creates a fragmented ecosystem where useful logic gets locked into MCP-specific implementations.


### MCS is a pragmatic evolution
MCS addresses these pain points by recognizing that this is fundamentally a driver problem, not a protocol problem.

It trims function calling down to two building blocks:

- **Spec:** Machine-readable function descriptions -- use standards if possible! OpenAPI, JSON Schema, GraphQL SDL, WSDL, gRPC/Protobuf, OpenRPC, EDIFACT/X12, or custom formats.
- **Bridge:** Transport layers (HTTP, AS2, CAN, ...) -- handled by parsers.

Just like operating systems use device drivers to communicate with hardware, LLMs need interface drivers to communicate with external systems. 
The key here is that an **MCS driver doesn't handle a specific function or API**, it handles **all APIs using the same protocol over the same transport**.
This makes it behave like a driver for the protocol and transport, abstracting these details for the LLM and App developer.


#### Universal Protocol Support
- Write once, use everywhere: A REST-HTTP driver works across all LLM applications
- Leverage existing standards: OpenAPI, OAuth, HTTP – proven and secure
- No wrapper servers needed: Connect directly to existing APIs
- Built-in optimizations: Prompts are tested and optimized within the driver

#### Direct API Integration
Instead of:

`Your API → MCP Wrapper → MCP Client → App → LLM`

MCS enables:

`Your API → MCS Driver → App → LLM`

**The crucial difference:** A MCS driver for REST-HTTP can handle any REST API, not just one specific API. Point it to different OpenAPI specs, and it works universally without modification.

#### Security by Design

- No custom protocols: Leverage HTTP's proven security model
- Optional autostart: Only when needed, containerized and sandboxed
- Reduced attack surface: Fewer components, established security practices
- Standard authentication: OAuth, API keys, JWT – use what already works

MCS drivers are static components like software modules that can be downloaded and used directly. This enables trusted driver repositories with checksum verification, similar to APT or Maven. Making secure deployment and auto-loading straightforward.

#### Developer Experience
With MCS, developers get:

- Plug-and-play integration: Configure drivers instead of writing wrappers
- No prompt engineering: Optimized prompts are built into drivers
- Standard tooling: Use existing HTTP debugging and monitoring tools.
- Zero API changes: Existing services become LLM-accessible
- Modular architecture: Mix and match drivers and orchestrators as needed


#### Migration Path
Already using MCP? MCS provides a smooth transition:

- Wrap existing MCP servers as MCS drivers (mcs-driver-mcp-stdio)
- Gradual migration without breaking existing functionality
- Immediate benefits for new integrations


#### The Bottom Line

MCP pioneered the concept, but MCS makes it practical. While MCP requires rebuilding HTTP functionality and managing wrapper servers, MCS leverages existing standards and eliminates unnecessary complexity.

| Aspect | MCP | MCS |
|--------|-----|-----|
| **Protocol** | Custom JSON-RPC stack | Standards like HTTP/OpenAPI |
| **Server Architecture** | Wrapper server per API | Direct API connection |
| **Autostart** | Required via STDIO | Optional, containerized |
| **Authentication** | Custom implementation | Provided by used standard, like OAuth/JWT/API keys for HTTP |
| **Distribution** | Complex server deployment | Simple module download (APT/Maven-like) |
| **Prompt Engineering** | App developer responsibility | Built into drivers |
| **Reusability** | API-specific servers | Universal protocol drivers |
| **Security Model** | New attack surfaces | Security delivered by used standard |
| **Tooling** | Custom debugging/monitoring | Standard tools can be used |
| **Learning Curve** | High (new protocol) | Low (uses familiar standards) |
| **Integration Effort** | High (wrapper + client code) | Low (configure driver) |
| **Credential Isolation** | Agent sees server secrets | Driver holds secrets, agent sees only results |




## Contributing: Building the Ecosystem Together
MCS is an open exploration of how LLM integration should work. The focus right now is refining the concept and proving it scales better than existing approaches.

### Current State & Expectations
**This is a proof-of-concept, not a product.** The Python SDK and examples exist to illustrate the idea and enable early experimentation. If you're looking for production-ready solutions, this isn't it...yet.\
**What I'm doing:** Developing and refining the core concept, creating reference implementations, and exploring what's possible when we rethink LLM integration from first principles.\
**What I'm not doing:** Providing free support, maintaining SLAs, or building your custom integrations. This is open source so everyone can learn. Take what's useful, leave what isn't.

### How You Can Contribute
- **Specification refinement** is crucial for MCS's success. We need contributors to identify edge cases, develop model-specific optimizations, and help us reduce complexity to the absolute essentials. Every improvement benefits the entire ecosystem.
- **Driver development** opens up new possibilities. Whether you're interested in creating drivers for GraphQL, MQTT, industrial protocols like CAN-Bus, or even specialized hardware like printers, your work enables countless applications. The Python SDK provides a solid foundation to get started.
- **SDK expansion** makes MCS accessible to more developers. We're particularly focused on fleshing out TypeScript support, with plans to generate interfaces from the Python implementation. If you're skilled in other languages, we'd welcome additional SDK contributions.
- **Documentation and tutorials** help newcomers understand MCS's potential. We're looking for contributors to create practical guides, such as "Building a Custom Driver" tutorials that demonstrate the complete workflow from initialization to automated prompt handling.
- **Registry infrastructure** will make driver discovery and sharing seamless. Help us launch mcs-pkg, a trusted repository system with checksum verification, similar to how APT or Maven handle package distribution. Not only for the drivers, but also with autoloading model specific prompts. See [Dynamic Optimization](https://modelcontextstandard.io/Specification/LLM_Prompt_Patterns)
- **Real-world examples** and security best practices documentation ensure MCS deployments are both effective and secure. Share your integration experiences and help establish community standards.

### Getting Real
- **Want production support?** Hire a consultant.
- **Found a bug?** PRs welcome, or fork it.
- **Having a good idea?** Be my guest. I'd genuinely love to see it.
- **Just want to complain?** There's a close button on your browser tab.

The best contributions are the ones that push the concept forward. If you're here to help shape the future of LLM integration, welcome aboard.

---

*Visit our [GitHub organization](https://github.com/modelcontextstandard/.github/edit/main/profile/README.md#project-structure) to explore the code, specs and drivers. For deeper discussions about the concept and its implications: [modelcontextstandard.io](https://modelcontextstandard.io).*


## Acknowledgment 
While MCS takes a different approach, it builds on the groundwork laid by the MCP team. Their effort in structuring documentation and project layout has been instrumental and deeply appreciated while putting the idea behind MCS online. Without MCP, the need for a standard like MCS might not have become so visible.

## The Vision
> "LLMs are the new operating systems." — Andrej Karpathy\
**MCS supplies the drivers.**

This isn't about building another tool, it's about establishing the foundational layer for AI-powered computing.
