# noos - A Framework Of Agentic Minds

**Run specialized AI agents the same way you run services: declared once, invoked over HTTP, sandboxed per call, audited end-to-end.**

Noos is a framework for building and operating **personas** — specialized AI
agents defined as data rather than code. Teams describe what an agent should
know, what it is allowed to do, and where its boundaries are; the framework
takes care of running it safely, observing it, and giving the rest of the
organisation a uniform way to call it.

---

## The problem Noos solves

Across most organisations the same pattern keeps repeating: a team has a
recurring, well-bounded task that an LLM could handle competently if it were
given the right context — a code audit, a ticket triage, a repo scaffold, a
document extraction, a framework-specific code generation. Today these tasks
are done in one of three unhappy ways:

- **Manually**, over and over, by people who could be doing higher-leverage
  work.
- **As ad-hoc scripts** that don't generalise, have no shared audit trail, no
  cost ceiling, no isolation, and rot the moment their author moves teams.
- **As a single "mega-agent"** with every tool and every prompt bolted on,
  where roles bleed into each other, credentials are over-scoped, and the
  system prompt grows until nobody understands it.

Each path produces the same long-term outcome: combinatorial maintenance, no
shared learnings, and audit / budget / sandboxing reinvented (badly) every
time.

Noos is built on the observation that this is a **platform problem**, not a
prompt problem. What teams need is one place to declare an agent, and one
place to run it — with the same isolation, observability, and budget guarantees
you would demand of any other piece of production infrastructure.

---

## What Noos is

Noos is built on three ideas that sit underneath everything else in the
framework.

### 1. A persona is data

A **persona** is a declarative bundle: identity, knowledge, skills,
capabilities, and limits. It is authored by the team that owns the task,
stored in a versioned registry, and resolved into an immutable form at the
moment it is executed.

A persona declares:

- **Who it is** — the system prompt and description that frame the agent's
  role.
- **What it knows** — the documents, standards, and reference material that
  should always be in scope.
- **How it does things** — reusable skills the agent can apply.
- **What it can do** — the tools and external services it is permitted to
  reach.
- **Its boundaries** — input and output contracts, timeouts, turn limits,
  token limits, cost ceilings, allowed network destinations, and the
  credentials it needs to operate.

Adding a new capability to the organisation is a pull request to the registry,
not a deployment. Teams own their personas; the platform owns the runtime.

### 2. The harness is generic

There is one execution harness per language family and one runtime
implementation per AI vendor. The harness takes a resolved persona plus a
caller's input, runs the agent loop inside an **ephemeral, isolated
sandbox**, and returns a structured result and any artifacts the persona
produced. The harness has no per-persona logic and no idea what the persona
"does" — it only knows how to honour the persona's declared contract and
limits.

This separation is what keeps the framework AI-vendor neutral. Adding a new
runtime — a different model provider, a different sandbox technology — does
not change the contracts that callers and persona authors depend on.

### 3. The platform is dumb on purpose

Noos does not branch, retry across personas, sequence multi-step processes,
or implement human-in-the-loop. Workflow logic lives **above** the framework,
in whatever orchestrator the consumer prefers — a workflow engine, a CLI, a
backend service. The platform's contract is the simplest one that is still
useful: *invoke a persona, get a result*. Everything else is somebody else's
job, deliberately.

---

## Core concepts

| Concept | What it is |
|---|---|
| **Persona** | A declarative description of one specialised agent. Versioned, composable, owned by a team. |
| **Fragment** | A reusable piece of a persona — a base prompt, a security posture, a common tool grant — that other personas extend. Composition can only *restrict*, never expand. |
| **Registry** | The versioned store where personas and fragments live. Resolves source manifests into immutable, fully-composed form on demand. |
| **Gateway** | The HTTP entry point. Validates calls, enforces idempotency, hands work to the runtime, and exposes job state to callers. |
| **Runtime** | The component that actually runs a persona — materialising it into a sandbox, enforcing its limits, streaming audit events, and producing the final result. |
| **Sandbox** | The per-invocation container in which a persona runs. Ephemeral, filesystem-scoped, network-filtered, resource-limited. The unit of isolation. |
| **Audit log** | An immutable, append-only record of every job, every model call, every tool call, every budget warning, and every policy decision. Noos refuses to accept new work if its audit sink is unhealthy. |
| **Notification** | A real-time signal — webhook, message bus, live channel — emitted as a job moves through its lifecycle. Distinct from audit: audit is the durable record, notifications are how callers find out. |

