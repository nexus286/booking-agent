# Learnings

A running journal of what I'm discovering as the build progresses. Updated as I go.

---

## May 24, 2026 — Pre-build

**On the value of spec-first discipline.** Three weeks of specification before opening Claude Code felt slow. I had to fight the instinct to just start building, especially because AI tools make starting so easy. By the end of the spec phase I had 1,800 lines of design across the original spec, two adversarial review responses, and one major architectural pivot. None of that work felt like progress in the moment. All of it is going to make the build itself fast and the result correct.

**On adversarial review.** I'd never tried using a different model to review my own work. The first GPT-5.5 review came back with eight architectural gaps I'd missed — identity resolution, state concurrency, prompt injection, deploy safety, others. About half the feedback was generic senior-engineer skepticism that didn't apply to my single-user demo system. The other half was sharp. The skill turned out to be filtering — knowing which feedback to accept and which to reject with reasoning. This is a process I'll repeat on every project.

**On the Managed Agents pivot.** I want to remember exactly how it felt to throw out v2. The work was good. The spec was coherent. I'd done two rounds of review. The momentum was there. Killing it felt wrong every minute, until I sat with the actual question: "given today's options, what's the right architecture?" The answer was unambiguous. The feeling of wrong took about four hours to fade. The decision was correct in real time and is more obviously correct in retrospect.

**On the customer relationship.** Building for someone you actually know is different from building for "users." Every architectural decision passes through a specific filter: would she tolerate this? Would she understand it? Would she abandon it? That filter is the most valuable design tool I have. It's also what I'll lose when this product scales beyond her — and that's a real challenge I haven't solved yet.

**On Cursor specifically.** This whole project is a case study in why AI-forward development matters for people like me. Not engineers, but technical-adjacent. Years of close work with engineers. Strong product judgment. Discipline. A year ago I couldn't have built this. Today I'm a week away from demoing it. The platform shift is real, and the people best positioned to take advantage of it aren't always the working engineers — sometimes they're the people who were close enough to engineering to know what they wanted but never built it themselves.

---

*More entries will land here as the build progresses. The point of this document isn't to be polished — it's to capture what I'm actually thinking in close to real time. Some of what's here will end up wrong. That's the value of writing it down.*
