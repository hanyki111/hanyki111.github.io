---
title: "Your AI Coding Agent Doesn't Know What It Doesn't Know"
date: 2026-04-10
summary: "The biggest unsolved challenge in AI-assisted software engineering isn't making agents smarter. It's making them aware of their own ignorance."
tags: ["ai-agents", "multi-agent", "epistemology", "coding-agents", "llm"]
ShowToc: true
TocOpen: true
---
The biggest unsolved challenge in AI-assisted software engineering isn't making agents smarter. It's making them aware of their own ignorance.

## The delegation problem

When you hand off a task to a colleague, three things are implicitly true. They know what they can and can't do. They ask when they're unsure. And if they mess up, you can undo it.

Current AI coding agents violate all three.

**They don't know what they don't know.** A NeurIPS 2024 paper ("Large Language Models Must Be Taught to Know What They Don't Know") showed that prompting alone doesn't produce reliable uncertainty estimates. You can tell the model "say I don't know when you're unsure" all you want — the model's confidence calibration doesn't improve from instructions. It takes fine-tuning on about 1,000 graded examples to get generalizable uncertainty estimation. That "be honest about uncertainty" system prompt? It's basically a placebo.

**They don't ask.** Every coding agent today handles uncertainty reactively: run, observe failure, retry. No production system as of 2026 proactively detects "I'm not sure about this" *before* acting. They all learn from crashes, not from caution.

**Undo is shallow.** Aider auto-commits each change. Cursor has editor undo. Claude Code shows diffs before applying. But none of these let you atomically roll back "the last two hours of work that was built on a wrong assumption." You can undo the last step. You can't undo a chain of reasoning.

Here's the paradox: the more autonomy you give an agent, the more its self-awareness matters — but LLM self-awareness doesn't scale with autonomy. In a chat-based tool, you review every response. In a long-running autonomous agent making 50 consecutive decisions, a bad judgment at step 5 silently corrupts steps 6 through 50.

The industry calls this "error compounding." One mistake raises the probability of the next. In multi-agent systems, it gets worse — one agent's hallucination becomes another agent's trusted input. The AutoGen team and CrewAI community have consistently reported cascading hallucination as the single worst practical problem.

## A taxonomy of not-knowing

The Rumsfeld matrix — known knowns, known unknowns, unknown unknowns — gets a bad rap for its source, but it's surprisingly useful for organizing what goes wrong with AI agents.

**Known knowns** — the agent knows JavaScript syntax, standard library APIs, common design patterns. Fine. Agents do well here.

**Known unknowns** — the agent knows it needs to look something up. "What's the database schema?" It recognizes the gap and reaches for a tool. Current systems handle this partially. Claude Code's tool-use architecture forces the agent to read files rather than hallucinating their contents. Cursor's @codebase lets agents search. This is where most engineering effort has gone.

**Unknown unknowns** — the agent doesn't know what it doesn't know. This is the dangerous quadrant. The agent writes auth code without knowing about the project's custom trust-proxy middleware. The agent implements a rate limiter without knowing one already exists three directories over. The context *looks* complete enough, so the agent never thinks to search further.

**Unknown knowns** — the model has relevant knowledge from training but can't access it in a specific context. Ask the same question differently, get a different answer. One agent misses what another agent (different model, different prompt) would catch. This is actually one of the few legitimate arguments for multi-agent systems.

The key insight: **every coding agent today focuses almost entirely on known unknowns.** Run the code, see it fail, fix it. Use tools to look things up. But unknown unknowns — the most dangerous category — have no structural defense in any production system.

## The uncomfortable truth about multi-agent systems

Let's talk about whether multiple agents actually help.

### The evidence is... not great

**Single agents match or beat multi-agent on 64% of tasks** when given equal compute and tools (Princeton NLP group). An April 2026 paper (arXiv:2604.02460) tested across three model families and five multi-agent architectures: single-agent systems were "the strongest default architecture for multi-hop reasoning" when you normalize for compute.

**SWE-bench tells an interesting story.** On SWE-bench Pro, scaffolding produces 22+ point swings with the *same model*, while switching frontier models accounts for roughly 1 point. Claude Sonnet 4.5 (the smaller model) scored 52.7%, beating Claude Opus 4.5 at 52.0% on Anthropic's own scaffold. A weaker model with better scaffolding outperformed the flagship. The implication: how you organize the agent matters far more than which model you use.

