# Model Context Standard (MCS)

<p align="center">
  <img src="assets/mcs-logo-lg.jpg" alt="MCS logo">
</p>

**A lightweight driver pattern that links any LLM to the digital world – only the essentials required.**

---

## Key Points at a Glance

* **Open standards first** – reuse OpenAPI, REST, OAuth, Docker …
* **No custom protocol** – simpler networking, easier auditing, let the driver to its thing.
* **Driver pattern** – once a driver exists (REST‑HTTP, EDI‑AS2, MCP‑stdio, Filesystem, Printers …) every LLM app can use it
* **Optional autostart** – run tools in containers when useful, *explicitly* and sandboxed
* **Complimentary to MCP** – you can even write an "MCP‑over‑SSE" or "MCP-over-STDIO" driver


## Project Structure

- [specification](https://github.com/modelcontextstandard/specification) – Protocol specification and documentation  
- [docs](https://github.com/modelcontextstandard/docs) – The documentation of MCS (comming soon)
- [python-sdk](https://github.com/modelcontextstandard/python-sdk) – Python implementation  
- [typescript-sdk](https://github.com/modelcontextstandard/typescript-sdk) – TypeScript implementation (planned)  
- [mcs-pkg](https://github.com/modelcontextstandard/mcs-pkg) – Official registry for publishing and discovering MCS-compatible drivers (planned)

---

## Why MCS exists

The **Model Context Protocol (MCP)** deserves credit for being the **first open standard** that gathered the loose ends of function‑calling.
The goal is to integrate LLM applications with external systems in a seamless way.

But you do not need a new protocol with new challenges for that. A driver-based approach is all you need. It comes with a lot of benefits that MCP does not provide, like a real plug & play experience and more security.

In practice, many teams just need a **simple, secure way** to expose existing APIs to an LLM without introducing a brand‑new protocol stack with its own new challenges.

At the end of the day, function calling is what connects LLMs with external systems.

<br>

So the question is: what is really needed to standardize this way?

It turns out the challenge is best handled by **drivers**.

LLMs are becoming the center of a new software stack. Like an OS, they need to talk to peripherals. That is exactly what drivers are for.

Drivers can be written once and reused across use cases.

Think about a REST-over-HTTP driver: once written, *any* REST API can be used. No more wrappers needed to connect LLMs to it.

**MCS trims the idea down to two building blocks**

1. **Bridge** – transport layer (HTTP, AS2, CAN …)
2. **Spec** – any machine‑readable function description (OpenAPI, JSON‑Schema …)

Resulting in a swappable **driver** – comparable to device drivers in an OS.
*Result: less boilerplate, fewer attack surfaces, fully reusable APIs.*

MCS is a standard way of doing function calling – without enforcing its implementation.


## Getting Started: Try the Proof of Concept – MCS in 2 minutes

You can verify the MCS pattern with *any* LLM that has web access by spinning up the tiny FastAPI demo included in this repo.

```bash
# clone on a VPS / cloud VM with a public DNS or IP
$ git clone <repo-url>
$ cd modelcontextstandard
$ docker compose -f docker/quickstart/docker-compose.yml up -d  # exposes :8000 on your public host
# optional: use a tunnel such as ngrok or cloudflared if you do not have a static IP
```

> 🛠️ Tip: Platforms like **Coolify** or **Render** make one‑click deployment of Dockerised apps very easy.

No server handy? A public demo is **temporarily** available at:

```
https://mcs-quickstart.coolify.alsdienst.de
```

(as long as the endpoint is up).

The demo service is implemented in `fastapi_server_mcs_quickstart.py` and exposes two endpoints:

| Path                  | Purpose                                            |
| --------------------- | -------------------------------------------------- |
| `/openapi-html`       | serves the OpenAPI spec as HTML (LLM‑readable)     |
| `/tools/fibonacci?n=` | returns *2 × Fibonacci(n)* to detect hallucination |

#### How to test with an LLM

1. Ensure the demo is reachable under a **public domain** (or use the hosted URL above).
2. Ask the LLM to fetch `/openapi-html` and construct the URL for the Fibonacci tool.
3. In a second prompt, ask the LLM to visit that URL (e.g. `...?n=8`).
4. A correct call returns **34**. If the model answers **21**, it hallucinated.

| Model             | Result | Notes                  |
| ----------------- | ------ | ---------------------- |
| ChatGPT (Browser) | ✅      | requires two prompts   |
| Claude 3 (web)    | ✅      | two‑step flow          |
| Gemini            | ❌      | refuses second request |
| Grok              | ❌      | mis‑parses OpenAPI     |
| DeepSeek          | ❌      | hallucination          |

---

## Contributing

We welcome contributions that

* refine the formal description of the MCS pattern
* provide **new drivers** for other transport layers or data formats
* Build SDKs for other programming languages
* improve docs & examples
* Building a centalized MCS-compatible registry

See **CONTRIBUTING.md** for details.

> **Proof‑of‑Work Notice** – This orga and its repositoris are shared *as is*. PRs and issues are welcome, but there is **no guarantee** of review, merging or long‑term maintenance. Contributions will be evaluated based on alignment with research goals and available time.


## Contact

Open a GitHub Discussion or mention **@your‑username** in an Issue/PR.

> "LLMs are the new operating systems." — Andrej Karpathy <br> **MCS supplies the drivers.**


<br>

<sub>
<b>Note:</b> While MCS takes a different approach, it builds on the groundwork laid by the MCP team. Their effort in structuring documentation and project layout has been instrumental and deeply appreciated while put the idea behind MCS online. Without MCP, the need for a standard like MCS might not have become so visible.
</sub>