# LinkedIn Content Calendar — Weeks of April 7–25, 2026

**Cadence:** 3x/week (Mon, Wed, Fri)
**Goal:** Technical thought leadership + traffic to site
**Strategy:** Lead with code, architecture, and system design. Minimal company-name-dropping — let the technical depth speak for itself.

---

## Week 1: April 7–11

### Post 1 — Monday, April 7
**Type:** Technical deep dive — MLOps
**Format:** Code snippet + lesson

---

Your ML model is silently serving stale predictions right now.

Here's the monitoring check most teams forget 👇

[📸 Attach: post-monday-code.png — feature freshness monitor in Python]

This is a 20-line function. It would have prevented three production incidents I've personally witnessed.

The pattern:
→ Upstream ETL fails silently
→ Feature table doesn't refresh
→ Model keeps serving predictions on yesterday's data
→ Nobody notices for 48 hours
→ Business metrics look "off" and nobody can explain why

Most MLOps checklists focus on model accuracy monitoring. That's important. But freshness monitoring catches problems hours or days before drift metrics do.

Three checks every ML system needs on day one:

1️⃣ Feature freshness — catch stale data before drift
2️⃣ Prediction volume anomaly — sudden drops = silent failure
3️⃣ Input distribution shift — detect schema changes before they corrupt predictions

The model is 20% of the problem. Monitoring is easily half of the other 80%.

#MLOps #MachineLearning #ProductionML #DataEngineering #Python

---

### Post 2 — Wednesday, April 9
**Type:** Trend-driven — LLM model migration (GPT-4o retired April 3)
**Format:** Code snippet + decision matrix
**Source:** Weekly trend brief 2026-04-06 — Ranked #1

---

GPT-4o died on April 3rd.

If your production system was hardcoded to "gpt-4o", it broke last week.

Here's the routing layer I'd build for the new model landscape 👇

[📸 Attach: post-wednesday-code.png — full code + cost decision matrix]

The pattern in the image:

→ Classify every request by complexity (simple, moderate, complex)
→ Map each tier to two providers — a primary and a fallback
→ Simple tasks like classification go to Haiku/Nano at $0.05 per 1K requests
→ Complex reasoning goes to Opus/Thinking at $3.00 per 1K requests
→ ~60% of agent tasks are simple. Sending those to your most expensive model is burning money.

⚡ Three rules for surviving model deprecations:

1️⃣ Never hardcode a model string in business logic. Put it in config.
2️⃣ Always have a fallback provider. If OpenAI goes down, Claude routes. If Anthropic goes down, OpenAI routes.
3️⃣ Classify task complexity before routing. Profile your distribution first — you'll be surprised.

The teams that got hurt last week had "gpt-4o" scattered across 30 files.

The teams that didn't notice had a routing layer.

#LLM #AI #MLOps #SoftwareArchitecture #ProductionML

---

### Post 3 — Friday, April 11
**Type:** Trend-driven — MCP production tutorial (97M installs in March)
**Format:** Code walkthrough + architecture diagram
**Source:** Weekly trend brief 2026-04-06 — Ranked #3

---

MCP hit 97 million installs in March.

NVIDIA, Salesforce, ServiceNow, and HubSpot all shipped native integrations.

It's not a protocol experiment anymore. It's infrastructure.

Here's a production-ready MCP server in Python in under 40 lines 👇

[📸 Attach: post-friday-code.png — FastMCP server code + architecture diagram]

The image walks through:
→ A FastMCP server that exposes your data warehouse to any LLM agent
→ Read-only query tool with SQL injection protection built in
→ Architecture: Agent ↔ MCP Protocol ↔ FastMCP Server ↔ Database

⚡ Three design decisions that matter for production:

1️⃣ Tool schemas are your API contract — every @mcp.tool() auto-generates a JSON schema from your type hints. Be explicit with docstrings about what queries are valid.

2️⃣ Read-only by default — your first MCP server should only expose read operations. Write access to production data through an LLM agent is a risk you add deliberately, not accidentally.

3️⃣ Deploy as a sidecar, not a monolith — run your MCP server as a separate container alongside your agent. Independent scaling, versioning, and auth — just like any other microservice.

