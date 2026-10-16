---
name: team-code-understanding
description: Understand how a topic, objective, or change relates to the code base in the current directory — its code, design, and architecture — using a team of subagents. Scouts sketch the repo and propose focus areas; after the user confirms, researchers (including an adversarial one and web researchers) dig in; an arbiter keeps only well-cited claims and writes a markdown report. Use when the user says "team-code-understanding", "help me understand how X fits into this code base", or wants a rigorous, cited map of a topic against a repo. Not for quick lookups.
---

# Team Code Understanding

Input: a topic or objective from the user (ask once if missing). The code base is the current working directory. Output: `understanding-<topic-slug>.md` in the cwd.

Rules for every agent you spawn:
- Read-only. No edits to the repo.
- Every claim carries a citation: `path:line` (or `path:start-end`), a commit SHA, or a URL. No citation, no claim.
- Label each claim `FACT` (read directly in the cited source), `INFERENCE` (reasoned from cited facts; list them), or `UNKNOWN` (looked, could not determine).
- Return claims as a list: `id | label | statement | citation(s) | confidence (high/med/low)`.

## Phase 1 — Expedition (fast, cheap)

Spawn 3–5 `Explore` agents in parallel, one message. Each gets the topic plus one lens:
1. **Structure** — languages, build system, module/package layout, entry points, size.
2. **Topic hits** — where the topic (and its synonyms) appears: code, config, tests, docs, ADRs, issues/commit messages.
3. **Architecture & data flow** — the main runtime paths the topic could touch; boundaries, interfaces, persistence.
4. **History & ownership** — `git log`/blame hot spots near the topic, recent churn, key contributors (skip if not a git repo).
5. (If the repo is large or polyglot) **Dependencies & integration edges** — external services, libraries, protocols relevant to the topic.

Each returns ≤ 400 words: a sketch plus 2–5 candidate focus areas, each with the paths involved and a size estimate, **plus a list of recommended experts** (see below).

**Expert recommendations.** The team in Phase 2 is a team of genuine domain experts, chosen from what the scouts actually found: languages, runtimes, frameworks, data stores, protocols, and the nature of the topic. Each scout names the specific expertise the topic demands here, with a one-line reason and the paths it applies to. Be specific and deep, not generic. Examples (illustrative, not a fixed list; the skill is language-agnostic):
- Java + shared mutable state: *Java Memory Model / concurrency expert* (happens-before, volatile/final-field semantics, `VarHandle`, safe publication, lock striping, `j.u.c` internals).
- Rust + `unsafe`: *Rust ownership/unsafe expert* (aliasing rules, `Send`/`Sync`, Pin, drop order).
- Go services: *Go runtime/scheduler expert* (channel semantics, goroutine leaks, memory model).
- Python at scale: *CPython internals expert* (GIL, asyncio event loop, GC, C-extension boundaries).
- Distributed systems: *consensus/consistency expert* (Raft/Paxos, linearizability, clock and failure models).
- Also as relevant: storage engines, query planners, network protocols, serialization and wire formats, JVM/GC tuning, build and supply chain, security/crypto, performance and cache behaviour, a specific framework or library.

Then you (the orchestrator):
- Merge the scouts' output into a deduplicated list of **focus areas** (typically 3–7). For each: one-line description, key paths, why it matters to the topic.
- Gauge complexity (repo size, number of focus areas, how scattered the topic is) → small / medium / large. This sets Phase 2 team size.
- **Recommend** a focus (which areas, and why) and say how many researchers that implies.
- Present to the user and **stop**. Ask them to confirm, drop, add, or re-prioritise areas. Do not start Phase 2 until they reply.

## Phase 2 — Research (after confirmation)

Spawn in parallel, one message. Scale to complexity:

| Complexity | Expert code researchers | Adversarial | Web researchers |
|---|---|---|---|
| small | 1–2 experts | 1 | 1 |
| medium | 3–5 experts | 1 | 2 |
| large | 6+ experts (one per expertise or expertise × area) | 2 | 3 |

