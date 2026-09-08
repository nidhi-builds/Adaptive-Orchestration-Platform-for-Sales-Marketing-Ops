# Implementation Plan v2 — Stage-Gated
## Adaptive Orchestration Platform for Sales & Marketing Ops

**Rule of this plan:** a stage is not "done" because time was spent on it or code was written for it. It is done when its Gate — a specific, measurable check — passes. If a gate fails, the stage is not complete, regardless of the calendar. This is deliberate: the router's credibility (and yours, in a defense) rests on being able to say "here is the check it passed," not "I built it."

---

## Development Workflow: Claude (Free) + ChatGPT Plus (Codex)

This matters more than it looks, because the two tools are not interchangeable and using them wrong wastes your limited budget on both sides.

**What each tool actually is, as of now:**
- **Claude (free)** — chat, web search, code execution (single-script/sandboxed, not a persistent multi-file repo), file creation, memory, MCP connectors. No Claude Code (that needs Pro/Max). This makes it the right tool for *design, specification, one-off scripts, and code review* — not for running a live multi-file agent codebase over weeks.
- **ChatGPT Plus (Codex)** — a real agentic coding tool: each task runs in a cloud environment preloaded with your repository, where it reads/edits files, runs tests, and iterates independently before returning results. This is your actual implementation engine. The catch: Plus has a **5-hour rolling usage cap** shared across Codex and related agentic features — practically, a few focused sessions per week, not unlimited use.

**The workflow that respects both constraints:**
1. **Specify here, in Claude, before opening Codex.** Take the relevant stage from this plan, and turn it into a precise task brief (file(s) to touch, exact function signatures, the gate it must pass) *before* spending Codex budget. Claude's free-tier code execution is a good place to prototype tricky logic in isolation first (as we just did with the data generator) — hand Codex a working reference implementation to integrate, not an open-ended "build the router" prompt that burns budget exploring.
2. **Batch Codex sessions around one stage at a time.** Given the rolling cap, don't split one stage across many short sessions with re-explained context — go in with the stage's full brief loaded, work until the stage's gate can be tested, then stop.
3. **Bring the diff back to Claude for review.** Paste the Codex-generated code (or a summary of it) back into this chat for a second read — a different model family catching different classes of bugs is a real, not superstitious, benefit. Use this pass specifically to check the diff against the stage's gate criteria before you spend Codex budget running it.
4. **Reserve Codex budget for the hard stages.** Stage 0 (environment scaffolding) and parts of Stage 6 (integrations) are largely boilerplate — draft that here via Claude's code execution first (schemas, docker-compose, FastAPI skeletons), and let Codex start from a working skeleton rather than from zero. Save Codex's limited sessions for Stage 2 (the router logic) and Stage 3 (agent execution + approval gate), which are where iterative debugging actually needs a real execution environment.
5. **Never let a gate be graded by the same tool that built the stage.** If Codex implemented the router, the router's gate (Stage 2 below) should be run and read by you, with Claude helping you interpret the results — not by asking Codex "did I pass?"

---

## Synthetic Sales Customer Data — Methodology

A working generator (`generate_synthetic_leads.py`) has already been built and validated against this methodology — see the attached file. The reasoning behind it, so you can extend it rather than treat it as a black box:

- **Non-uniform, weighted distributions**, not clean/tidy numbers: company size is weighted 60% SMB / 30% mid-market / 10% enterprise; reply probability is drawn from a *mixture* of rates (0.03–0.14) rather than one flat number, because real outreach performance varies by segment, not by a single constant.
- **A generated ground-truth temporal history per lead** — every status change carries `valid_at`/`invalid_at`, which is exactly what your Zep/Graphiti layer needs, and because you generated it, you know the *correct* answer and can check the memory layer's retrieval against it directly, not just eyeball plausibility.
- **Inter-touch and reply delays drawn from a skewed distribution** (exponential, via `random.expovariate`), not a uniform random range — most follow-ups happen quickly, a long tail waits weeks, which is what real engagement timing looks like and what would actually stress-test your adaptive loop's termination logic.
- **Deliberate messiness, not a happy-path dataset:** ~4% malformed records (missing fields, to test robustness/error handling) and ~3% near-duplicate contacts (to test the CRM Hygiene agent specifically) are injected on top of the clean simulation.
- **Validated against a real-world sanity check, not just internal consistency:** the script prints reply-rate and conversion-rate summaries against a stated real-world benchmark range (cold outreach reply rates of roughly 1–10%), so an unrealistic dataset is caught before it quietly inflates your evaluation numbers. At current settings, the generator produces an 18.1% cumulative reply rate (compounded across up to three touches) and a 1.7% conversion rate — a re-engagement campaign running somewhat above cold-outreach baseline, which is defensible, not inflated.
- **Reproducible by seed** (`SEED = 42`) — this is what makes the four-configuration comparison (Stage 5) a controlled experiment: every configuration runs against the exact same 360-record dataset, so differences in the results are attributable to the routing strategy, not to random dataset variation.

