# Project Context — for a new Claude chat

Paste this whole file as your first message in a fresh chat, then ask for
whatever you need next (the Claude Code kickoff prompt, repo setup, scope
questions, a blog post, resume bullets).

---

## Who I am

Malav. MS Computer Software Engineering at Northeastern, graduating **December
2026**. Targeting **backend / cloud / infrastructure / SRE roles starting
January 2027**, full-time, not seeking internships. Need visa sponsorship.
Based in Boston.

Portfolio: https://malavgajera.is-a.dev
GitHub: https://github.com/malav-250

### Background that matters here

- **Software Engineer Co-op, Crewasis** — Jan 2026 to May 2026, New York.
  Django/FastAPI backend, Docker, GitHub Actions, AWS, Postgres, Redis.
- **Software Developer Intern, Tatvasoft** — two stints, Ahmedabad, India.
  Jun–Aug 2023 (3 mo) and Jan–May 2024 (5 mo). C#/ASP.NET Core, Entity
  Framework, SQL Server, Azure DevOps.
- **Research Assistant, Nirma University** — Mar to Dec 2023, under Prof. Anuja
  Nair. Reinforcement learning for vehicle collision avoidance (SUMO), and lung
  sound disease classification (EfficientNet-B0 + attention, ICBHI dataset).
  Not published.

### My strongest existing project — this new one builds on it

**Distributed Task Queue** (`github.com/malav-250/distributed-task-queue`) —
FastAPI, Celery, RabbitMQ, Redis, Postgres, Prometheus.

What it actually does, because it's the foundation for the new project:
- `task_acks_late` + `task_reject_on_worker_lost` for at-least-once delivery
- **Application-level dead-lettering**, not RabbitMQ DLX — a Celery `on_failure`
  hook classifies the exception, writes `DEAD_LETTERED` to Postgres, and
  forwards to a dedicated queue. The DLQ is a SQL table, so it's queryable and
  paginated; replay is an HTTP call, not a republish loop.
- **Completion-gated idempotency** — `before_start` skips only if status is
  already `COMPLETED`; `on_success` writes the terminal state *after* the work.
  `running` is a claim that doesn't suppress retries; `completed` is a
  commitment that does.
- Redis sliding-window rate limiting, Redis circuit breakers
- Measured: zero duplicate executions across 10K jobs under `kill -9` on
  workers in a loop
- Known limits I state openly: no mutual exclusion on `running`, `on_success`
  commits in its own session so a narrow work→marker window persists,
  at-least-once not exactly-once

---

## The new project

**Transactional effect layer for AI agents.** Working name: Ledger.

**Thesis:** Durable execution restores your agent's state. It doesn't undo what
your agent already did to the world.

Checkpointing lets a crashed agent resume at step 4. It does nothing about the
email sent at step 1 or the status page updated at step 3. The agent
frameworks hand you idempotency keys and stop there. This layer adds
compensation.

**Domain:** incident response. An agent reacts to an alert and acts on a small
service fleet — restart a container, scale a service, roll back a deploy, post
to a status page, page on-call. Chosen because blast-radius classification is
naturally motivated in this domain, and because it puts the project inside the
roles I'm targeting.

**Core pieces:** two-phase execution (LLM proposes, deterministic executor
runs), idempotency keys from `(run_id, step_id)`, compensating transactions
with registered inverses, a policy engine classifying tools as
reversible/compensable/irreversible with approval gates on the last, budget
circuit breakers, and an append-only effect journal in Postgres.

**The demo that converts:** a "crash the worker" button. A visitor clicks it,
watches the process die mid-run, and watches recovery complete without
double-sending.

**Stack:** Python, FastAPI, Postgres, Redis, Docker Compose, Prometheus +
Grafana, one LLM provider. No Kubernetes, no Kafka, no multi-agent.

**Timeline:** 5-7 weeks. Full spec is in `AGENT_EFFECT_LAYER_SPEC.md`.

### The open question that gates everything

Temporal has saga and compensation primitives, so "nobody solves compensation"
is too strong a claim. The narrowest true version needs verifying before I
build on the premise. Hypothesis: general-purpose durable execution engines
have compensation, but the *agent* frameworks (LangGraph checkpointers,
WorkflowAgent, Azure Durable Task for agents) give you resume without rollback
— and nobody wires compensation to a blast-radius policy over LLM-proposed tool
calls. **If that's wrong, the project needs repositioning, not building.**

---

## Three standing rules I work under

These came out of a real failure. I published a blog post describing a
RabbitMQ Dead Letter Exchange architecture I had not built — my repo does
application-level dead-lettering. Zero grep hits for `pika`,
`x-dead-letter-exchange`, `x-death`. It sat live for four months, linked to the
repo that contradicted it. The same fabricated mechanism had propagated to my
portfolio case study and my resume.

1. **Greppability** — every identifier in a claim must be greppable in the
   artifact the claim describes.
2. **Sourced figures** — every figure traces to a primary source or a
   measurement I took, with a date. Machine-readable feeds over rendered pages.
3. **Metric + protocol** — a metric ships with its evaluation protocol or not
   at all.

Rule 3 exists because my lung-sound research reports ~92% accuracy on a
**cycle-level split with augmentation applied before splitting** — both leak,
since ICBHI has 126 patients and a cycle-level split puts the same patient on
both sides. That number never appears without its caveat.

### A harness mistake worth not repeating

My task queue test counted *duplicates* and found zero. A duplicate-counting
harness structurally cannot see a job that ran zero times. For the new project,
count effects in the journal **against effects observed at the tool boundary** —
completions against submissions, not duplicates.

---

## How I like to work

- Preview locally before anything ships. I review, then I say deploy.
- One stage at a time, finished, before the next.
- Tell me when something I've asked for is a bad idea. I'd rather be told than
  agreed with, and I act on hard feedback.
- The failure mode I'm most worried about on this project: **building a
  framework instead of a system.** Keep one real agent doing one real task
  running on top of it at all times.

---

## What I want from this chat

I'm running the build in a separate Claude Code session. Use this chat for
planning, prompts, research, and writing. Specifically I'll be asking for:

- The Claude Code kickoff prompt for each stage
- Repo setup and structure decisions
- Verification research when a claim needs a primary source
- The case study and blog post when the project is far enough along
- Resume bullets once measurements exist

Ask me what I need rather than assuming.
