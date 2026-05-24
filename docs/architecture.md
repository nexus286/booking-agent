# Architecture

How the system is structured, and the architectural pivot that shaped it.

---

## The pivot, in brief

This project went through two significant architecture decisions. The first was wrong. The second was right *because* the first existed.

**v1 / v2:** Custom multi-agent system on LangGraph. Postgres-backed state machine with versioned compare-and-swap transitions. Mem0 for memory with a per-agent permission matrix. Custom undo queue with idempotency keys. Custom HITL framework with tiered escalation. Roughly 1,800 lines of specification across three review passes. Genuinely well-engineered for what it was.

**v3:** Built on Claude Managed Agents — Anthropic's hosted runtime for multi-agent orchestration. Multi-agent sessions, stateful conversation, built-in memory with cross-session curation (Dreaming), MCP connectors for integrations, observability via the Claude Console. Roughly 60–70% of the custom infrastructure became platform features.

The pivot happened mid-spec. Anthropic shipped Managed Agents in April 2026, while I was finishing the v2 specification. I evaluated, decided to pivot, threw out a meaningful amount of design work, and rebuilt the architecture around the new primitive.

This document explains why.

---

## What v2 was

The v2 architecture was a defensible custom build:

- **Orchestration:** LangGraph StateGraph with `interrupt()` for human-in-the-loop checkpoints
- **State:** Postgres with versioned pipeline records, compare-and-swap updates, locked-by tracking
- **Memory:** Mem0 with namespace permissions and write categories (verified_fact, user_preference, inferred_preference, external_claim, temporary_note)
- **Undo queue:** Custom table with idempotency keys, dedupe keys, polling worker, race condition handling under DB lock
- **HITL framework:** Custom interrupt + approval flow with tiered escalation (STANDARD / PRIORITY / URGENT) and quiet hours
- **Deployment:** Railway with persistent workers and cron
- **Identity resolution:** Custom fuzzy matching with confidence thresholds
- **Reply classification:** Custom LLM classifier with strict extraction schema

Everything was specified, reviewed, and architecturally sound. It would have shipped. The build estimate was 14–21 calendar days, 30–45 hours of active work.

---

## What changed

Anthropic launched Claude Managed Agents in April 2026. The key features that mapped directly to my v2 work:

- **Multi-agent orchestration as a platform primitive** — lead agent delegates to specialist sub-agents in parallel, automatic context management, shared filesystem
- **Stateful sessions** that resume after pauses and survive interruptions
- **Built-in memory** with Dreaming (cross-session memory curation)
- **MCP connectors** for integrations like Gmail, Calendar, Airtable
- **Observability** via Claude Console
- **Outcomes** — rubric-based grading where the agent revises output until it meets criteria

This wasn't a slightly-easier alternative. It was the *exact category of platform* I was building from primitives.

---

## The decision

The seductive move was to keep v2 and add Managed Agents as a "future improvement." This is how engineering teams end up shipping the version they started with instead of the version they should have built. Sunk cost masquerading as commitment.

The honest move was to evaluate whether v2 still made sense given the new option, then act on the answer.

**Three questions I asked:**

1. **Does v2 architecture still meet the actual requirements?** No. The product needed to be portable, self-serviceable by a non-technical user, and maintainable through conversation rather than code edits. v2's LangGraph + Postgres + Railway + Mem0 stack required ongoing engineering attention. The user would never maintain it herself.

2. **Is Managed Agents production-ready enough?** Beta, but improving fast. For a v1 to a single trusted customer who knows it's a beta product, the risk tier was acceptable. For a paying enterprise customer, it wouldn't be — yet.

3. **What does the pivot cost vs. continuing on v2?** Pivot cost: re-specifying around the new primitive (~3 days). Continuing cost: building infrastructure that already exists, then maintaining it forever, plus a worse product (less portable, harder to modify, more brittle).

The pivot was correct. I did it.

---

## What v3 looks like

**Architecture:**