**The cost multiplier is real.** Agentic systems consume 5-9x more tokens per workflow. A task costing $0.10 with a single agent costs $1.50 with multi-agent — a 10-15x multiplier. One mid-size e-commerce firm saw monthly infrastructure costs jump from $5,000 to $50,000 going from prototype to staging. Gartner warns that 40% of agentic AI projects will fail by 2027 due to cost and complexity.

**Most "multi-agent" systems aren't really multi-agent.** Yoav Goldberg argues convincingly that true multi-agency requires private state not observable by other agents and state persistence across invocations. Different prompts, different tools, parallelism — that's not multi-agent, it's multi-role. Most frameworks marketed as "multi-agent" are really modular single-agent pipelines.

### When multiple agents actually help

Despite all this, there are specific conditions where the evidence *does* support multiple agents.

**Context isolation.** Anthropic's context engineering guide explicitly recommends sub-agent architectures — specialized agents working in clean context windows, reporting condensed summaries. JetBrains Research confirmed "context rot" — LLM accuracy degrades as context grows. Separate contexts aren't about having more agents. They're about having cleaner attention.

**Structural diversity.** Giving the same LLM different prompts creates artificial diversity. Having different models (Claude, GPT, Gemini) look at the same problem creates structural diversity. This helps with unknown knowns — knowledge one model can't access might surface in another.

**Cognitive stage separation.** Planning, executing, and verifying in the same context creates self-confirmation bias — you're less likely to find bugs in code you just wrote. Separating these into different contexts helps. But note: the value comes from context isolation, not from the number of agents.

### The bitter lesson, playing out in real time

Here's the part that should concern anyone building agent scaffolding.

Browser Use, a browser automation agent team, threw away their entire 10,000-line codebase and started from scratch. Their conclusion: "All the value is in the RL'd model, not your 10,000 lines of abstractions."

Anthropic's own harness evolved through three stages: a 2-agent system for Sonnet 4.5, expanded to 3 agents for Opus 4.5, then *simplified back* for Opus 4.6 — removing sprint decomposition, multi-pass evaluation, and context resets. Their takeaway: "Every harness component encodes an assumption about what the model can't do alone. When models improve, those assumptions must be re-tested. Strip away what's no longer needed."

Lance Martin at LangChain initially avoided tool calling, used rigid report section decomposition, and forced parallel writing. All three became unnecessary as models improved.

The pattern: **scaffolding is a bundle of assumptions about model limitations.** Models improve quarterly. Assumptions expire. Scaffolding becomes dead weight.

This applies to any agent system. Every verification layer, every assumption gate, every coordination mechanism encodes a belief about what the model can't do on its own. Some of those beliefs will be wrong by next quarter.

## The missing piece: assumption tracking

Here's something surprising: **nobody tracks assumptions.**

When an agent starts working on your project, it makes implicit assumptions. "This project uses PostgreSQL." "Auth is JWT-based." "This API is RESTful." These assumptions come from context clues or from training data patterns. They're never written down.

When an assumption is wrong, you find out at the worst possible time — when the SQL query fails against MongoDB, when the JWT middleware conflicts with the existing OAuth flow. And by then, 10 decisions have been stacked on top of that bad assumption.

This matters for three reasons:

**You can't verify what you don't record.** If the agent had written down "I'm assuming PostgreSQL," someone (or something) could check that assumption before 10 hours of work depend on it.

**You can't trace blast radius.** When an assumption breaks, which downstream decisions are affected? Without explicit tracking, the only option is manually reviewing everything.

**You can't measure progress.** Is the agent's exploration converging on a solution, or spinning its wheels? Without tracking which uncertainties have been resolved and which are new, you can't tell.

There is prior art. In 40 years of AI research, Assumption-based Truth Maintenance Systems (ATMS, de Kleer 1986) did exactly this — tracked assumptions and propagated their consequences. This idea has never been applied to LLM agents. Not because it's a bad idea. Because LLM agents are new, and nobody's connected the dots yet.

## Token budgets don't detect wasted work

Every system has spend limits. Claude Code has `--max-turns`. AutoGen has `max_consecutive_auto_reply`. CrewAI has `max_iter`. You can set dollar caps.

These prevent catastrophic runaway — the agent won't spend $500 in an infinite loop. But they don't detect *semantic* divergence: the agent spending 45 turns on a dead-end approach within its $5 budget, because a bad assumption at turn 5 sent it down the wrong path.

A LangGraph user reported 11 unintended revision cycles costing $4. The cost wasn't the problem — the system's inability to detect "you're going in circles" was.