The teams building agents without MCP right now are writing custom tool integrations that will be legacy in 6 months.

The protocol won. Build on it.

#MCP #AI #AgentDesign #Python #SoftwareArchitecture

---

## Week 2: April 14–18

### Post 4 — Monday, April 14
**Type:** MLOps deployment pattern
**Format:** Code snippet + architecture lesson

---

Stop A/B testing your models in production. There's a safer way.

Run every candidate model in shadow mode first 👇

[📸 Attach: post-monday-w2-code.png — async shadow mode predict function in Python]

The pattern in the image:
→ Production model serves every request as normal
→ Candidate model runs async in parallel on the same traffic
→ Both predictions are logged for comparison
→ Users always get the current model's output — zero risk

The async part matters — you don't want the candidate model's latency to affect production response times.

Log both outputs to your experiment tracker and diff them. You'll catch things offline evaluation never will:

1️⃣ Latency spikes on edge-case inputs
2️⃣ Prediction distribution shifts (candidate too aggressive/conservative)
3️⃣ Error patterns on data the test set didn't cover
4️⃣ Memory and CPU behaviour under real concurrency

I've killed two model releases in shadow mode that looked great in offline evaluation.

One had P99 latency 3x worse than production — would've blown our SLA on day one.

The other had a subtle feature engineering bug that only triggered on weekend traffic. Weekday patterns looked identical. Weekend volume shifted the input distribution just enough to expose a dtype mismatch in one preprocessing step. We would've shipped it on a Monday and not known until Saturday.

⚠️ One caveat: shadow mode doubles your compute cost on that endpoint. Size your infrastructure for it before you flip the switch. For most teams, running it on 10–20% of traffic is enough to get signal without blowing your budget.

Ship shadow mode before you ship the model.

#MLOps #MachineLearning #ProductionML #SoftwareEngineering #Python

---

### Post 5 — Wednesday, April 16
**Type:** Trending repo review — Agent framework comparison
**Format:** Infographic + analysis
**Source:** Discovery research — DeerFlow 2.0 (bytedance/deer-flow, 58.5K stars)

---

ByteDance just open-sourced a multi-agent framework where a SuperAgent decomposes tasks into specialised sub-agents — each with their own tools, memory, and sandboxed execution.

It's called DeerFlow. One of the fastest-growing agent repos on GitHub right now.

Here's why the architecture matters 👇

🏗️ Most agent frameworks use a single-agent loop — one LLM calling tools sequentially.

DeerFlow splits the work. A SuperAgent breaks tasks into specialised sub-agents — Search, Writer, Coder — each with tuned system prompts and independent tool sets. The Coder runs in a Docker/K8s sandbox, isolating execution by default.

[📸 Attach: post-deerflow-infographic.png — architecture diagram + comparison matrix]

🔄 I tested DeerFlow alongside Claude Code, CrewAI, and AutoGen. The differences are architectural, not just feature-level. Full comparison matrix in the image.

⚡ 3 things that stand out:

1️⃣ Gateway Mode — DeerFlow exposes agents as API endpoints. You can integrate it into existing backend services, not just use it as a CLI tool. This is what makes it production-viable, not just a research toy.

2️⃣ Sub-agent specialization — The research agent, coding agent, and writing agent each have tuned prompts and tool sets. Not one generic agent doing everything. The coordination overhead pays for itself on complex, multi-step tasks.

3️⃣ Multi-LLM routing — Swap between GPT, Claude, Gemini, or open-source models per sub-agent. No vendor lock-in at the framework level. Your research agent can run on a cheap model while your coding agent uses something stronger.

🎯 When to use what:

→ Building a dev workflow with Claude? **Claude Code**
→ Multi-agent pipelines with role specialization? **DeerFlow / CrewAI**
→ Quick prototyping with conversational agents? **AutoGen**

The agent framework space is splitting into clear categories. DeerFlow is betting that the future is multi-agent coordination, not single-agent power.

Whether that bet pays off depends on whether the coordination overhead is worth the specialization gains. In my testing — for complex, multi-step research and code generation tasks — it is.

📎 Architecture diagram + comparison matrix in the image.

