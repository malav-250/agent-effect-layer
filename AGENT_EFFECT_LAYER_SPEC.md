# Project Spec — Transactional Effect Layer for AI Agents

Working name: **Ledger** (rename later — it's descriptive, not clever)

## One-sentence pitch

Durable execution restores your agent's state. It doesn't undo what your agent
already did to the world. This is the layer that does.

## The problem, stated precisely

Durable execution for agents is solved and crowded — Temporal, Inngest,
Restate, DBOS, LangGraph checkpointers, Azure Durable Task, Vercel
WorkflowAgent. They all checkpoint agent state and resume after a crash.

None of them solve what happens to **side effects already committed to the
outside world** when a run fails partway through. Published reference
architectures list what durable execution does *not* fix: hallucinations,
runaway loops, eval drift, store-level DR. Partial-effect rollback belongs on
that list too.

Concretely: an agent books travel. Step 1 reserves a flight. Step 2 books a
hotel. Step 3 fails. Checkpointing lets you resume at step 3 — it does nothing
about the flight you already reserved if the run is ultimately abandoned. The
agent has left the world in a half-finished state and nothing owns cleaning
it up.

## Thesis

Agent tool calls are distributed transactions without a transaction manager.
Give them one.

## Architecture

```
Agent (LLM)  ──proposes──>  Action Plan  ──persisted──>  Executor
                                                             │
                                          ┌──────────────────┤
                                          │                  │
                                    Policy Engine      Effect Journal
                                   (blast radius)      (what actually ran)
                                          │                  │
                                          └────> Tool Adapters <────┘
                                                      │
                                                 Real world
```

### 1. Two-phase execution — decision separated from action

The LLM proposes; it does not execute. The agent emits a structured action
plan, the plan is persisted, and a deterministic executor runs it. Published
guidance on agent idempotency names this separation as the architecture-level
fix: the decision phase is nondeterministic, the execution phase must not be.

Consequence that matters: a replay re-runs the *executor*, never the LLM. No
token spend on recovery, and no risk of the model deciding differently the
second time.

### 2. Idempotency keys derived from `(run_id, step_id)`

Deterministic derivation, never from LLM output. Same rule as the task queue's
`uq_jobs_idempotency_key`, with the same reasoning — the key must be stable
across replays or it isn't a key.

### 3. Compensating transactions — the core differentiator

Every tool registers an inverse:

Domain: **incident response.** An agent receives an alert and acts on a small
service fleet. Chosen because blast-radius classification is naturally
motivated here rather than contrived, and because it puts the project inside
the backend/SRE roles being targeted.

| Tool | Compensation | Class |
|---|---|---|
| `get_service_health` | none needed | reversible |
| `tail_logs` | none needed | reversible |
| `restart_container` | none needed (idempotent) | reversible |
| `scale_service(n)` | `scale_service(previous_n)` | compensable |
| `drain_node` | `uncordon_node` | compensable |
| `rollback_deployment` | `redeploy(previous_sha)` | compensable, asymmetric |
| `post_status_page_update` | `post_correction` | **irreversible** |
| `page_oncall` | none possible | **irreversible** |

The two irreversible cases are the interesting ones and should drive the
approval-gate design. You cannot un-tell customers you had an outage, and you
cannot un-wake someone at 3am. A correction post is not an inverse — it is a
second public statement.

On abort, the executor walks completed steps in reverse and runs each
compensation, journaling the outcome. Saga pattern, applied to agent tool
calls.

The hard cases are the interesting ones and the post should say so:
compensation can itself fail; some compensations are asymmetric (a refund is
not the inverse of a charge — the money moved twice and both show on a
statement); and some actions have no inverse at all.

### 4. Policy engine — blast radius classification

Every tool is tagged:

- **reversible** — no side effect, or trivially undone. Runs freely.
- **compensable** — has a registered inverse. Runs, journaled for rollback.
- **irreversible** — no inverse exists. **Suspends the run for human approval.**

This is where the design judgment lives, and it's what an interviewer will
probe. The classification is a policy decision, not a technical one.

### 5. Budget circuit breakers

Token, cost, and step ceilings per run. Trip → halt the loop → journal the
reason. Reuses the circuit-breaker pattern from the task queue. Runaway loops
are explicitly named as something durable execution does not solve.

### 6. The effect journal

Append-only record of every effect actually committed: tool, arguments hash,
idempotency key, timestamp, result, compensation status. This is the artifact
that makes the whole thing auditable — and it's queryable SQL, same argument
as the task queue's DLQ-as-a-table.

## The measurement story

This is what makes it a project rather than a demo. Reuse the `kill -9`
methodology from the task queue:

1. **Exactly-once under crash.** Kill the worker mid-tool-call, in a loop, while
   a producer drives runs. Assert: each effect committed exactly once. Count
   *effects in the journal against effects observed at the tool boundary* — not
   duplicates, which is the harness mistake documented in the earlier post.
2. **Compensation completeness.** Force failures at every step index. Assert
   every prior effect has a matching compensation entry.
3. **Compensation failure handling.** Make a compensation itself fail. Assert
   the run lands in a terminal state that a human can find — this is the DLQ
   argument again.
4. **Budget enforcement.** Drive a deliberately looping agent. Assert the
   ceiling trips and the run halts.

Every number published must carry its protocol, per the standing rule.

## The demo — the part that converts

A dashboard with a **"crash the worker" button**. A visitor clicks it, watches
the process die mid-run, and watches recovery complete without double-sending.

Visible: run timeline, per-step idempotency keys, effect journal, compensation
status, budget consumption.

Every project on the site has a `[Code]` link. Almost none have a button that
proves the central claim in ten seconds. This is the highest-conversion element
of the whole project and it should be built early, not last.

## Stack

Deliberately boring, deliberately aligned with the target roles:

- **Python + FastAPI** — matches the task queue, matches the Crewasis stack
- **Postgres** — effect journal, run state, idempotency constraints
- **Redis** — budget counters, circuit breaker state
- **One LLM provider** — do not build a provider abstraction
- **Docker Compose** — reproducible local run, no cloud dependency to demo
- **Prometheus + Grafana** — effects committed, compensations run,
  compensation failures, budget trips

No Kubernetes, no Kafka, no multi-agent orchestration. Those are gaps to close
separately, not scope for this.

## Scope control

**Build in this order.** Each stage is demoable on its own:

1. Action plan persistence + deterministic executor + idempotency keys
2. Effect journal + a dashboard that renders it
3. Three tools with registered compensations
4. Abort path — reverse walk, compensations run
5. **The crash button**
6. Policy engine + approval gates for irreversible actions
7. Budget circuit breakers
8. Adversarial harness + measurements

Stop after 5 and it's already a strong project. Everything after is upside.

**Explicitly out of scope:** multi-agent coordination, a tool marketplace,
provider abstraction, a UI for authoring agents, anything resembling a
framework for other people.

The failure mode is building a framework instead of a system. Keep one real
agent doing one real task running on top of it at all times.

## Timeline

5-7 weeks at a realistic pace alongside coursework. Stages 1-5 are roughly
three weeks and produce something shippable.

## Where it points

- Case study on the portfolio, same five-section template
- Blog post: "Your agent framework checkpoints state. It doesn't un-send the
  email." — the compensation asymmetry argument
- Resume bullet, once the measurements exist and carry their protocol

## Facts to verify before publishing anything

- That the named durable-execution frameworks do not provide compensation —
  check Temporal's saga support specifically, since Temporal *does* have
  compensation primitives and the claim needs to be narrower than "nobody
  does this"
- Any statistic about agent adoption or failure rates
- Any comparison to a named product