```
                     ┌─────────────────────────────────┐
                     │  Chat UI (PWA on her phone)      │
                     └────────────────┬─────────────────┘
                                      │
                     ┌────────────────▼─────────────────┐
                     │  Application Shell               │
                     │  • Webhook handlers              │
                     │  • Approval flow                 │
                     │  • Undo window                   │
                     │  • Chat interface                │
                     │  • Auth                          │
                     │  (~1,000 lines Python)           │
                     └────────────────┬─────────────────┘
                                      │
                     ┌────────────────▼─────────────────┐
                     │  Claude Managed Agents           │
                     │  • Multi-agent orchestration     │
                     │  • Stateful sessions             │
                     │  • Memory + Dreaming             │
                     │  • Sandbox execution             │
                     │  • Observability                 │
                     │  (Anthropic hosted)              │
                     └─┬──────────┬──────────┬──────────┘
                       │          │          │
                ┌──────▼───┐ ┌────▼─────┐ ┌─▼────────┐
                │ Gmail    │ │ Calendar │ │ Airtable │
                │ MCP      │ │ MCP      │ │ MCP      │
                └──────────┘ └──────────┘ └──────────┘
```

**Six sub-agents** (Scout, Outreach, Scheduler, Logistics, Research, CRM) defined within Managed Agents. Coordinated by a super-agent the user talks to.

**Seven HITL checkpoints** surfaced as in-chat approve/reject prompts:

1. Weekly Scout shortlist approval
2. Pitch email review before send
3. Positive reply routing
4. Booking terms confirmation
5. Out-of-policy logistics escalation
6. Risk/controversy flag
7. Research brief sign-off

**90-second undo window** for any action that reaches the outside world (email send, calendar invite, contract send, SMS to external). Visible in chat with a countdown; cancelable by tap or by SMS reply.

**Memory model:** Three categories of stored knowledge:

- **Verified facts** (rep contact info, past appearances) — trusted, used freely
- **User preferences** (her stated rules, her voice profile) — trusted, applied automatically
- **External claims** (anything from inbound emails, web search) — surfaced for verification, not auto-applied

This is the prompt-injection boundary. Inbound content can be summarized, classified, and quoted, but never followed as instructions. Memory writes from external sources are tagged as such; agents reading them know to treat them as claims rather than truth.

---

## What we kept from v2

The pivot wasn't a rewrite from scratch. Most of the product thinking transferred cleanly:

- All four system prompts (Scout, Outreach, Research, Coordinator)
- The scoring model for Scout
- The seven HITL checkpoint definitions
- The reply classifier taxonomy (11 categories)
- The voice profile approach (placeholder voice, then ingest real samples post-demo)
- The intervention command set (/pause, /resume, /blacklist, /takeover, etc.)
- The prompt injection boundary
- The 90-second undo window
- The demo mode flag (no external actions fire during demos)
- The Apple-aesthetic design system

What changed was the *infrastructure*. The product is the same product. The chassis is different.

---

## What this taught me

**Pivot detection is a real skill, distinct from pivot execution.** Many people would have either ignored the new platform (committed to their plan) or rebuilt from scratch (over-corrected). The right move was surgical: keep what was correct, replace what wasn't. The skill is knowing which is which.

**Sunk cost is a tax on judgment.** I'd put three weeks into v2 design when Managed Agents shipped. The work was good. None of that mattered to the question "given today's options, what's the right architecture?" Throwing out good work because it's the wrong work is harder than it sounds.

**Specifying rigorously before building is what made the pivot survivable.** If I'd been mid-build instead of mid-spec, the pivot would have cost three weeks instead of three days. The spec-first discipline is what made the change cheap.

**Platforms compound faster than primitives.** Building from primitives (LangGraph, Postgres, custom workers) gives you control but locks you out of future platform improvements. Building on a platform means you inherit whatever Anthropic ships next. For a small project with a single customer, this asymmetry is overwhelming.
