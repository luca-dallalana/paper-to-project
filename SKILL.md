---
name: paper-to-project
description: Turns a research paper into a grounded implementation guide instead of generic advice - a plain-English summary, prerequisites, core data structures, key algorithms with pseudocode, a build order, the edge cases the paper glosses over, and a verification strategy. Use whenever the user hands over a paper (pasted text, PDF, or URL) and wants to build, implement, or code up what it describes - phrasings like "help me implement this paper", "I want to build this", "extract what I need to implement from this", "what does this paper actually require me to build", or "implementation guide for this" should all trigger it. Also trigger when the user pastes paper content and asks any implementation-adjacent question, even without the word "implement" - e.g. "how would I build the leader-change part of this" or "what state do I need to track here". Do not trigger for requests to summarize, review, or critique a paper academically with no stated intent to build or code something from it.
---

# Paper to Project

Research papers are optimized to prove correctness to reviewers, not to guide an implementer. They hand-wave edge cases ("the leader eventually times out"), use mathematical notation that doesn't map cleanly onto data structures, assume prerequisite knowledge without stating it, and bury the ordering and gotchas that matter most for writing code in a proof appendix or an offhand sentence in the evaluation section. This skill's job is to do the extraction work a working implementer would do by hand: read the paper the way someone about to write code reads it, and produce a guide grounded in what the paper actually says, not a generic tutorial on the paper's topic.

The output is a document, not a conversation. Someone should be able to open it in a second window while writing code and use it as a working reference - which means it belongs in a file, not only in the chat transcript. A guide this long is unwieldy to scroll through as a chat message and easy to lose once the conversation moves on.

## Step 1: Get the actual paper content

Before writing anything, make sure you have the full paper in front of you, not a summary of it.

- **Pasted text**: use it directly. If it looks truncated (cuts off mid-section, missing references or appendices the text refers to), say so before proceeding rather than filling gaps from general knowledge of the topic.
- **PDF**: read the entire document, including appendices. For algorithmic and systems papers, the appendix frequently contains the complete pseudocode, the full proof (which reveals invariants the main text only gestures at), or implementation notes that got cut from the main text for space. Skipping the appendix is the single easiest way to produce a shallow guide.
- **URL**: fetch it. For arXiv links, the abstract page alone is not enough - get the full paper (HTML version if available, otherwise the PDF).

Do not proceed to Step 2 on a partial read. If the paper is long, it is worth the extra time to read it completely once rather than producing a guide that misses something the last two pages covered.

## Step 2: Produce the guide

Output exactly these seven sections, in this order, using these headers. Do not add extra top-level sections, and do not skip one because the paper seems to make it trivial - if a section is genuinely thin for this paper, say so briefly in that section rather than omitting it.

### 1. Summary

One paragraph, plain English, written for someone about to implement this, not someone reviewing it for a conference. State the problem being solved, the core mechanism that solves it, and why that mechanism works well enough to matter in practice. Skip the academic framing (motivation via related work, contribution bullet points) - that's for a different audience.

### 2. Prerequisites