- **Expert code researchers** — assembled from the scouts' expert recommendations (merge duplicates, drop the marginal), mapped to the confirmed focus areas. One researcher per expertise, or per expertise × focus area when an area is large. Include the expert list, with reasons, in the Phase 1 presentation so the user can add or remove experts when confirming. Each is briefed as a **genuine specialist**: *"You are a world-class expert in <specific domain>. Bring the full depth of that expertise: the precise semantics, specification-level rules, known pitfalls, and failure modes, not a surface read."* They go through the code with a fine-grained, analytical orientation: read the actual call sites, types, config, and tests line by line; apply the domain's exact rules to what the code does; and map how the topic is positioned against it (what exists, what is missing, what the topic would touch, what constrains it, what would break). Example: a concurrency expert on a Java code base identifies each shared mutable field the topic touches, states the happens-before edges (or their absence) that guard it, and cites the lines. A claim that merely restates the code without applying domain knowledge is weak; the arbiter should prefer claims where expertise adds something. Give each the topic, their focus areas, the relevant scout findings, and the claim format. Trace real flows end to end.
- **Adversarial researcher(s)** — also expert-level (match the repo's dominant language and runtime), told to try to break the emerging picture: find counter-examples, dead code that looks live, config/feature-flag paths that change behaviour, tests that contradict assumptions, places where the obvious reading is wrong, and gaps scouts glossed over. Give them the scout sketch and focus areas. They return refutations and risks in the same claim format.
- **Web researchers** (`general-purpose`, since they need WebSearch/WebFetch) — non-code context only: specs, RFCs, vendor/library docs, known issues, prior art, design rationale, blog posts by maintainers. Use WebSearch/WebFetch. Cite URLs; prefer primary sources; note publication date. Must not make claims about this repo's code.

## Phase 3 — Arbitration

Spawn one **arbiter** agent (a fresh `general-purpose` agent; note its ID) and give it all claim lists, the topic, and the confirmed focus. It must:

1. Keep a claim only if the citation exists and supports the statement. Spot-check by reading the cited lines/URLs; check the high-impact claims first.
2. Drop uncited, vague, speculative, or duplicated claims, and claims that only restate code without expert analysis. Resolve conflicts between researchers (adversarial findings that survive checking override a researcher's claim and are reported as caveats).
3. If a claim is important but weak, it may ask **the originating researcher** to confirm it. Subagents can't message each other, so the arbiter ends its turn with a `CONFIRM` list instead of a report: `claim id | researcher | exact question`. Budget: at most ~5 queries in total, one round per claim. You (the orchestrator) forward each question to that researcher with `SendMessage` (keep the agent IDs from Phase 2), then `SendMessage` the answers back to the arbiter, which continues to step 4. Claims still weak after that are dropped or downgraded to `INFERENCE`/`UNKNOWN`. If nothing needs confirming, the arbiter goes straight to step 4.
4. Write the report with this structure:
   - **Summary** — the topic positioned against the code base, in a few sentences.
   - **How it fits** — per focus area: what exists, how it works, key paths (all cited).
   - **Design & architecture constraints** — what shapes or limits the topic here.
   - **External context** — web findings, cited with URL and date.
   - **Risks, gaps, and counter-evidence** — adversarial results that held up.
   - **Open questions** — `UNKNOWN`s worth the user's attention.
   - **Where to start reading** — ordered file list.
   Citations inline as `[path:line]` or `[url]`. Mark `INFERENCE` claims explicitly.
5. **Anti-slop pass before output.** Invoke the `stop-slop` skill on the draft if it is available; otherwise apply these by hand: delete filler openers and closers, hedging stacks, "delve/leverage/robust/seamless/comprehensive/crucial" style vocabulary, rule-of-three padding, em-dash overuse, "not just X but Y" constructions, and restating the question. Prefer short, concrete sentences. The pass must not remove or alter any citation or change a claim's meaning.
6. Write the file `understanding-<topic-slug>.md` in the cwd and return only the path plus a 3–5 line summary.

You (the orchestrator) then show the user the path and summary. Do not paste the whole report unless asked.

## Limits

- Don't over-spawn: more agents than focus areas just duplicates work.
- Don't re-run Phase 1 after confirmation; if the user changes focus materially, update the list and re-confirm.
- If the cwd isn't a code base, say so and stop.