#AI #AgentFramework #DeerFlow #LLM #SoftwareArchitecture

---

### Post 6 — Friday, April 18
**Type:** Conference recap — Agent Advantage (Google Cloud Sydney)
**Format:** Takeaways + analysis
**Source:** Personal attendance — late March 2026

---

Only 12% of AI agent projects make it to production.

That stat came from Google Cloud's "Agent Advantage" event in Sydney — and the rest of the session explained why.

3 hours of demos, architecture deep dives, and real case studies. Here are the 4 things that stuck with me 👇

1️⃣ The 12% production gap is a people problem, not a tech problem.

We're all building POCs and demos — but the gap between "impressive demo" and "production system that delivers business value" is enormous. The teams that close it aren't the most technical. They're the ones where business and engineering speak the same language about what agents can and can't do.

2️⃣ MCP is the "USB-C of agents."

Love this analogy. Model Context Protocol is becoming the universal standard for how agents connect to data and tools. Instead of writing custom integrations for every system, you write to one protocol. The industry — including Google — has actually aligned on a single standard. That almost never happens.

3️⃣ Agent architecture is splitting into 3 clear patterns:

→ Function calling (model + tools, simplest)
→ Agent-to-data via MCP (agent connects to databases, APIs, enterprise systems)
→ Agent-to-agent orchestration (mesh of specialised agents on complex tasks)

Most teams are on pattern 1. Production teams are moving to 2. The industry is heading to 3. I dug into this in my DeerFlow breakdown earlier this week — the framework space is already splitting along these lines.

4️⃣ The "scale paradox" is real.

2025 was the adoption year. 2026 is supposed to be exponential growth. But scaling agents is fundamentally different from scaling traditional software. You need sandboxed execution, memory management, evaluation frameworks, and governance — all still being figured out.

The best demo of the day: a voice agent handling multilingual car sales, switching between English and Spanish mid-conversation, using the camera to identify a car colour, and filtering inventory from it. All on Gemini with MCP integrations. The kind of thing that makes you rethink what "production agent" means.

📎 Full writeup on my site — link in comments.

#AI #AgentFramework #GoogleCloud #MCP #SoftwareArchitecture #MLOps

---

---

## Week 3: April 21–25

### Post 7 — Monday, April 21
**Type:** Technical deep dive — Knowledge Graphs + AI Agents
**Format:** Architecture insight + lesson
**Source:** Original research — Graph RAG, MAGMA, Microsoft GraphRAG

---

Your AI agent's retrieval is broken and you don't know it yet.

Here's why vector search alone isn't enough 👇

Most teams building agents use vector RAG — embed your docs, similarity search, generate.

It works great for "what does our refund policy say?"

It completely fails for "which publishers in our network cover topics that overlap with this advertiser's brand values AND have audiences in the 25-34 demo?"

That's a multi-hop query. Vector similarity can't traverse relationships. It just finds similar chunks.

Knowledge graphs fix this.

🧠 The architecture shift:

→ Vector RAG: query → find similar text chunks → generate answer
→ Graph RAG: query → identify entities → traverse relationships → gather structured context → generate answer

Microsoft's GraphRAG proved this at scale. Their approach: extract entities from documents, build a knowledge graph, run community detection (Leiden algorithm) to find clusters, then generate hierarchical summaries. Result: ~8% better factual correctness and ~7% better context relevance vs vector-only.

⚡ Three things that changed my thinking:

1️⃣ Agent memory is going graph-native

MAGMA (Jan 2026) organises agent memory across semantic, temporal, causal, and entity graphs. The result: 45.5% higher reasoning accuracy while using 95% fewer tokens. That's not incremental. That's architectural.

2️⃣ The real distinction: similar vs connected

Vector memory retrieves facts that are semantically similar to your query. Graph memory retrieves facts that are connected through relationships. When your agent needs to reason about HOW things relate — not just WHAT is similar — graphs win.

3️⃣ Hybrid RAG is the practical answer

You don't replace vector search. You layer graph retrieval on top. Vector for broad initial retrieval, graph traversal for relationship-aware refinement. This is where production systems are landing in 2026.

🛠️ The stack that's emerging:

→ Neo4j (graph database — 65% of production Graph RAG systems)
→ LlamaIndex PropertyGraphIndex (Python-native, hybrid retrieval)
→ Microsoft GraphRAG (best for complex document reasoning)
→ LangGraph ≠ knowledge graphs (it's agent orchestration — common confusion)

The teams that will have the strongest AI agents 12 months from now aren't the ones with the best models. They're the ones with the best knowledge graphs feeding those models.

Models are commoditising. Your graph is your moat.

📎 Full writeup on my site — link in comments.

#AI #KnowledgeGraph #GraphRAG #LLM #AgentArchitecture #MachineLearning

---

### Post 8 — Wednesday, April 23
**Type:** TBD — trend-driven (check weekly brief Sunday Apr 19)
**Format:** TBD

---

### Post 9 — Friday, April 25
**Type:** TBD — original technical content
**Format:** TBD

---

### Backlog
- Dagster Asset-Based ML Pipelines (evergreen, moved from Post 6)
- Meta's Ad Knowledge Graph vs Content Commerce Matching (competitive analysis angle)

---

## Posting Tips

1. **Timing:** Post between 7:30–8:30am AEST
2. **Code formatting:** LinkedIn renders code blocks poorly on mobile — use the code in a screenshot/image carousel for better readability if engagement is low on text-only
3. **Engagement window:** Reply to every comment in the first 2 hours
4. **Hashtags:** 3–5 per post, mix of broad (#AI) and niche (#MLOps)
5. **Hook:** First 2 lines show before "see more" — make them count
6. **Technical credibility:** Don't explain what a function does — show the function and explain *why* this pattern matters

## Content Themes

### Theme 1: Original Technical Patterns (core)
Architecture decisions, code snippets, system design lessons from production experience.

### Theme 2: Research Paper Breakdowns (new)
Take a trending arxiv paper and explain what it means for practitioners. Format: "Paper says X. Here's what that actually means if you're shipping ML."

### Theme 3: Trending Repo / Tool Reviews (new)
Walk through a hot GitHub repo or new tool release. Format: code walkthrough, comparison with alternatives, or "I tried X so you don't have to."

### Theme 4: LLM Best Practices (new)
Claude, Gemma, GPT — practical tips for using them in production. Format: code snippet showing a technique, cost breakdown, or architecture pattern.

### Suggested Mix Per Week
- Mon: Theme 1 or Theme 4 (original technical content)
- Wed: Theme 2 or Theme 3 (trend-driven, timely)
- Fri: Theme 1 or Theme 4 (original technical content)

This keeps 2 out of 3 weekly posts as evergreen technical depth, with 1 trend-driven post for timeliness and discoverability.

## Automated Trend Research

A scheduled task runs every **Sunday at 8am AEST** that:
1. Searches for trending AI/ML papers, repos, LLM updates, and data engineering news
2. Writes a weekly brief to `linkedin-content/weekly-brief-{date}.md`
3. Suggests the top 3 post ideas ranked by timeliness, technical depth, and engagement potential

Check the latest brief each Sunday to pick your Wednesday trend post for the week.

## Content Pipeline (Beyond Week 2)

**Original Technical:**
- "The 3-layer observability stack every ML system needs" (monitoring deep dive)
- "RAG is not a product. It's a retrieval strategy." (LLM architecture)
- "Why I chose Databricks over Snowflake for ML workloads" (data platform trade-offs)
- "Feature stores are overengineered. Here's what to build instead." (hot take + code)
- "The async pattern that makes real-time ML serving 5x faster" (system design)
- "How to structure a model registry that doesn't become a graveyard" (MLOps)

**Paper Breakdowns (example angles):**
- "This paper just made RAG 3x faster. Here's how it works." (retrieval research)
- "Google's new paper on mixture-of-agents — why it matters for cost" (architecture)
- "The training technique that's replacing fine-tuning" (emerging methods)

**Tool/Repo Reviews (example angles):**
- "I replaced my entire logging stack with this one library" (new OSS tool)
- "Claude Code vs Cursor vs Copilot for data engineering — tested all 3" (comparison)
- "This new orchestrator just dropped. Here's how it compares to Dagster." (evaluation)