List the specific concepts someone needs before this paper will make sense, each with a one-line description and, where it helps, a pointer to where they could learn it (a named textbook, a classic paper, a well-known concept to search for - don't invent a URL for it).

Specificity is the whole value of this section. "Understand distributed systems" is useless. "Understand quorum-based voting: in a system of n nodes, any two sets of size floor(n/2)+1 must overlap, which is what prevents two conflicting decisions from both getting a majority" is what makes this section worth reading. If you can't state the concept concretely enough to give it a one-line description, it's too vague to include.

Only list what this specific paper actually assumes - check the paper's introduction and notation section for what it takes for granted, rather than listing generic background for the field.

### 3. Core data structures

For every data structure the paper needs - whether it names it explicitly or the structure is only implied by the notation - give:

- **Name** (use the paper's name if it has one, otherwise a name you'd actually use in code)
- **Represents**: what it is, in one sentence
- **Fields**: name and type for each field
- **Invariants**: what must always hold about this structure, and at what points in the algorithm it could be violated if the implementation is wrong

Ground each one in where it comes from in the paper (a section, a definition, a figure). If the paper defines a structure only through prose or a formula rather than a clean listing, that translation from math to fields-and-types is exactly the work this section exists to do - don't just quote the formula back.

### 4. Key algorithms

For each algorithm, protocol phase, or procedure in the paper:

- Extract the pseudocode if the paper gives it, or write it out from the prose description if it doesn't
- Translate mathematical notation into concrete operations a data structure from Section 3 would actually perform - a summation with a threshold condition becomes "count matching items and compare to a threshold", a normalization term becomes "divide by the computed statistic over the relevant dimension" - not a restatement of the formula in prose
- Annotate each step that has a way to get it wrong: what an implementation must check at that point, and what happens if it doesn't

Cite the section, algorithm number, or equation the pseudocode is drawn from so this stays checkable against the source.

### 5. Implementation order

A numbered build sequence, ordered so each step produces something you can run or test before the next step adds complexity. This is not the order the paper presents ideas in - papers usually present the full mechanism before any of the special cases, which is backwards for building it. Favor an order where the safety-critical, deterministic core comes before the parts that depend on timing, concurrency, or networked coordination, since those are the pieces hardest to debug once several of them interact at once.

### 6. What the paper glosses over

This is the highest-value section, and the easiest one to fake. Don't write generic caveats - hunt for the specific places this paper's own text says "eventually", "assume", "for simplicity", or simply never mentions a scenario its own mechanism makes possible.

What to hunt for depends on what kind of mechanism the paper actually describes. The two patterns below are given to show the depth expected, not as a checklist to run against every paper - most papers are neither, and deserve their own version of this question rather than being forced into one of these two:

- **A protocol with multiple roles, phases, or rounds** (consensus, coordination, distributed protocols): what happens if a participant crashes or restarts mid-phase, not just between phases. What happens when a message arrives twice, out of order, or not at all. What happens when two instances of the protocol overlap. What the paper's liveness assumption ("eventually synchronous", "eventually the leader is correct") actually has to become in code - almost always a timeout and a retry policy the paper never specifies a value for. What state must be persisted to survive a restart versus what can live only in memory.

- **A numerical or learned method** (ML, optimization, statistical estimation): what the algorithm box leaves out that any real run needs (initialization scheme, numerical epsilon terms to avoid division or log of zero, normalization, a schedule or warmup). What hyperparameters only appear in the experiments section or an appendix table rather than as part of the stated method. Where the claimed result is sensitive to a choice the paper doesn't flag as a choice.

For anything else, ask the equivalent question in that paper's own terms - concretely imagine what would break this specific mechanism, and check whether the paper addresses it. A new data structure needs its concurrent-access behavior and its resize, rebalance, or eviction path interrogated (what happens on the worst-case sequence of operations the paper's average-case analysis doesn't cover). A cryptographic construction needs its adversary model boundaries and parameter choices interrogated (what security level a given parameter actually buys you, what happens outside the stated adversary model). A compiler or type-system result needs its soundness assumptions checked against malformed or adversarial input. The point is never to match a template - it's to name the specific failure this mechanism is prone to and confirm whether the paper's text actually covers it.

Also, for any paper: notation reused for two different things, or defined once and never quite pinned down (what exactly is in scope of a variable, what its valid range is). Edge cases the definitions don't address (the smallest possible n, an empty set, a tie). Anything the correctness proof assumes that the "practical" version of the algorithm quietly drops.

Where you're confident based on how these systems or methods behave in practice, say so directly and specifically - don't hedge everything into "you should research this further." Where you are inferring beyond what the paper states, say that too, so the reader knows which parts are grounded in the text and which are added implementation knowledge.

### 7. Verification strategy

Concrete ways to know the implementation is right, not just that it runs:

- Test cases that exercise each invariant named in Section 3 - what input drives the system right up against that invariant
- Fault injection scenarios pulled directly from Section 6's gotchas - the scenarios that separate an implementation that merely works on the happy path from one that's actually correct (for protocols: crash a participant mid-phase, duplicate a message, partition the network, delay delivery; for ML methods: a toy dataset small enough to overfit as a sanity check, a numerical edge case, an ablation that isolates one claim of the paper at a time)
- If the paper states a formal property (safety, convergence, a bound), what a test would have to do to give you confidence in that property specifically, as opposed to just confidence that the code doesn't crash

## Step 3: Save it as a file

Write the finished guide to a markdown file rather than only posting it inline in chat. Use a kebab-case slug of the paper's title or its common name, e.g. `hotstuff-implementation-guide.md` or `paxos-made-simple-implementation-guide.md`, saved in the current working directory (or wherever the user's project files live, if that's clearly established). Tell the user the file path once it's written, and give a short summary of what's in it rather than repeating the whole document in chat - the file is the deliverable, the chat message is a pointer to it.

If there's no real filesystem to write to (no file tools available, or the environment makes a saved file useless to the user), post the guide inline in full instead - the point is that the guide ends up somewhere the user can actually keep and reopen, not that a file gets created for its own sake.

## Grounding, not hallucination

Every claim in Sections 3, 4, and 6 should be traceable to somewhere specific in the paper - a section number, an algorithm/figure number, or an equation. If something a real implementation needs simply isn't in the paper, say that plainly in the relevant section instead of inventing a plausible-sounding value and presenting it as the paper's own content. The value of this guide over generic advice is that a reader can check it against the source; silent invention destroys that.

## Scope discipline

If the user's actual intent is a summary, a critique, or an academic review of the paper rather than building something from it, that's a different task - don't force this seven-section structure onto it. This skill is specifically for the moment someone is about to write code against a paper and needs to know what that requires.
