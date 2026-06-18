## Agentkit — Framework Overview

A high-level map of what agentkit is and how it fits together. For how to use it  
 day to day see the README; for the full rationale see the chapter book and ADRs.

 

 

## What it is

agentkit is a low-level, async-first base framework for building agentic-AI  
 features. One runtime that Django (web), Airflow (batch/DAGs) and the CLI  
 (build-time) all build on. It ships opinion-free primitives and one way to compose  
 cross-cutting concerns, and leaves the opinions — loop shape, repair, approval,  
 compaction — to the layers on top.

 

A feature is **wired, not written**: you pick a control pattern, assemble two  
 middleware chains, inject the seams (LLM, store, vector), and run. The framework  
 owns the plumbing; you own the opinions.

 

Two properties shape everything:

 

* **Zero required runtime dependencies.** The core is stdlib only. Observability,  
   cost accounting, providers and databases are injected seams, never imported by the  
   core. It runs the moment it's imported, with no provider keys.  
* **Async-first.** Every I/O method is async. A sync host (Django, Airflow)  
   crosses the boundary exactly once via run\_sync. There are no a-prefixed twins.

 

 

## The one idea

Every agentic system has exactly **two units of work**: complete a chat turn, and  
 execute a tool. Every cross-cutting concern — tracing, retry, model-fallback,  
 metering, idempotency, egress, audit, caching, compaction, guards — is an  
 interceptor around one of those two units.

 

So instead of a god-loop with ten optional collaborators, there is a thin loop  
 calling an Invoker, plus a list of middlewares you assemble:

 

          
     Call (chat | tool) \-\> mw0 \-\> mw1 \-\> ... \-\> terminal (llm.stream | tool.run)     
                          tracing  meter        retry        the seam     
          

 

The chain is stream-shaped: a handler yields the operation's items (token deltas for  
 a chat, one result for a tool), and a collected chat() is just collect(stream()).  
 You write a middleware either as a raw async def mw(call, next) (full control —  
 retry, fallback and cache re-invoke or skip next), or as an ergonomic  
 BaseMiddleware class (on\_request / on\_response / on\_error) that compiles down  
 to the same primitive. You reorder a concern by editing the list, not by patching a  
 method.

 

 

## The layers

Lower layers never import higher ones. The kernel is a leaf with zero policy and zero  
 third-party imports.

 

| Layer | Holds |
| :---- | :---- |
| kernel/ (L0) | value types, the four seams, the middleware contract, resilience, concurrency, errors, stream operators — zero opinion about how to use an LLM |
| runtime/ (L1) | RunContext (identity \+ meters \+ services \+ autonomy), Invoker, Budget (per run) / Quota (per tenant) |
| middlewares/ (L2a) | tracing, meter, compaction, egress, audit, security, retry, fallback, memoize/idempotent |
| patterns/ (L2b) | AgentLoop, Agent, Workflow, teams, orchestrators, termination, gate, compose |
| capabilities/ (L2c) | tool registry, memory tool, guardrail, compactor, RAG, working context, evaluator, prompts |
| adapters/ | provider clients, stores (memory/file/redis/postgres), vector, observers — and a deterministic fake for every seam |

   
The litmus test for "does this belong in the kernel?" — *does it encode an opinion*  
 *about how to use an LLM or tool?* If yes, it's a middleware, pattern or capability you  
 compose in, never L0/L1.

 

 

Two control models, one spine

The framework is built to express the full control spectrum — you author the path, or  
 the model does — across two independent axes.

 

**Axis 1 — who authors the control flow:**

 

* **Explicit** — Workflow: a typed node graph with conditional routing and  
   human-gate nodes. Fixed, inspectable, repeatable. Best when you run it many times or  
   must audit it.  
* **Emergent** — AgentLoop (ReAct), RoundRobinTeam / SelectorTeam: the model  
   decides the next step at runtime. Flexible, adapts to intermediate results, less  
   predictable.

 

**Axis 2 — who steers at runtime** (set on ctx.autonomy):

 

* **manual** — a human approves nearly every step.  
* **gated** — a human gates only risky or expensive steps; the rest runs free.  
* **auto** — no human; the run proceeds to a budget or iteration ceiling.

 

The two axes are orthogonal: any combination is valid. Both control models and the  
 spine are async, bounded (a hard ceiling always exists), cancellable, and observable.  
 Autonomy is one shared policy (should\_gate) over one mechanism (Suspended \+  
 ctx.store), honoured by every pattern rather than re-coded per module — which is  
 also what makes human approval durable across a process restart.

 

 

## Seams and adapters

