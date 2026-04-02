# Distilling Claude Code Source — A Harness Engineering Practice Log

[中文版](distillation-harness-practice-zh.md) `READ⏰: 25min`

It occurred to me that the process of distilling Claude Code source into an Agent Harness Patterns Skill might make for a decent harness engineering read — possibly more valuable than the skill itself. So here's a write-up, also as supplementary material for a future proper Harness Engineering blog post.

<img src="../images/agent-harness-patterns-header.png" alt="agent-harness-patterns" style="zoom: 25%;" />

A few hours after the Claude Code source leak, I started distilling. The motivation was simple: this is probably the most mature production agent harness implementation publicly available, and I was curious about its context engineering internals.

But 512,000 lines of TypeScript is clearly more than one agent can handle in a single session. So the distillation process itself became a harness engineering exercise — I needed to design a coordination mechanism for multiple agents to collaborate efficiently, while keeping my own involvement as precise and minimal as possible. My design principles drew heavily from these posts:

- [Anthropic - Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [OpenAI - Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- [OpenAI - How we used Codex to build Sora for Android in 28 days](https://openai.com/index/shipping-sora-for-android-with-codex/)

In retrospect, the process resembles a PCA-style dimensionality reduction. I injected my own blog posts and preferences as a set of "basis vectors," then had agents project from the complex code space to extract the most important principles. **Code is high-dimensional, but valuable design patterns are low-rank — distillation is about finding those principal components.**

This post covers three layers: the harness architecture, where I intervened as a human, and reflections from the process.

## 1. Harness Design: Split by Role, Coordinate via Filesystem

The core idea is separating review from execution across two different model families, with every review-action cycle happening in a fresh session. (I chose this design because in my daily experience, Codex GPT 5.4 xhigh doesn't cut corners on review and design tasks.)

| Role | Agent | Responsibilities |
|---|---|---|
| Review & Architecture | Codex (GPT 5.4 xhigh) | Source-of-truth verification, positioning decisions, abstraction-level judgment, UX audit |
| Build & Execute | Claude Code (Opus 4.6 max) | Source exploration, reference drafting, parallel sub-agent orchestration, edit execution |

Why separate them? Same reason you don't let the implementer review their own code. Codex reviews against source with fresh eyes; Claude Code executes with full understanding of prior decisions. Neither agent sees the other's session. (Anthropic makes a similar point in [Building multi-agent systems: When and how to use them](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them): *One multi-agent pattern that consistently works well across domains is the **verification subagent**. This is a dedicated agent whose sole responsibility is testing or validating the main agent's work.*)

How do they coordinate? The filesystem. File names + a simple Read Tool — low cost, no information loss. (This follows the principle from Anthropic's [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system): **Subagent output to a filesystem to minimize the 'game of telephone.'** See also: [One Poem Suffices: Multi-Agent System](https://keli-wen.github.io/One-Poem-Suffices/one-poem-suffices/multi-agent-system/))

Before any agent started working, I had Codex generate the full harness infrastructure. The design goal: a clean agent with no prior conversation context can pick up where the last one left off — knowing what to read, what to do next, where to write results, and how to get reviewed.

![fig1_v3_20260402_215205_0](../images/fig1-harness-filesystem.png)

Files organized into three groups:

**Role Briefs** — telling each agent "who you are and what the rules are":

| File | Purpose |
|---|---|
| `clean-agent-brief.md` | Builder agent onboarding: operating rules, minimum output requirements, when to stop and ask a human |
| `review-agent-brief.md` | Reviewer agent onboarding: review priorities, output template, severity grading criteria |

**Coordination** — shared state between agents:

| File | Purpose |
|---|---|
| `context-map.md` | Source context map: 50+ source files grouped by harness layer, annotating why each matters |
| `pattern-notes.md` | Exploration notes: key findings from early scans, so later agents read rather than re-explore |
| `task-board.md` | Shared task queue: 35 tasks with dependencies and exit criteria |
| `progress-log.md` | Append-only activity log + decision log (13 key decisions recorded) |
| `handoff_v0_*.md` | Action briefs between review and execution phases |
| `codex_review_v0_*.md` | Codex review outputs: severity-graded findings, source citations, recommended fixes |

**Quality Gates** — defining "what counts as done":

| File | Purpose |
|---|---|
| `review-checklist.md` | Blocking issues (factual errors, unsupported claims) vs. quality checks (cross-runtime portability, trigger accuracy) |
| `execution-strategy.md` | Multi-agent execution strategy: who does what, parallelism level, sub-agent brief templates |
| `output-format.md` | Output format spec: file structure, metadata fields, length limits |

This file set embodies a core harness engineering principle: **build coordination mechanisms before starting work.** Without a task-board tracking progress or a review-checklist defining quality standards, multi-agent collaboration has nothing to coordinate around.

Handoff documents are the API between agents. Each handoff contains exactly what the next agent needs to act — typically an index pointing to where relevant context lives. The receiving agent starts a fresh session, reads the latest handoff (e.g., `@xxx_handoff_xx.md`), and executes.

So for every review or execution cycle, **an agent has three categories of context**:

- Agent role
- Agent task handoff (task-specific) (for Codex, this is the diff in progress-log.md)
- Repo filesystem (*We made repository knowledge the system of record... **give Codex a map, not a 1,000-page instruction manual.***)

(This aligns with the progressive disclosure concept from [JIT Context](https://keli-wen.github.io/One-Poem-Suffices/one-poem-suffices/just-in-time-context/) — the handoff acts as a coarse index, giving agents just enough context at the right time.)

The design inspiration comes primarily from [Anthropic - Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents), which **uses two different agents** — each coding agent picks up a feature in a fresh blank session. I took this further by using token usage scaling to run both the reviewer (Codex) and executor (Claude) in new sessions, aiming for better results. (In my experience, with good harness design, this approach works well and can scale to substantial projects.)

One caveat: since this is document generation rather than coding, the impact of harness quality on efficiency is less obvious (text evaluation is harder to measure). But for coding tasks, a well-designed harness can easily keep an agent running productively for 30-60 minutes with high-quality output.

## 2. Human-in-the-Loop: Taste Injection

The workflow was designed to minimize human effort while distilling at scale. But minimum doesn't mean zero. In practice, certain steps required human judgment.

### 2.1 Taste Injection: Blogs as Instructions

> From OpenAI [Harness engineering](https://openai.com/index/harness-engineering/): *Enforcing architecture and taste* — agents work most efficiently in environments with strict boundaries and predictable structure.

This was probably the most interesting part of the entire process. I fed my previous blog posts on Context Engineering as instructions to Claude Code, so it would reference my own analytical framework when extracting patterns.

The agent didn't extract patterns from scratch. It carried my understanding of the select / write / compress / isolate four-axis framework, my preference for "Do the simple thing that works," and my criteria for abstraction levels. What came out isn't an "objective mapping" of the Claude Code source (that's what the first attempts produced) — it's the result of projecting through my preference basis vectors.

Back to the PCA analogy: the blog posts are the basis vectors. They define "**what directions matter**," and the agent projects along those directions. Without these basis vectors, the agent might extract completely different principal components — perhaps more implementation-focused, perhaps more API-oriented — but not necessarily the harness design principles I was after. (The initial extractions were frankly terrible.)

![fig3_v2_20260402_214909_0](../images/fig3-pca-projection.png)

> [!Tip]
>
> This reflects my broader experience with agent-driven tasks, and OpenAI's blog echoes it. Give an agent a nearly infinite action space and it will fail. You need to lay the foundation by hand first (from: [How we used Codex to build Sora for Android in 28 days](https://openai.com/index/shipping-sora-for-android-with-codex/)). This constrains the action space — once you establish the direction, the agent's output gets progressively better (or at least aligned with what you need).

### 2.2 Key Decision Points

While the actual reviews and edits were all agent-driven, I stepped in at several critical moments:

**Positioning choice.** Codex's first review pointed out the skill was answering the wrong question — "What subsystems does Claude Code have?" instead of "What problem is the builder trying to solve?" The repositioning was Codex's suggestion, but choosing to accept it and directing Claude Code to execute accordingly was my call. (Shifting from code explanation to best-practice extraction.)

**Abstraction boundary.** "If you remove 'Claude Code' from this principle, does it still have value?" This criterion was stabilized over multiple review rounds. Initially Codex and Claude Code had different interpretations of "what counts as a portable principle" — I made the rule explicit in the handoff.

**Bringing in the user perspective.** After 5 rounds focused on content accuracy, I realized we were missing a **user-perspective review**. From my blogging experience, I always end up thinking from the reader's perspective — I hardcoded this rule to require both agents to adopt this viewpoint. (In theory, this should be distilled into a skill itself.)

## 3. Process Overview

![fig2_fanout_convergence_20260402_213926_0](../images/fig2-fanout-convergence.png)

### Phase 0: Scaffolding

Before any agent started, I aligned with Codex on project scope and naming, then had it generate the full harness file set from §1. My prompt was essentially:

> "We need a complete guidance (for clean agents to pick up tasks and quickly understand context) and progress tracking (so clean agents know what's been done). Can you generate this harness set?"

Compared to coding tasks, the harness design for document tasks can be much more lightweight.

### Phase 1-2: Exploration & Parallel Drafting

4 parallel Explore Agents each covered different source areas, scanning ~1,900 files to produce a context map. Then 7 parallel sub-agents each took one harness layer and wrote a corresponding deep analysis (the final `references/*.md`).

### Phase 3-4: Verify → Fix → Pivot

Codex's first review found 6 factual errors (memory system not covered, concurrency classification wrong field, permission source count wrong, etc.). Claude Code fixed them in a new session. Then Codex flagged a direction problem — the focus should be extracting design principles, not explaining code.

### Phase 5: Template Standardization — 8 Agents Rewrite in Parallel

After establishing "principles first," 8 sub-agents simultaneously rewrote all documents to a unified template (largely driven by my manual feedback):

```
Problem (universal) → Golden Rules (portable) → When To Use → Tradeoffs → Implementation Patterns (no code) → Gotchas → Claude Code Evidence
```

The final section describes Claude Code's implementation decisions in natural language — no source paths, function names, or code snippets.

### Phase 6: Review Convergence

(P1 = must-fix-before-shipping errors. P3 = wording-level polish.)

| Review Round | Findings | Key Issue |
|---|---|---|
| v0.4 | 1P1 + 6P2 + 1P3 | Memory model still described incorrectly |
| v0.5 | 3P3 (pass) | Only wording-level adjustments |
| UX audit | 1P1 + 2P2 + 2P3 | Skill runtime's listing has a 250-char hard limit — all carefully tuned trigger words were being cut off |

Severity was decreasing, indicating convergence. But the UX audit surfaced a new P1 — content had converged, but the presentation layer still had a critical problem nobody had noticed.

### Phase 7: Final Optimization

Shortened the skill description (248 → 116 characters) so trigger words reappear in the listing. Added "Start here" guidance to each chapter. Declared the target audience.

## 4. Reflection

Most of the concrete lessons are already covered above. A few more general observations:

### On "Objective" Extraction

Building on the PCA analogy from §2.1: an **objective** pattern extraction might actually be the least useful kind — because it has no perspective, and therefore no priority (it can't constrain the agent's action space). Good distillation requires a clear stance, and then honest labeling of what that stance is. (This feels close to concepts in Ray Dalio's *Principles* — I've actually been building a personal Principles Skill based on that book.)

### On the Roadmap

Future analyses of Codex CLI and Gemini CLI will use the same set of basis vectors. If a pattern appears along the same direction across three independent implementations, it likely reflects the structure of the problem domain itself — not any single team's design preference (and not just mine). That's perhaps the most worthwhile hypothesis to test.

## References

- [Agentic Harness Patterns Skill](https://github.com/keli-wen/agentic-harness-patterns-skill) — the distillation output
- [One Poem Suffices: Context Engineering](https://keli-wen.github.io/One-Poem-Suffices/one-poem-suffices/context-engineering/) — basis vector #1
- [One Poem Suffices: Just-in-Time Context](https://keli-wen.github.io/One-Poem-Suffices/one-poem-suffices/just-in-time-context/) — basis vector #2
- [Thinking in Context: Context Engineering from Codex](https://keli-wen.github.io/One-Poem-Suffices/thinking-in-context/context-engineering-from-codex/) — basis vector #3
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [OpenAI: Harness engineering](https://openai.com/index/harness-engineering/)
- [OpenAI: How we used Codex to build Sora for Android](https://openai.com/index/shipping-sora-for-android-with-codex/)