---

## Stage 0 — Environment & Tooling Setup

**Build:** repo skeleton, TiDB Cloud cluster, Redis (Upstash), FalkorDB container, OmniRoute running locally, Langfuse account.

**Gate (all five must pass in one script run):**
1. TiDB: an INSERT followed by a SELECT returns the inserted row.
2. Redis: a SET followed by a GET returns the set value.
3. FalkorDB: a Cypher write followed by a read returns the written node.
4. OmniRoute: one completion call returns a non-empty response with a cost header present.
5. Langfuse: that OmniRoute call appears as a trace in the dashboard.

No partial credit. If any one fails, Stage 0 is not complete — do not proceed to Stage 1 on "four out of five."

---

## Stage 1 — Synthetic Data + Single-Agent Vertical Slice

**Build:** run `generate_synthetic_leads.py`; build the Lead Research/Enrichment Agent against it; TiDB schema for `leads`, `messages`, `agent_runs`.

**Gate:**
1. Dataset validation script (built into the generator) reports distributions within the stated realistic ranges — no manual eyeballing, read the printed stats.
2. The enrichment agent processes 50 sampled leads with a hard-failure rate (unhandled exceptions) of ≤5%.
3. A human spot-check of 10 enrichment outputs scores ≥4/5 average plausibility (you rate them yourself against the source record — this is a real, if small, human evaluation, not skipped).
4. 100% of these runs appear as Langfuse traces — no silent, untraced calls.

---

## Stage 2 — Task Decomposer + Adaptive Router (the core deliverable — do not shortcut this gate)

**Build:** the five-topology decision logic (single / parallel / workflow / loop / debate).

**Gate:**
1. Build a hand-labeled test set of **20 synthetic goal/task scenarios**, 4 per topology, each with a human-assigned "correct" topology label written *before* running the router (to avoid post-hoc rationalization).
2. The router's chosen topology matches the human label on **≥80% of the 20 cases**. This is the single most important number in the whole project — it is your router actually being evaluated, not just existing.
3. 100% of routing decisions have a non-empty logged rationale field.
4. A 20-run stress test of the adaptive-loop path shows **zero infinite loops** — every run hits an explicit termination condition (confidence threshold, max-iteration cap, or no-response cutoff).

If the 80% threshold isn't met, the correct response is to revise the router's decision heuristics and re-run the same 20-case test — not to lower the threshold after the fact.

---

## Stage 3 — Specialist Agents + Approval Gate

**Build:** outreach drafting, campaign strategy, CRM hygiene agents; Escalation/Approval Agent; LangGraph checkpointing.

**Gate:**
1. An end-to-end run across the full 360-record dataset completes for **≥90%** of records without unhandled exceptions.
2. **10 real approval-pause events** are triggered and resolved, with **100% correct resume-from-checkpoint** behavior (verified by log inspection — no run silently restarts from scratch after an approval).
3. At least 3 of the 4 non-debate topologies are exercised at least once in this run, confirmed correct by inspecting the decision log, not just by the run completing.

---

## Stage 4 — Frontend (Command Deck)

**Build:** wire the existing mockup to live WebSocket data; approval queue; goal intake.

**Gate:**
1. Across 10 tested state transitions, the live task graph reflects the backend state within **2 seconds**.
2. 10/10 test approve/reject clicks produce a correct backend state change reflected back in the UI.
3. A 30-minute soak test shows **zero silent WebSocket disconnects**.