Four injected ports, each with a deterministic fake so the whole stack runs offline  
 with no keys or cost:

 

* LLMPort — a chat turn. Fake: FakeLLM (can script a multi-step tool loop and  
   stream token deltas offline).  
* ToolPort — a tool execution. A plain Python function becomes a tool: its  
   signature and type hints generate the JSON schema automatically.  
* VectorPort — scored search, behind RAG / long-term memory; tenant-scoped.  
* StorePort — one durable key-value store behind checkpoints, idempotency, audit and  
   cache. Adapters: in-memory, file, Redis, Postgres — same shape.

 

Observability is itself a seam (TracePort / ObserverPort) with a no-op default. A  
 client lights it up by injecting an adapter (obskit, OpenTelemetry, Sentry, or none)  
 and nothing else changes.

 

 

What a feature gets for free

Because the cross-cutting concerns live in the two chains, the pattern itself stays  
 thin — there is no try/except or with span(...) inside it. Wiring the standard  
 chains gives a feature, with no code in the loop:

 

* chat and tool spans (once a tracer is injected),  
* classified retry plus a circuit breaker, and model fallover,  
* per-run and per-tenant metering with cache-aware cost,  
* a prompt-injection guard and default-deny, SSRF-safe egress,  
* idempotent side-effecting tools and an audit record per tool call,  
* human-approval suspend and durable resume (survives a process restart).

 

 

## Building a feature (the shape)

          
     \# 1\) assemble the two chains once — order is explicit and yours     
     chat\_chain \= \[tracing(), meter(), SecurityMiddleware(),     
                   fallback(\[...\]), retry(breaker=CircuitBreaker("llm"))\]     
     tool\_chain \= \[tracing(), meter(), egress(Guardrail(...)),     
                   idempotent(), audit(), retry(breaker=CircuitBreaker("tools"))\]     
     invoker \= Invoker(llm=llm, chat\_middleware=chat\_chain, tool\_middleware=tool\_chain)     
           
     \# 2\) a plain function IS a tool     
     def fetch(url: str) \-\> str:     
         "Fetch a URL and return its text."     
         return \_http\_get(url)     
           
     \# 3\) identity \+ meters \+ services \+ autonomy, then run a pattern     
     ctx \= RunContext(correlation\_id=cid, scope=Scope(org\_id, domain\_id),     
                      budget=Budget(max\_cost\_usd=5.0, max\_calls=200, max\_concurrency=8),     
                      services=Services(invoker=invoker, store=FileStore("/var/agentkit")),     
                      autonomy="gated")     
           
     loop \= AgentLoop(name="page\_analyzer", model="anthropic/claude-sonnet-4-6",     
                      tools=\[fetch\], system="...", response\_format={"type": "json\_object"})     
           
     result \= await loop.run(task, ctx)      \# sync host: run\_sync(loop.run(task, ctx))     
          

 

Change a behaviour — say, add retry to tools — by editing a middleware list, never by  
 touching the loop.

 

 

Patterns at a glance

* **\`AgentLoop\`** — the thin ReAct loop: reason, call tools, feed results back (framed  
   as untrusted), then stop, repair once, or hit max\_iterations. Durable resume and  
   human approval built in; run() collects, stream() yields events.  
* **\`Agent\`** — one structured chat turn, with optional typed parsing and bounded repair.  
* **\`Workflow\`** — an explicit graph of agent / tool / fn / human-gate / subworkflow /  
   team nodes, with data edges, conditional routing, wave-concurrency and a step ceiling.  
* **\`RoundRobinTeam\` / \`SelectorTeam\`** — an emergent multi-agent loop over a shared  
   working-context blackboard; the next speaker can be chosen by a model selector or a  
   handoff, and the loop stops on a termination condition.  
* **Orchestrators** — fan-out planning and task/progress ledgers that re-plan on stall.

 

 

Memory

* **Short-term** — the live transcript / working context (messages \+ scratchpad, with  
   fork/merge).  
* **Long-term RAG** — auto-injected: pass memory= to an Agent or AgentLoop and  
   retrieved, tenant-scoped context is folded into the prompt.  
* **Agent-managed** — MemoryTool: the model drives a note file-tree itself.

 

 

Why it's shaped this way

* One interception model collapses N concerns across M call sites into a single place.  
* A pattern holds only its control opinion, so it stays small and composable.  
* Seams give offline determinism, host portability and a dependency-free core.  
* Results carry partial and evals rather than throwing for expected outcomes, so a  
   parent agent can reason about a child's partial result.  
* Safe by construction: idempotent side effects, default-deny egress, untrusted-framed  
   tool output, tenant isolation, redacted secrets.