What you actually want is a signal that measures *progress*, not just *effort*. If the agent is resolving uncertainties — answering open questions, confirming or disproving assumptions — it's converging. If uncertainties keep piling up while none get resolved, it's diverging.

This is where assumption tracking connects to divergence detection. If you track assumptions explicitly, you can compute an "assumption delta" per work unit: resolved assumptions minus newly introduced ones. A persistently negative delta over the last K units means the agent is getting *more* uncertain as it goes — the definition of unproductive exploration.

This signal is unvalidated. It might turn out that agents cherry-pick easy assumptions first, showing false convergence while the hard ones remain untouched. Or the delta might not correlate with actual progress at all. But conceptually, *something* needs to measure content-level progress, and assumption resolution rate is a reasonable candidate.

## Detecting unknown unknowns without asking the agent

If prompting can't reliably make agents admit uncertainty, what can?

**Deterministic symbol verification.** When an agent generates code that calls `getUserById()`, check whether that function appeared in the agent's tool call history — the files it actually read, the searches it actually ran. If the agent never verified the function exists, you know deterministically that it assumed rather than checked. This costs zero LLM calls, produces near-zero false positives, and catches a real category of hallucination.

The limitation: this catches identifier-level hallucination but misses design-level wrong assumptions ("this system uses microservices"). That's where a second layer helps.

**Cross-agent review.** Have a different agent, in a different context, review the output. What's invisible in one context might be obvious in another. This is an automated approximation of "have a colleague look at it." It costs additional LLM calls, so it should only fire when the deterministic layer can't catch the problem.

**Layered verification.** Deterministic checks (free, narrow) first. Cross-agent review (medium cost, wider) second. Active inquiry by the agent itself (expensive, widest) third. Exhaust cheap methods before reaching for expensive ones.

The key principle: **detection of unknown unknowns should not rely on the agent that has them.** The whole problem is that the agent doesn't know what it doesn't know. An external mechanism — deterministic, cross-agent, or structural — must do the detecting.

## Where this leaves us

Four directions seem defensible based on current evidence:

1. **Don't trust the agent's self-report.** Build detection mechanisms outside the agent. Deterministic verification, cross-agent review, structural checks. The evidence that LLMs can't reliably self-assess uncertainty is solid.

2. **Track assumptions explicitly.** This is the biggest gap in the field right now. No coding agent system does it. The idea is 40 years old (ATMS). Error compounding, divergence detection, blast radius analysis — they all need explicit assumption tracking as infrastructure.

3. **Context isolation over agent count.** "More agents = better results" is not supported by evidence. But isolated contexts that prevent attention dilution and self-confirmation bias *are* supported. The value is in the isolation, not the multiplication.

4. **Measure progress, not just effort.** Token budgets prevent catastrophe but miss waste. Something needs to measure whether the agent is actually getting closer to a solution. Assumption resolution rate is a reasonable candidate, but it's unproven.

These problems belong to everyone building long-running autonomous coding agents. None of them have clean answers yet. But pretending they don't exist — shipping agents that confidently hallucinate and hoping users will catch every mistake — isn't a viable path either.

---

*If you're working on similar problems — especially assumption tracking or convergence detection in agent systems — I'd love to compare notes.*

---

### References

- Banda et al. "Large Language Models Must Be Taught to Know What They Don't Know." NeurIPS 2024.
- arXiv:2604.02460. "Single-Agent LLMs Outperform Multi-Agent Systems on Multi-Hop Reasoning Under Equal Thinking Token Budgets." April 2026.
- Anthropic. "Effective Context Engineering for AI Agents." Engineering Blog, 2025.
- JetBrains Research. "Cutting Through the Noise: Smarter Context Management." December 2025.
- Galileo. "Why Do Multi-Agent LLM Systems Fail." 2025.
- Browser Use. "The Bitter Lesson of Agent Frameworks." 2026.
- Anthropic. Harness Design Philosophy & Evolution. 2025-2026.
- Goldberg, Yoav. "What makes multi-agent LLM systems multi-agent?" GitHub Gist, 2025.
- de Kleer. "An Assumption-based TMS." Artificial Intelligence 28(2), 1986.
- Shinn et al. "Reflexion: Language Agents with Verbal Reinforcement Learning." NeurIPS 2023.
- Zhou et al. "Language Agent Tree Search." ICML 2024.
- Microsoft. "Taxonomy of Failure Modes in Agentic AI Systems." April 2025.
