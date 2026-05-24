# Methodology

How this project was actually built. Of the docs in this repository, this one is most worth reading if you're thinking about whether to hire me or work with me.

---

## The hypothesis I was testing

The conventional framing of AI coding tools is that they make good engineers faster. That's true but undersells what's happening. The thesis I wanted to test on this project was broader:

**AI tools dramatically reduce the friction between knowing what to build and shipping it — and that reduction matters most for people who have product judgment but aren't working engineers.**

I'm one of those people. I've written HTML, CSS, JavaScript, a fair amount of SQL, and can read Python. I sold to developers and DBAs at Redgate Software for years, which means I learned to evaluate technical decisions through thousands of conversations with people who do them for a living. I'm fluent in the language and the work; I just chose sales over coding because I prefer working with people.

A year ago, this project would have required hiring an engineer or a small team. Today, with the right approach, one person can ship it. This document is about what that approach actually looks like in practice.

---

## The discipline

The single most important thing I did was spec before code. I wrote roughly 1,800 lines of specification across three review passes before opening Claude Code. This felt wrong at every step — the AI is sitting there ready to write code, and I'm writing markdown. It was the right call.

The reason: AI coding tools execute fast. They write code at a speed where the bottleneck is no longer typing — it's deciding what to type. If you don't have a clear specification, the AI will happily produce code, and the code will look right, and you'll spend the rest of the project unwinding decisions you didn't realize you were making.

The specs I wrote weren't just "what to build." They included:

- Data models with field definitions
- State machine with allowed transitions
- Failure modes and how to handle each
- Memory permission matrix
- Idempotency keys for every external action
- Audit event taxonomy
- The exact words of every system prompt

When I finally opened Claude Code, the build was execution, not design. I knew what I wanted. I could tell when the AI produced something off-spec immediately, because I had the spec in front of me.

---

## Adversarial review

I ran two adversarial review passes on the spec using a different model (GPT-5.5 Thinking). The prompt: "Be a senior engineer reviewing this for production readiness. Be harsh. If something is broken, tell me what and why."

The first pass caught real things I missed:

- **Identity resolution.** I had `guest_id` everywhere but no system for assigning canonical IDs or handling fuzzy matching. The system would have generated duplicate outreach within a month.
- **Prompt injection.** I had no boundary around inbound external content. A rep could have embedded "ignore previous instructions" in an email and the agent would have processed it.
- **State concurrency.** I had a state machine but no compare-and-swap semantics. Two simultaneous updates would have produced inconsistent state.
- **Deploy safety.** I had no plan for what happens when the system redeploys mid-undo-window.

The second pass after my revisions caught a few more. The total: about 18 architectural improvements that would have been expensive to find mid-build.

This works because different models have different blind spots. The first model (Claude) helped me design with full context of my product. The second model (GPT-5.5) had no investment in my decisions and would say "that's wrong" without softening. I filtered the feedback against product reality, accepted what mattered, rejected what didn't. The whole pattern is what software teams call code review, applied to specs and done with AI.

I'll use this pattern on every project from now on.

---

## Pivot under uncertainty

Mid-spec, Anthropic released Claude Managed Agents — a hosted runtime for the exact kind of multi-agent orchestration I was building from primitives. Roughly 60–70% of my custom infrastructure became obsolete overnight.

The temptation was to keep going. The work was done. The spec was good. The plan was clear. The new platform was just an "alternative."

I killed the v2 build before starting it.

What this required:

1. **Honest evaluation.** Does v2 still serve the product requirements given today's options? No — Managed Agents better serves the portability and self-service requirements that emerged as I understood the customer better.

2. **Sunk cost containment.** Three weeks of design work. Two adversarial reviews. A coherent spec. None of that mattered to the question "what's right now?"

3. **Surgical reuse.** Most of the product thinking transferred. System prompts, scoring model, HITL checkpoints, voice approach, intervention commands, design system. What changed was the chassis. I kept the engine.

4. **Speed.** Pivoting on a spec costs days. Pivoting on a half-built system costs weeks. The spec-first discipline made the pivot survivable.

This is the single anecdote I'd most want a technical buyer to hear about how I work. Most engineers I've sold to over the years are good at execution. The harder skill — and the rarer one — is knowing when to throw out work that's wrong even when it took a long time to produce.

---

## Working with AI as a discipline

There's craft to working with AI tools, and it's distinct from "using AI." Things I learned across this project:

**Push back on the model.** AI is fast and confident. Confidence is a signal to slow down, not speed up. When Claude produced an answer that felt off, I asked "what are you not telling me?" or "what would someone who disagreed with this say?" Often the better answer came on the second pass.

**Use different models for different jobs.** Claude for design with full context. GPT for adversarial review where I want a stranger's eye. Don't let the same model be both architect and reviewer — it has the same blind spots in both roles.

**Specify the protocol, not just the task.** When I asked for code review, I specified: be harsh, no generic feedback, if something is broken tell me exactly what. Without protocol specification, AI defaults to agreeable, surface-level analysis.

**Build a TRUST scale.** I asked Claude to label its responses by stakes — L1 factual, L2 technical, L3 strategic, L4 creative quality, L5 identity work. For L3 and up, I required steelmanning and explicit failure modes. This forced me to engage at the right depth for the right decisions and not rubber-stamp consequential calls.

**Direct work; don't approve work.** When AI produces something, I evaluate against the spec, the product context, and my own judgment. I don't ask "is this good?" — I ask "is this what I asked for?" Different question, different muscle.

**Treat the AI as a fast, confident, very capable junior.** This isn't dismissive. It's accurate. Manage accordingly: give clear specs, review carefully, require self-checks, expect iteration.

---

## What I'd tell someone trying to do this themselves

The five things that mattered most, in order of leverage:

1. **Spec rigorously before any code is written.** This is the multiplier on everything else. Without it, AI tools amplify confusion. With it, they amplify clarity.

2. **Use adversarial review.** Pay $20 for a month of a different model's premium tier. Run your spec through it. The feedback you'll get for $20 is worth a week of rework you would have done instead.

3. **Build the spine before the muscles.** Whatever your project is, identify the safety-critical, hard-to-change core. Build that first. Test it. Get it stable. Then add features on top. Resist building features first because they're more visible.

4. **Pivot when the ground moves.** The platforms are evolving fast. Capabilities ship that didn't exist when you started. Notice. Evaluate. Pivot if the answer says to. Sunk cost is a tax on judgment.

5. **Document the methodology, not just the product.** This repo isn't just code. It's a record of how the project was built. If the build works, the methodology is more valuable to other people than the artifact. If the build fails, the methodology is what I take to the next attempt.

---

## What this isn't

Some things this is *not*, in case the methodology read as evangelism:

It is not a claim that "anyone can build software now." The product judgment, technical literacy, customer empathy, and discipline required are non-trivial. AI tools make these qualities go further than before. They don't substitute for them.

It is not a claim that AI replaces engineers. Large complex systems still need traditional engineering coordination. Multi-team coordination, performance optimization, security audits, infrastructure operations — all still hard, all still require deep specialists. What changes is the floor of what's possible solo, for projects of bounded scope.

It is not a claim that I'm an engineer now. I directed this build, made the architectural decisions, ran the reviews. The implementation is the AI's. The judgment is mine. Both matter; neither alone would have shipped this.

It is a claim that the methodology — specify, review, pivot, build with AI, document — works. The proof is the artifact this README sits in front of. Read the rest of the docs and decide for yourself.
