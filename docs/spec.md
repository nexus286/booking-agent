# Technical Specification (Redacted)

System architecture and design decisions for the booking agent. This document is a redacted version of the full internal specification — specific names, dates, agencies, and identifying details have been removed for privacy. The technical substance is unchanged.

For the story of how this architecture evolved (including the mid-spec pivot to Claude Managed Agents), see `architecture.md`.

---

## Product overview

A bespoke AI booking assistant for a senior talent booker at a major national sports show. The user talks to a single super-agent in plain English. The super-agent coordinates six specialized sub-agents on her behalf:

| Sub-agent | Responsibility |
|---|---|
| Scout | Monitors sports news 24/7, scores guest candidates 0–100 |
| Outreach | Contact enrichment, pitch drafting, 3-touch follow-up sequences |
| Scheduler | Calendar negotiation, confirmations, automated reminders |
| Logistics | Travel, hotel, ground transport, green room, day-of monitoring |
| Research | Pre-show host briefs, segment questions, no-go zones, producer notes |
| CRM | Relationship tracking, rebooking scores, weekly pipeline reports |

The user experience is conversational — she types instructions and approvals into a chat interface. The dashboard exists for transparency (seeing what's in progress) but never for configuration.

---

## Architecture

Built on Claude Managed Agents (Anthropic's hosted multi-agent runtime). The platform provides:

- Multi-agent orchestration with parallel sub-agent execution
- Stateful sessions that survive interruptions and resume on demand
- Built-in memory with cross-session curation (Dreaming)
- Sandbox execution for tool calls
- MCP connectors for external integrations
- Observability via Claude Console

The application shell is a thin Python layer (~1,000 lines) that:

- Defines the agents and their system prompts
- Handles webhooks (inbound emails via Gmail Pub/Sub push, SMS replies via Twilio)
- Manages the chat interface
- Implements the 90-second undo window for external actions
- Surfaces human-in-the-loop approvals as in-chat prompts

**Primary model:** Claude Opus 4.7 for coordinator reasoning and complex tasks.
**Routing model:** Claude Haiku 4.5 for fast routing and low-stakes calls.

---

## Guest scoring model

The Scout sub-agent scores candidates on five dimensions:

| Factor | Weight |
|---|---|
| Relevance to show content themes | 30 pts |
| Timeliness of news hook (last 7 days) | 25 pts |
| Audience pull (social following + search trend) | 20 pts |
| Accessibility (known contact, friendly media history) | 15 pts |
| Risk deduction (controversy, legal issues, media ban) | -10 pts max |

Output: ranked JSON of top 10 candidates weekly, with score, news hook, segment suggestion, risk flags, and priority level.

---

## Outreach sequence

| Touch | Day | Behavior |
|---|---|---|
| 1 | 0 | Personalized pitch — HITL review before send |
| 2 | 5 | Follow-up nudge — semi-auto |
| 3 | 10 | Final ask — auto-send if no response |
| Archive | 11+ | Move to 90-day retry flag |
| Reply detected | any | Sequence pauses; reply classifier runs; appropriate routing |

---

## Human-in-the-loop checkpoints

Seven defined moments where the system pauses for the user's explicit approval. Surfaced as in-chat prompts; she taps approve or reject.

1. **Weekly Scout shortlist approval** — system delivers ranked 10, she approves who to pursue
2. **Pitch email review** — every outbound email shown before send
3. **Positive reply routing** — agent surfaces reply + recommended response
4. **Booking confirmation** — date, terms, format confirmed
5. **Out-of-policy logistics** — escalates if travel exceeds standard
6. **Risk/controversy flag** — pauses all outreach, sends risk brief immediately
7. **Research brief sign-off** — delivered 48 hours pre-taping for producer review

### Escalation policy

Three tiers based on action consequence:

- **STANDARD** — most approvals; dashboard badge, 24hr → soft SMS, 48hr → daily digest
- **PRIORITY** — time-sensitive but not urgent; 8hr → direct SMS, 24hr → firm digest
- **URGENT** — real-world consequence imminent; immediate SMS + repeat at 1hr + SMS+call at 3hr

**Quiet hours suppression:** 10pm–7am local time. Only URGENT breaks through.

**Weekend behavior:** STANDARD downgrades to badge-only (no SMS until Monday). PRIORITY and URGENT unchanged.

---

## Undo primitive

Every external action — email send, calendar invite, contract send, SMS to external — gets a 90-second soft-delete window before firing.

```
User approves action
       ↓
Action queued. fire_at: now + 90 seconds
Dashboard shows live countdown with [Undo] button
SMS to user: "Sending in 90s — reply STOP to cancel"
       ↓
       ├─→ User cancels within window (UI tap or SMS STOP)
       │       ↓
       │   Status: CANCELED. Action discarded. Audit logged.
       │
       └─→ 90 seconds elapse with no cancel
               ↓
           Worker fires API call with idempotency key
           Status: EXECUTED. Audit logged.
```

**Configuration is centralized:** `UNDO_WINDOW_SECONDS = 90` is the single source of truth. Change one value, the whole system reads the new window — dashboard countdown, SMS template, worker timing.

**Internal-only actions fire immediately.** Memory writes, state changes, log entries don't need the window. The rule is: if it's visible to anyone other than the user, it gets the window.

**Demo mode flag:** When set, all external actions are intercepted and logged but never executed. Allows risk-free demos and onboarding.

---

## Reply classifier

Inbound replies aren't simple "yes" or "no." Real email behavior includes out-of-office responses, replies from new contacts, ambiguous prose, forwarded threads, fee asks with conditions, and combinations of all of these.

The reply classifier categorizes inbound emails into 11 types:

```
human_positive          — clean yes
human_conditional       — yes with conditions
human_negative          — clean no
human_future_interest   — no now, open later
out_of_office           — autoresponder
bounced                 — undeliverable
forwarded_to_new_contact — thread expanded
assistant_added         — gatekeeper inserted
ambiguous               — unclear intent
unrelated               — off-topic
risk_signal             — controversy mentioned
```

Each type routes differently. Out-of-office doesn't permanently pause the sequence. Bounces trigger contact re-verification rather than user escalation. New contacts added to a thread become candidates rather than trusted reps until approved.

**Critical security boundary:** The classifier uses a strict JSON schema, not freeform LLM reasoning. It has no tool access. Inbound email content is *data*, not *instructions*. This is the prompt injection boundary — see below.

---

## Prompt injection boundary

All inbound external content (emails, web search results, scraped data) is treated as untrusted.

**Untrusted content may be:**
- Summarized
- Classified
- Quoted in audit logs
- Extracted into structured data

**Untrusted content must never:**
- Modify system instructions
- Override approval rules
- Access memory directly
- Trigger tools directly
- Be followed as instructions to the LLM

The classifier and risk analyzer use strict extraction schemas (typed JSON output) with no tool access. The strict-schema pattern eliminates the attack surface where an attacker could embed "ignore previous instructions" in an email and have it processed.

---

## Memory model

Three categories of stored knowledge, each with different trust semantics:

| Category | Trust | Used by agents |
|---|---|---|
| Verified facts | High | Used automatically |
| User preferences (explicitly stated) | High | Applied automatically |
| External claims (from email/web) | Low | Surfaced for verification, not auto-applied |

This prevents memory poisoning. If a rep email says "the user always waives fees for our clients," that becomes an *external claim* tagged with its source — not a *preference* the system silently honors.

---

## Identity resolution

Before any outreach action, the system resolves the candidate's identity:

```python
def resolve_guest_identity(name, context) -> (canonical_identity, confidence):
    # confidence < 0.85: block action, surface to user for manual confirmation
    # confidence 0.85–0.95: proceed with warning logged
    # confidence > 0.95: proceed normally
```

Canonical identities track aliases, external IDs, and merge history. This prevents the common failure mode where "Marcus Holloway" and "M. Holloway" become two separate records receiving duplicate outreach.

---

## Voice profile

Outreach emails must sound like the user, not like an AI. The system ships with a placeholder voice ("warm but efficient, first name basis, never sycophantic, avoid common email clichés"). After demo, the user provides 20 real sample pitches; the system extracts her actual voice profile and stores it as a user preference.

The extraction is a one-time prompt:

1. Sentence structure patterns
2. Vocabulary preferences and aversions
3. Opener patterns
4. Sign-off patterns
5. How she expresses opinions and asks
6. Habitual phrasings
7. Things she never does

The Outreach sub-agent reads this profile before drafting any pitch. If the profile is missing, drafts are flagged with `[REVIEW: voice profile not yet established]`.

---

## Intervention commands

Available in the chat interface and via SMS:

```
/pause [agent]            Halt agent at current step
/resume [agent]           Resume paused agent
/blacklist [name]         Freeze all activity on a guest, log reason
/status                   Full pipeline summary
/brief [guest]            Generate research brief on demand
/manual_update [guest]    Update guest state manually
/mark_replied [guest]     Mark thread as user-handled directly
/mark_confirmed [guest]   Mark booking confirmed outside the system
/stop_followups [guest]   Stop automated follow-up sequence
/log_call [guest]         Log a phone call the user made
/takeover [guest]         Enter human takeover mode for this guest
/release [guest]          Exit takeover mode, resume automation
```

The takeover commands (the last six) are essential. The system must coexist with manual work the user does outside it — phone calls, in-person conversations, direct emails sent from her phone. When she takes over a thread directly, automation pauses for that guest until she releases.

---

## Deployment

Hosted application shell on Railway. PWA web interface (added to her phone home screen, feels native). One-click deploy from this repository — environment variables, Anthropic API connection, MCP connector setup all happen in the Railway dashboard.

**Setup time from zero to working assistant:** approximately 80 minutes, broken down as:

| Step | Time |
|---|---|
| Create Anthropic account, add payment method | 5 min |
| Click "Deploy to Railway" from repo | 3 min |
| Add API keys to environment variables | 10 min |
| Connect MCP connectors (Gmail, Calendar, Airtable) | 15 min |
| Bootstrap session (forward emails, set rules, voice samples) | 45 min |

The bootstrap session is conversational, not a configuration wizard. The user forwards 10 emails from top contacts; the system extracts contacts. She forwards 20 sent pitches; the system learns her voice. She mentions her blackout dates; they're stored as rules.

---

## Ownership model

The user owns:

- Her Anthropic account (billed directly)
- Her Railway deployment (billed directly)
- A fork of this repository (she or anyone can modify with Claude Code)

The original developer's involvement after setup: zero required. If the user wants help, she pays for time. The system is designed to make this practical — the application shell is small enough that anyone with basic technical literacy and Claude Code can maintain it.

---

## Open items deferred to v1.1

These are config-level changes, not architectural changes. The system ships without them; they fill in during onboarding.

- **Voice profile** — Placeholder voice for v1. Real voice profile extracted after user provides sample pitches.
- **Specific agency flag list** — User provides during onboarding.
- **Blackout date list** — User provides during onboarding.
- **Show calendar themes** — Populated weekly by user.
- **Per-rep automation tuning** — Adjust trust levels based on actual usage data.

---

## What's not in this document

The full internal specification includes additional sections covering deployment safety, schema migration policy, audit event taxonomy, retry policy across error tiers, cost guardrails, security details (Gmail OAuth scope, PII encryption at rest, Twilio webhook signature verification), and the full simulation harness with adversarial test scenarios.

Those sections have been omitted from this redacted version either because they contain implementation details that don't add value for a reader at this stage, or because they reference specific names, dates, and identities that I don't want public.

If the technical depth here is interesting to you and you want to discuss specifics, I'm happy to walk through any of it in a conversation.
