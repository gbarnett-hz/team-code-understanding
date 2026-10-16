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

Each returns ≤ 400 words: a sketch plus 2–5 candidate focus areas, each with the paths involved and a size estimate.

Then you (the orchestrator):
- Merge the scouts' output into a deduplicated list of **focus areas** (typically 3–7). For each: one-line description, key paths, why it matters to the topic.
- Gauge complexity (repo size, number of focus areas, how scattered the topic is) → small / medium / large. This sets Phase 2 team size.
- **Recommend** a focus (which areas, and why) and say how many researchers that implies.
- Present to the user and **stop**. Ask them to confirm, drop, add, or re-prioritise areas. Do not start Phase 2 until they reply.

## Phase 2 — Research (after confirmation)

Spawn in parallel, one message. Scale to complexity:

| Complexity | Code researchers | Adversarial | Web researchers |
|---|---|---|---|
| small | 1–2 | 1 | 1 |
| medium | 3–5 | 1 | 2 |
| large | 6+ (one per focus area or cluster) | 2 | 3 |

- **Code researchers** — one per confirmed focus area. Brief: the topic, the focus area, scout findings relevant to it, the claim format. Trace real flows end to end (call sites, data, config, tests). Positioning question: *how does the topic relate to this code — what exists, what is missing, what would it touch, what constrains it?*
- **Adversarial researcher(s)** — told to try to break the emerging picture: find counter-examples, dead code that looks live, config/feature-flag paths that change behaviour, tests that contradict assumptions, places where the obvious reading is wrong, and gaps scouts glossed over. Give them the scout sketch and focus areas. They return refutations and risks in the same claim format.
- **Web researchers** — non-code context only: specs, RFCs, vendor/library docs, known issues, prior art, design rationale, blog posts by maintainers. Use WebSearch/WebFetch. Cite URLs; prefer primary sources; note publication date. Must not make claims about this repo's code.

## Phase 3 — Arbitration

Spawn one **arbiter** agent (a fresh `general-purpose` agent) and give it all claim lists, the topic, and the confirmed focus. It must:

1. Keep a claim only if the citation exists and supports the statement. Spot-check by reading the cited lines/URLs; check the high-impact claims first.
2. Drop uncited, vague, speculative, or duplicated claims. Resolve conflicts between researchers (adversarial findings that survive checking override a researcher's claim and are reported as caveats).
3. If a claim is important but weak, it may ask **the originating researcher** (via `SendMessage`) to confirm it. Budget: at most one round per claim, and no more than ~5 queries in total. After that, drop it or downgrade it to `INFERENCE`/`UNKNOWN`.
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
