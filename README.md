# booking-agent

**An AI booking assistant for talent bookers in sports media. Built with Claude.**

A conversational AI agent that handles guest research, outreach drafting, scheduling, logistics, host briefs, and CRM on behalf of a senior talent booker at a major national sports show. She talks to it in plain English; it does the work; she approves anything that touches a real person.

The interesting part isn't the product — it's how it was built. Designed across three review passes (one self-review, two adversarial reviews using GPT-5.5 Thinking to catch blind spots), pivoted mid-spec when Anthropic shipped a new primitive that made 60–70% of the custom infrastructure obsolete, built with Claude Code.

**Status:** Spec complete, build in progress. Demo imminent.

---

## What this is

A bespoke AI assistant for one professional. The user is a senior talent booker — expert at her job, constrained only by bandwidth. Existing booking software solves the wrong problem (it's mostly CRM databases). What she needed was an assistant that could do the operational work, not a tool that organized it.

The system handles six functional areas through specialized sub-agents coordinated by a single super-agent the booker talks to:

| Sub-agent | Role |
|---|---|
| Scout | Monitors sports news 24/7, scores guest candidates 0–100, surfaces a ranked weekly shortlist |
| Outreach | Contact enrichment, pitch drafting in her voice, 3-touch follow-up sequences |
| Scheduler | Calendar negotiation with reps, confirmations, automated reminders |
| Logistics | Travel, hotel, ground transport, green room, day-of monitoring |
| Research | Pre-show briefs for the host: 20 segment questions, no-go zones, producer notes |
| CRM | Relationship tracking, rebooking scores, weekly pipeline reports |

She never sees the routing. She types "run this week's scout" and gets back a ranked list. She approves three; drafts come back in her voice; she approves the drafts; emails go out with a 90-second undo window in case she changes her mind.

Every external action — every email, calendar invite, contract — requires her explicit approval. Seven human-in-the-loop checkpoints govern the workflow. Nothing reaches a real person without her clicking through.

---

## Why I built it

Three reasons, in order of honesty:

**She's a friend with a real problem.** Senior bookers in sports media have relationship graphs that take fifteen years to build. They can only place so many guests on so many shows. AI doesn't replace the relationships — it amplifies the throughput. If she can run two shows instead of one, three instead of two, eventually her own boutique agency, that's a real change in her career trajectory.

**I wanted to test a hypothesis about AI-native work.** The conventional wisdom is that AI tools amplify good engineers. I wanted to test whether AI tools could collapse the engineer/PM/strategist split for small projects — whether someone with strong product judgment and discipline about specification could ship what previously required a team. This project is my test case.

**The booking industry is structured around big agencies (CAA, WME) with huge headcounts.** A talented solo booker can't compete on volume. With an AI system that handles the operational tier, a solo booker can run a multi-show roster with the relationship quality only individuals can offer. The competitive moat compounds — her relationship graph plus the agent's accumulated memory is something agencies can't easily replicate. This isn't just an internal tool; it's a business thesis.

---

## How I built it

I'm a sales professional, not a working engineer. I currently sell AI products to enterprise customers at Google Workspace, and over fifteen years of enterprise SaaS sales I've worked closely enough with engineering teams to direct technical work, even though I don't write production code myself. I've written HTML, CSS, JavaScript, and a fair amount of SQL; I can read Python well enough to follow what's happening. The thousands of conversations with developers and DBAs across my career taught me to evaluate technical decisions in real time, which is the skill that matters for directing AI tools.

For this project, the methodology was:

1. **Spec rigorously before writing code.** I wrote roughly 1,800 lines of specification before opening Claude Code. This felt slow. It made the build itself fast.

2. **Run adversarial AI reviews.** I sent the spec to a different model (GPT-5.5 Thinking) twice and asked it to be harsh. The first pass caught real architectural gaps — identity resolution, prompt injection vulnerabilities, state concurrency edge cases. The second pass after my revisions caught a few more. I filtered the feedback against product context, accepted some, rejected some with reasoning.

3. **Pivot when the ground moves.** Mid-spec, Anthropic launched Claude Managed Agents — a hosted runtime for exactly the multi-agent orchestration I was building from primitives. Roughly 60–70% of my custom infrastructure became obsolete overnight. I killed the v2 build before starting it and rebuilt the architecture around the new platform. Painful, but the right call.

4. **Build with Claude Code.** Spec-driven, slice-by-slice. Build the spine (state, approval flow, undo queue) before the agents. Test each piece. Refuse to skip ahead.

5. **Document the methodology, not just the product.** This repo isn't just code; it's a record of how the project was built. See `docs/` for the specification, architecture decisions, methodology, and learnings.

---

## Technical approach

**Platform:** Claude Managed Agents (Anthropic's hosted multi-agent runtime). The platform handles multi-agent orchestration, stateful sessions, memory curation across sessions, sandbox execution, and observability. I build a thin application shell on top.

**Primary model:** Claude Opus 4.7 for coordinator reasoning and complex tasks; Claude Haiku 4.5 for routing and low-stakes calls.

**Application shell:** Python (~1,000 lines). Defines the agents, handles webhooks, manages the chat UI the booker talks to, surfaces the 90-second undo window for any action that touches the outside world.

**Integrations via MCP connectors:** Gmail, Google Calendar, Airtable. Reduces setup friction for the booker to ~80 minutes from zero to working assistant.

**Deployment:** Railway. One-click deploy from this repository.

**Ownership model:** She owns her Anthropic account and Railway deployment. I set it up with her once, then walk away. The repository is a fork-and-deploy kit; she can modify it with Claude Code if she wants behavior changes. If she ever wants to run it without me, she already does — the system runs on her infrastructure, on her dime.

---

## Design principles

A few specific decisions that shaped everything else:

**Conversation as the primary interface.** The user is tech-skeptical; she won't adopt anything that feels like "using software." The interface is chat. Configuration happens by talking to the assistant, not by editing settings. When she says "stop pitching anyone from a specific agency in December — they're saturated," the system stores that as a rule and respects it forever.

**Trust over throughput.** Seven HITL checkpoints, not three. Fewer checkpoints would be smoother UX, but it would undermine the trust architecture. She has to know — absolutely — that nothing escapes her oversight. Trust is product; trust is also feature.

**90-second undo window for everything external.** Tech-skeptical users hesitate before approving anything irreversible. The undo window converts "did I just send the wrong thing?" anxiety into "I have time to fix it." Behaviorally, almost no one uses it. But its existence changes how confidently they approve.

**Prompt injection as a first-class concern.** All inbound external content — emails, web search results, scraped data — is treated as untrusted data. The reply classifier uses a strict extraction schema with no tool access. Inbound content can be summarized, classified, and quoted, but never followed as instructions.

**Designed for the user I had, not the user I imagined.** The hardest discipline in product design is staying focused on a specific real person when you could be designing for "the market." Her specific constraints — tech-skepticism, Apple aesthetic preference, conversation-first, hard relationships with specific reps — were the design constraints. The system would be different and worse if I'd designed for "senior bookers in general."

---

## Status

**As of May 24, 2026:**

- Specification: complete (v3, post-Managed-Agents pivot)
- Architecture review: three passes complete
- Build: in progress this week
- Demo to the user: imminent
- Adoption: unproven until she has it in her hands

**Honest uncertainty:** I might be wrong about her willingness to chat with an AI in plain English. I might be wrong about which sub-agents are most useful. I might be wrong about the 90-second undo window. The demo is where the design meets reality. I'll know in days, not months.

---

## What's in this repo

```
booking-agent/
├── README.md              ← you are here
├── LICENSE                ← MIT (kit portability)
├── .gitignore
├── docs/
│   ├── spec.md            ← redacted system specification
│   ├── architecture.md    ← the Managed Agents pivot story
│   ├── methodology.md     ← how I built this (the meta-story)
│   └── learnings.md       ← running journal as the build progresses
├── agents/                ← agent definitions (in progress)
├── app/                   ← application shell (in progress)
└── tests/                 ← simulation harness (in progress)
```

The code directories are intentionally bare at the start. Commits will land over the coming days as the build progresses. By the time you read this, there will probably be more in them.

---

## About me

Nnamdi Ejiogu. Enterprise sales professional with 15+ years across SaaS, cloud, and AI. Currently at Google Workspace selling AI productivity tools to enterprise customers. Previous roles at Dynatrace (observability), DFIN (financial SaaS), Redgate Software (developer/DBA tools), and J.P. Morgan Private Bank. Founder of tammah on the side. I don't write production code, but I've spent my career close enough to engineering to direct technical work — and AI tools have made it possible to do that at a level that previously required a team.

**Contact:** nnamdi.ejiogu@gmail.com · [linkedin.com/in/nnamdi-ejiogu](https://www.linkedin.com/in/nnamdi-ejiogu/)

---

## What I'm interested in

I'm applying for roles where the work involves selling AI-forward developer tools to engineering audiences. The intersection of technical fluency, product judgment, and customer empathy is where I've been most effective in my career — and AI-native development tools are reshaping what that intersection produces. This project is one example of why I'm convinced that shift is real.

If you're considering me for a role, this repo is meant to be a window into how I think and work. The specification documents in `docs/` will tell you more about my approach than my résumé will. Read the methodology doc first if you only have time for one.