---

## Stage 5 — Evaluation Harness (your primary evidence)

**Build:** run the full dataset through four configurations — always-single-agent, always-parallel, fixed-workflow-only, adaptive router.

**Gate:**
1. All four configurations complete a full run against the same seeded dataset.
2. Cost (tokens/$), latency, and task success rate are computed from **real Langfuse and OmniRoute logs**, not estimated or hand-waved.
3. A comparison table is produced and the result is reported honestly, including any case where the adaptive router does not win — completeness of the comparison is the gate, not a specific outcome.

---

## Stage 6 — Integrations (treat as descopable, not silently skippable)

**Gate:** at least one real sandbox integration (e.g., HubSpot developer account) completes one real read + one real write. If time does not allow this stage, it must be explicitly marked "descoped" in your report — not quietly dropped without comment.

---

## Stage 7 — Deployment, Hardening, Defense Prep

**Gate:**
1. The full demo script runs successfully end-to-end on **two separate days**, not once — a single lucky run is not evidence of reliability.
2. A deliberately induced rate-limit condition (e.g., simulate hitting the NIM free-tier cap) triggers the correct alert/fallback behavior.
3. The final dry run is completed **at least 72 hours** before the actual demo or defense date.

---

## Attached: `generate_synthetic_leads.py`
Working, seeded, validated generator described above — run it directly to produce `leads.json` for Stage 1 onward.

---

## Addendum A — Verification Gate Design (how "owning the environment" is actually enforced)

This was previously described conceptually; it now needs to be a concrete, checkable schema, not a principle you hope agents follow.

Every action a specialist agent takes is tagged, at design time, with one of three verification tiers. The harness — not the model — enforces which tier applies and blocks progress until it is satisfied.

**Tier 1 — Read-back verification (cheap, automatic, applies to nearly all writes).**
After any write (TiDB record, Zep fact, CRM field), the harness immediately issues a read against the same record and diffs it against the intended write. No model reasoning involved — this is mechanical and near-zero cost. Logged as `{action_id, claimed_write, readback_result, match: bool}`.

**Tier 2 — Semantic verification (LLM-as-judge, moderate cost, applies to generated content).**
Before an outreach draft, ad creative, or reply is allowed to proceed toward sending, a second, cheap model call checks it against a fixed rubric (no fabricated claims, no factual mismatch with the lead record, tone within bounds). This is the "reviewer agent" pattern from the design discussion, now formalized as a mandatory gate rather than an optional step. Logged as `{action_id, rubric_scores, pass: bool}`.

**Tier 3 — Outcome verification (delayed, ground-truth, applies to goal-level claims).**
The Evaluator/Monitor does not accept "all subtasks returned success" as evidence the goal was met. It re-queries the actual state (demos booked, replies received) against the stated goal on its own schedule, independent of what any agent reported. This is what makes the goal-completion check in your end-to-end trace a real check rather than a rubber stamp.

**Every logged action carries a `verification_result` field** (`{tier, verified: bool, method, discrepancy_detail}`). This is not just a debugging aid — it becomes a first-class metric in Stage 5: *percentage of actions with a confirmed postcondition vs. an assumed one*. A system that owns its environment should trend toward a high percentage here; a regression in this number is itself a signal worth investigating even if task success rate looks fine.

---

## Addendum B — Cross-Session Context Management (keeping every new prompt lightweight)

**In one sentence:** never replay history — re-derive a compact working state from indexed storage, sized only to what the current action needs, and only pull raw history when a specific query explicitly asks for it.

Mechanism, concretely:
1. **Session summaries, not session replays.** At the end of each session (or every N turns), the Context Manager writes a short summary (a few hundred tokens: decisions made, open items, key new facts) to TiDB. This is a distinct artifact from the raw `messages` table, which remains a full, untouched audit log that is never fed back wholesale into a prompt.
2. **A bootstrap step on every new prompt**, not a full-history load: pull (a) the single latest session summary, (b) task-graph nodes still in `pending`/`in_progress` state relevant to the new prompt, via a filtered TiDB query, and (c) a Zep query scoped only to entities the new prompt actually mentions — never a full graph dump.
3. **Raw history is retrieved only on explicit request** — a tool call like "show me exactly what was said in the approval thread for lead X" is a deliberate, targeted retrieval, not something that happens by default.
4. This is the multi-session generalisation of the Write / Select / Compress / Isolate framework from the context-engineering research: "Write" becomes the session summary, "Select" becomes the scoped bootstrap query, applied across session boundaries rather than only within one agent run. The practical effect is that context size becomes a function of *current relevance*, not *relationship length* — a user on day 200 gets a similarly sized prompt to a user on day 2.