### The three context lifetimes

Every piece of information an agent uses falls into exactly one of three
buckets. Keeping them separate is the single most important discipline the
framework enforces.

- **Persona-static.** From the manifest. The system prompt, organisational
  standards, framework docs, skill files. Mounted read-only into the sandbox.
- **Request-dynamic.** From the invocation. The specific task, the target
  repository, the ticket text. Delivered as the job input.
- **Agent-fetched.** Pulled live during execution. The current state of a
  git repo, today's data, a search result. Reached through declared,
  filtered capabilities.

The failure mode this prevents is the one every ad-hoc agent eventually
falls into: stuffing everything into the prompt until it is fifty thousand
tokens of stale text, or making the agent rediscover context the caller
already had.

---

## How a request flows, in plain terms

1. An orchestrator, service, or human calls the **gateway**, asking for a
   specific persona at a specific version with a payload of inputs.
2. The gateway validates the call, checks the idempotency key, and accepts
   the job. It responds immediately — the work is asynchronous.
3. The **registry** resolves the persona: composing it with its fragments,
   producing the immutable bundle that will actually run.
4. The **runtime** spins up an ephemeral sandbox, materialises the persona
   into it, injects only the credentials the persona declared it needs, and
   starts the agent loop. Every model call and every tool call is recorded
   in the audit log as it happens.
5. The persona produces a structured result (and optionally, artifacts).
   The runtime tears the sandbox down. The job's terminal state is
   recorded and the caller is notified.

Throughout the lifecycle the caller can query the job's status, cancel it,
or subscribe to its events. Once a job reaches a terminal state, its result
is immutable.

---

## What Noos guarantees by design

- **Isolation per invocation.** No persona shares filesystem, network egress,
  or credentials with another. Sandboxes are torn down at the end of every
  job.
- **Restrictive by default.** Personas receive exactly the tools, network
  destinations, and credentials they declared. There is no permissive mode
  and no fallback that loosens the contract.
- **Composition that only restricts.** A persona that extends a fragment
  cannot grant itself broader access than the fragment allowed. Permission
  fields intersect; limit fields take the minimum. Safety is a property of
  the algebra, not of the author's care.
- **Immutable audit.** Every AI call, tool call, and lifecycle event is
  written to an append-only log. The platform refuses to accept work if the
  audit sink is unhealthy.
- **Idempotent submission.** Every mutating call carries an idempotency key;
  retries return the original job rather than creating a duplicate.
- **Budget enforcement.** Timeouts, turn caps, token caps, and cost ceilings
  are honoured by the runtime, not by hoping the model behaves.
- **Credentials never travel through data planes.** Secrets reach the
  sandbox as environment variables at spawn time. They are absent from the
  manifest, absent from the input, and absent from every audit payload.

---

## What Noos is not

Being explicit about non-goals is part of the design.

- **Not a chat product.** Noos runs bounded, asynchronous jobs. It is not a
  conversational interface.
- **Not a public marketplace.** Personas are organisational assets, not
  community downloads.
- **Not a workflow engine.** Branching, retries across personas, and
  multi-step orchestration live in the systems that call Noos.
- **Not an evaluation framework.** Measuring persona quality is a separate
  problem and is intentionally left to the teams that own each persona.
- **Not vendor-locked.** The contracts that callers and persona authors
  depend on contain no AI-vendor-specific surface. The current runtime is
  one implementation, not the only one.

---

## Who Noos is for

- **Platform teams** who want to give the rest of the organisation one safe,
  observable way to run AI work — and one place to enforce policy across all
  of it.
- **Domain teams** who have a recurring task they would like to encode once
  and let others invoke uniformly, without writing or operating a bespoke
  service.
- **Orchestrators and workflow authors** who want personas to be just
  another step they can call over HTTP, with predictable inputs, outputs,
  and lifecycle semantics.

---

## Project status

Noos is in active development. The contracts that callers and persona
authors depend on are stable; runtimes, persistence backends, and
notification channels are being added incrementally. The canonical
specification — including the binding interface and lifecycle contracts —
lives in [`persona-platform-architecture.md`](./persona-platform-architecture.md).

If you are evaluating Noos, start with the problem statement and the three
pillars in this README, then read the architecture spec for the binding
details.

---

## License

See [`LICENSE`](./LICENSE) if present in this repository, or contact the
maintainers.