---

## Addendum C — Long-Horizon Task Accuracy: What the Evidence Says, and What to Measure

This was not explicitly covered before and should be treated as real methodology, not an afterthought, because the evidence is sobering and directly relevant to a multi-step orchestration system.

**What the research shows.** METR's time-horizon research defines an agent's capability as the length of task (measured in the time a skilled human would need) it can complete with 50% reliability, and finds this horizon has been doubling roughly every seven months since 2019 — but the number that matters more for a business-critical system is the **80%-reliability horizon**, which is substantially shorter in absolute terms than the 50% headline number for the same model. Separately, frontier models were found to achieve near-100% success on tasks taking a skilled human under ~4 minutes, but under 10% success on tasks taking more than ~4 hours — and per-step failure risk compounds, so success drops roughly exponentially, not linearly, as task length grows. A related finding, directly relevant to your context-management design: long-context web-agent studies found success rates of 40–50% on short-horizon task variants fell below 10% once the same task was embedded in a longer interaction history, even when the needed information was technically still present in context — evidence that context bloat degrades accuracy independent of context-window size, reinforcing why Addendum B matters practically, not just for cost.

**What this means for your goal ("re-engage leads, book 15 demos").** That goal is not one task — it is a long-horizon composite of many short subtasks. The relevant question is not "can the model do this," it is "what is the compounded, end-to-end reliability across the whole chain of subtasks it takes to get there." If each subtask has a 95% success rate and a goal chains 20 of them, naive compounding puts full end-to-end success meaningfully below 50% — which is precisely the argument for decomposition, verification gates, and re-planning rather than one long unsupervised run, and it is a genuine justification for your adaptive router's existence, not just a design preference.

**What to add to the Stage 5 evaluation harness, concretely:**
- Measure success rate **as a function of chain length** (how many subtasks deep a goal is), not just overall — plot it, don't just report one aggregate number.
- Measure **variance across repeated runs of the same goal**, not only mean success rate — a goal that succeeds 70% of the time via a consistent, predictable strategy is a materially different (better) system than one that succeeds 70% of the time via a bimodal split of full success or catastrophic failure, even though the headline number is identical.
- Report an explicit **reliability-at-length curve** for your system, the same shape as METR's time-horizon curves, as a defensible, citable evaluation artifact for your report.

---

## Addendum D — Live Verification Protocol for Stage 2 / Stage 3 (hands-on, while you implement)

Since you'll be personally verifying the core implementation as it happens, use this checklist per work session rather than only at the end of a stage:

1. **Before running anything:** open Langfuse's trace view and OmniRoute's cost dashboard side by side with your terminal — you want to watch both live, not check them after the fact.
2. **For every router decision made during the session:** confirm, in the trace, that (a) a topology was chosen, (b) a rationale string is non-empty, (c) a confidence score is present. If any of the three is missing on a single run, stop and fix the logging before writing more logic — a router you can't audit in real time is not one you can trust later.
3. **For every tool call an agent makes:** check the Tier 1 read-back result (Addendum A) actually fired and matched. A tool call with no corresponding verification entry is a bug, not a detail to fix later.
4. **Watch latency per call, not just total run time** — a single slow call hiding inside a "fast enough" total is exactly the kind of thing that only surfaces under load later; note any call over ~3–5 seconds and ask why before moving on.
5. **Watch token count per call against your tiered-routing plan** — if a structured, supposedly cheap-tier task is landing on your frontier-tier model, that's a routing misconfiguration to catch immediately, not at the Stage 5 cost-comparison stage.
6. **At the end of each session, check the adaptive-loop termination condition manually on at least one run** — step through why it stopped, don't just confirm that it did.

This is deliberately slower than "run it and check the final output" — that's the point. The gates in this plan exist so that verification happens continuously during implementation, not retroactively during a demo rehearsal when it's too late to fix cheaply.

---

## Stage 8 — Production Hardening (what separates this from a prototype)

Everything through Stage 7 proves the system *works*. Nothing through Stage 7 proves it *survives contact with real, concurrent, occasionally adversarial usage*. Stage 8 exists to close that gap explicitly, and is treated as its own gated stage, not a polish pass.

### 8.1 System design changes required first

| Loophole | Fix |
|---|---|
| Race conditions on shared lead/task-graph records | Optimistic locking (row version column) on every shared-state table in TiDB; a write against a stale version is rejected, not silently overwritten |
| Duplicate real-world side effects on retry | Idempotency keys on every external-effect action (send, spend, CRM write), stored in a dedupe table; a repeated key is a provable no-op |
| Retry storms under upstream failure | Exponential backoff with jitter and circuit breakers around OmniRoute and all external API calls |
| Unbounded cost/fan-out | Hard, enforced ceilings — max agents per goal, max tokens per goal, daily spend cap that halts execution automatically, not just alerts |
| Approval policy bypassable via prompt injection | The approval-required decision is made by deterministic rule-based code, never by an LLM call judging untrusted content |
| PII with no deletion path | A cascade-delete routine spanning TiDB, Zep, and Redis for a given lead; Langfuse trace redaction limitations documented honestly, not glossed over |
| No graceful degradation per dependency | Each external dependency (FalkorDB, OmniRoute, CRM/ad APIs) has a defined fallback behaviour, not a shared failure mode |
| Secrets in plain environment variables | AWS Secrets Manager, least-privilege IAM roles |
| Unbounded approval-queue backlog | Staleness policy: auto-escalate or flag any pending approval older than a defined threshold |

### 8.2 Deployment (AWS, $100 credit budget)

- **First action, before anything else is provisioned:** an AWS Budget alarm at $20 / $50 / $80.
- Backend: **AWS App Runner** (not ECS+ALB — an ALB's ~$16/month fixed cost is avoided entirely; App Runner scales to zero and includes HTTPS).
- Task queue: **AWS SQS**.
- Secrets: **AWS Secrets Manager**.
- Observability: **CloudWatch** for infrastructure-level failures, alongside Langfuse for LLM-level tracing — they see different failure classes and neither substitutes for the other.
- Database, cache, and frontend remain on TiDB Cloud, Upstash Redis, and Vercel — already free-tier and production-capable at this scale; migrating them would spend budget without closing a real gap.
- Estimated cost: **~$10–20/month**, leaving headroom across the remaining project timeline.

### 8.3 Gate (new test categories — this is what actually differentiates a production-tested system from a prototype)

1. **Load test** (Locust/k6): 20/50/100 concurrent goal submissions; explicit p95 latency and error-rate thresholds defined and met, not just observed.
2. **Concurrency test:** two simultaneous approval resolutions fired at the same pending action; exactly one applies, verified by inspecting the row version, not by the run merely completing.
3. **Idempotency test:** the same send action fired twice; exactly one real-world effect occurs.
4. **Chaos test:** FalkorDB killed mid-run; the system degrades gracefully (flags reduced temporal accuracy) rather than hanging or corrupting state. Repeated for a simulated upstream model-provider outage.
5. **Cost-ceiling test:** a deliberately pathological input designed to maximise fan-out; the hard cap halts execution, confirmed by log inspection, not just by config existing.
6. **Security red-team test:** lead-reply content containing injected instructions attempting to bypass approval; the deterministic policy is confirmed unmoved by the injected content.
7. **Data-deletion test:** one synthetic lead's data deletion is requested; what is actually removed across TiDB/Zep/Redis is verified, and what remains in Langfuse traces is documented honestly rather than claimed clean.
8. **Extended soak test:** multi-hour run (not 30 minutes) checked for memory leaks and connection-pool exhaustion.

**This stage's gate is not "all tests pass on the first run."** It is that every test in this list has been run at least once, with its result — pass or documented failure with a fix plan — recorded. A system that fails the chaos test but documents exactly how and why is more credible in a defense than one that never ran the test at all.


