# paper-to-project

A Claude Code skill that turns a research paper into a grounded implementation guide instead of generic advice.

Research papers are written to prove correctness to reviewers, not to guide someone writing code. They hand-wave edge cases, use notation that doesn't map cleanly onto data structures, assume prerequisite knowledge without stating it, and bury the ordering and gotchas that matter most for implementation in a proof appendix or an offhand sentence. This skill reads a paper the way an implementer would, and produces a document grounded in what the paper actually says: section by section, figure by figure, equation by equation, not a summary of the paper's topic.

## What it produces

Hand it a paper, pasted text, a PDF, or a URL, along with an implementation-intent request ("help me implement this", "what do I actually need to build here", "extract what I need to implement from this"), and it writes a markdown guide with these seven sections, in order:

1. **Summary** - plain-English statement of the problem and mechanism, written for an implementer
2. **Prerequisites** - the specific concepts you need before the paper makes sense, not generic background
3. **Core data structures** - every structure the paper needs, named, typed, with invariants
4. **Key algorithms** - pseudocode translated from the paper's notation, annotated with what can go wrong at each step
5. **Implementation order** - a build sequence ordered for incremental testability, not the paper's presentation order
6. **What the paper glosses over** - the edge cases, implicit assumptions, and hand-waved parts, distinguishing what the paper itself flags from what's added implementation knowledge
7. **Verification strategy** - test cases and fault injection scenarios that exercise the paper's actual invariants

It does not trigger for requests to summarize, review, or critique a paper academically - that's a different task.

## Install

Copy `SKILL.md` into your Claude Code skills directory:

```
mkdir -p ~/.claude/skills/paper-to-project
cp SKILL.md ~/.claude/skills/paper-to-project/SKILL.md
```

Or, if you have the packaged `.skill` file, unzip it into `~/.claude/skills/`.

## Use

Once installed, just hand Claude a paper and ask to implement it:

> "help me implement this paper: https://arxiv.org/abs/1803.05069"

No special syntax needed - the skill triggers automatically on implementation-intent requests.

## Tested against

HotStuff, Paxos Made Simple, and Vertical Paxos (distributed consensus), Proximal Policy Optimization (reinforcement learning), and Cuckoo Hashing (data structures) - across all three input modes (pasted text, local PDF, URL).

## Proof of concept: does the skill actually help?

Curated examples are easy to cherry-pick, so this was checked with a controlled comparison instead. Two agents got the identical extracted text of the HotStuff paper (arXiv:1803.05069), the identical prompt, and the same model. The only difference between them: whether the `paper-to-project` skill was invoked. One agent was explicitly forbidden from using any skill (the control); the other was explicitly required to invoke `paper-to-project` (the treatment).

Prompt used (pulled verbatim from this repo's `eval_set.json`):

> "ok so my systems professor assigned the HotStuff paper (the BFT one, arxiv 1803.05069) for our final project and we have to actually implement a working prototype in Java by december. can you help me figure out what to build first"

An 8-item gap rubric was written down first, grounded in the paper's own text and quoted with section numbers, before either output was read, to avoid grading toward a predetermined answer.

### Result

| Gap | No skill | Skill |
|---|---|---|
| Catch-up protocol omitted from pseudocode (Sec 4.2) | PASS | PASS |
| Catch-up colliding with block-tree pruning | MISS | MISS |
| Reference implementation skips real threshold signatures (Sec 8.1/8.3) | PARTIAL | PASS |
| Crash recovery / durable state | PARTIAL | PASS |
| Pacemaker timeout values unspecified (Sec 4.4) | PARTIAL | PASS |
| Blank/dummy node padding (Sec 5) | PASS | PASS |
| Genesis block bootstrap edge case (Sec 6) | MISS | PASS |
| Duplicate votes: correctly not claimed as a gap | PASS | PASS |

Aggregate: 3 PASS / 3 PARTIAL / 2 MISS without the skill, versus 7 PASS / 0 PARTIAL / 1 MISS with it. Every citation above was checked against the paper text directly (not taken from either agent's say-so).

### The one thing both runs missed

Neither agent found the interaction between the catch-up/sync protocol and block-tree pruning, the specific gap that made building [DepChain](https://github.com/luca-dallalana/DepChain) hard. That gap only became visible after actually building the protocol and going back to the paper with a "there's a gap here" mindset. The paper never mentions pruning or garbage collection at all, so there was nothing in the text for either agent to notice, skill or no skill.

What the skill is actually good at, evidenced above, is different and narrower: it reliably surfaces what the paper itself hedges on ("for brevity, omitted from the pseudocode," "left implicit," "specified by the application"), cited to the exact section and line, without inventing gaps that aren't there. It is not a substitute for the insight that only comes from building the thing.

### Full outputs

<details>
<summary><b>Without the skill (control)</b></summary>

````markdown
Good news: this is a very implementable paper, and the fact that you have until December means you can afford to build it in the right order instead of the fast-and-wrong order. Here's how I'd sequence it.

## Before writing anything: pin down scope with your professor

The paper actually describes three increasingly refined versions of the protocol, and "implement HotStuff" is ambiguous between them:

- **Basic HotStuff** (Algorithm 2, Section 4): one view = one proposal, four sequential phases (prepare, pre-commit, commit, decide). Easiest to reason about and test, but not pipelined.
- **Chained HotStuff** (Algorithm 3, Section 5): same logic, but each view only does one phase's worth of work and phases from consecutive proposals overlap. This is where "linear view change" and pipelining actually pay off.
- **Event-driven HotStuff** (Algorithm 4 + Pacemaker in Algorithm 5, Section 6): the authors say this "is almost like the code skeleton for a prototype implementation" - it's literally what their real (C++) implementation looks like, with safety (`update`/`onReceiveProposal`/`onReceiveVote`) cleanly separated from liveness (the `Pacemaker`).

If the assignment says "implement HotStuff," find out whether they want (a) just the safety/commit logic working correctly, (b) the pipelined/chained version, and (c) real networking + a leader-election/timeout scheme, or whether a single-JVM simulation with in-memory message passing is acceptable. That answer changes what "first" means. Absent guidance, I'd target the event-driven version as the end goal, since it's what the paper itself presents as the implementation-ready spec, but get there by building Basic HotStuff first (it's much easier to unit test) and then refactoring toward the chained/event-driven form.

## Build order

**Milestone 0 - data model and message shapes.** Before any protocol logic, nail down the objects everything else depends on: a tree node (parent link, command/batch, and eventually a `justify` QC field and `height`), the QC type (`type`, `viewNumber`/`height`, `node`, and the aggregated signature), and the five message types from Algorithm 1 (`new-view`, `prepare`, `pre-commit`, `commit`, `decide` for Basic; collapsed to `new-view`/`generic` for Chained). Get equality/hashing and "does branch A extend branch B" (ancestry check) right here, since almost every later bug traces back to a broken extends-check or a broken hash-based node identity.

**Milestone 1 - single-JVM simulation of Basic HotStuff, no faults, no timeouts.** This is the smallest slice that's actually worth building first: 4 replicas (n=4, f=1) running as threads or actors in one process, talking over in-memory queues (not sockets yet), one fixed leader that never fails, network never drops or reorders messages. Implement Algorithm 2 exactly as written - the leader's prepare/pre-commit/commit/decide broadcasts, the replica's vote logic, and the `safeNode` predicate from Algorithm 1 (both disjuncts: the safety rule "extends `lockedQC.node`" and the liveness rule "`justify.viewNumber > lockedQC.viewNumber`"). Success criterion: all 4 replicas commit the same command in the same order and you can print the resulting log. Don't touch view-change or crypto correctness yet - just get one full view to execute end to end.

On signatures: the paper's QC construction (`tcombine`/threshold signatures) is doing real cryptographic work you don't need to reproduce faithfully to get correct protocol *behavior*. Java has no mainstream threshold-BLS library, and building one is its own multi-week project unrelated to what this assignment is actually testing. A defensible simplification: each vote carries a normal digital signature (ECDSA via `java.security.Signature` is built into the JDK) over `(type, viewNumber, node)`, and a QC is just a set of n-f such signatures rather than a single combined constant-size value. Protocol logic and safety are unaffected; you just lose the O(n) vs. O(1)-sized-authenticator distinction the paper cares about for its complexity claims. Flag this explicitly as a scoping decision in your writeup - it's the single most common trap where people burn weeks on crypto plumbing instead of the consensus logic the professor is actually grading.

**Milestone 2 - Byzantine/faulty behavior, still single JVM.** Add a replica that can equivocate (send conflicting proposals as leader, or vote for two conflicting branches) or simply go silent. Verify Theorem 2 empirically: with f=1 out of n=4, no two conflicting nodes ever both get committed, no matter what the faulty replica does. This is a good place to also implement the paper's own worked counterexample: the "two-phase HotStuff" livelock in Section 4.4 (drop the pre-commit phase, watch the system fail to make progress forever). Building that failure case is cheap and is a great way to demonstrate you actually understand *why* the third phase exists, not just that you copied the pseudocode.

**Milestone 3 - liveness: nextView timeouts and leader rotation.** Add the `nextView` interrupt and a round-robin `leader(viewNumber)` schedule (the paper explicitly says any deterministic rotating schedule suffices, Section 4.4). Now you can kill the current leader mid-view and confirm the system advances to the next view and eventually commits - this is where you're exercising the "linear view change" property that's the paper's whole point. Use exponential backoff on the timeout as the paper suggests, so replicas' waiting windows eventually overlap.

**Milestone 4 - move from in-process queues to real networking.** Only after 1-3 work, swap the in-memory `Network` abstraction for real sockets (or something like Java NIO / gRPC / plain length-prefixed TCP - doesn't need to be fancy) and serialize messages. Because you kept a clean interface boundary between "protocol logic" and "how a message gets from A to B" from the start, this should be a localized change, not a rewrite. This is also where you'll hit the thing the paper mentions but omits from pseudocode: a replica that's behind needs to fetch missing ancestor nodes from peers before it can process a message referencing them (mentioned around the "Tree and branches" paragraph in Section 4.2) - real networks reorder and drop, so you actually need this once you're off in-memory queues.

**Milestone 5 - refactor to Chained HotStuff, then the event-driven/Pacemaker form.** Once Basic HotStuff is solid, restructure around Algorithm 3's single `generic` phase per view and the One-Chain/Two-Chain/Three-Chain commit rule, then Algorithm 4's `update`/`onReceiveProposal`/`onReceiveVote`/`onCommit` plus the Pacemaker skeleton (Algorithm 5). Watch for the "dummy/blank node" detail (Section 5, "Dummy nodes" paragraph): `createLeaf` needs to pad with blank nodes up to the current view's height when the previous view's leader didn't get a QC, so that view number and node height stay in lockstep - this is easy to skip and will silently break your chain-length checks later.

**Milestone 6 (stretch, if time allows) - the evaluation angle.** The paper's own evaluation (Section 8) measures authenticator counts during view-change (Figure 9/10) to show BFT-SMaRt scales cubically while HotStuff stays linear. Instrumenting your own implementation to count authenticators per view change and plotting it as n grows is a cheap, high-payoff experiment for a final report, since it's a direct, quantitative demonstration of the paper's central claim, and you already have the counting logic mostly for free from your message objects.

## Specific correctness gotchas the paper calls out (Appendix B.1) that are worth reading closely before you code the commit rule

- **Monotonic `vheight`:** a replica must never vote for a lower node after voting for a higher one, even across different branches. The appendix gives a concrete two-line counterexample of why relaxing this breaks safety - worth re-deriving by hand so you don't reintroduce it under refactoring pressure.
- **Direct-parent requirement for commit:** the Three-Chain must be of *direct* parents (`b*.justify.node = b*.parent`, not just an ancestor) for the safety proof's height argument to hold. It's tempting to "relax" this for a simpler implementation (the paper notes you can relax it for updating `qcHigh`/`lockedQC`, just not for the commit decision itself) - mixing these up is an easy way to silently break Theorem 8.
- **Duplicate-vote guarding:** `onReceiveVote` (Algorithm 4, line 23) explicitly checks it hasn't already recorded a vote from that sender for that node - skip this and a single message replay or duplicate delivery inflates your vote count past a real quorum.

## A reference point, not a template

The original authors open-sourced a C++ implementation (`libhotstuff`) that follows Algorithm 4/5 closely. It's worth skimming once you're at Milestone 5 to sanity-check your understanding of the Pacemaker/safety split, but I wouldn't start from it or port its architecture - its concurrency model is C++-specific and won't map cleanly onto Java, and the point of the assignment is presumably that *you* work through the state-machine design, not that you replicate someone else's class layout.

One thing I'd genuinely push back on if I were reviewing your plan: don't start with networking or with real threshold signatures, even though both feel like "real distributed systems" work and might feel like the obvious first step. Neither is where the paper's actual intellectual content lives - the safety argument (`safeNode`, the locked-QC mechanism, the Three-Chain commit rule) is the part that's hard to get right and the part a grader will actually probe with adversarial scenarios. Get that correct on 4 threads in one JVM first; sockets and real crypto are comparatively mechanical additions once the state machine is right.
````

</details>

<details>
<summary><b>With the skill (treatment)</b></summary>

````markdown
# HotStuff Implementation Guide
### For: "HotStuff: BFT Consensus in the Lens of Blockchain" (arXiv:1803.05069, Yin, Malkhi, Reiter, Golan Gueta, Abraham)
### Target: Java prototype, course project timeline

---

## 1. Summary

HotStuff is a protocol for Byzantine fault-tolerant state machine replication (BFT SMR): a fixed group of `n = 3f+1` replicas needs to agree on one ordered log of client commands even though up to `f` of them may behave arbitrarily (crash, lie, collude). The problem every leader-based BFT protocol before it shared was that when a leader is replaced, the new leader has to prove to everyone that its proposal is safe, and that proof (in PBFT and its descendants) requires relaying the last-known certificate from every replica in the new quorum - an `O(n^2)`-`O(n^3)` message per view-change, which becomes `O(n^3)`-`O(n^4)` over a cascade of failed leaders (Section 1, Table 1).

HotStuff's fix is structural: every proposal a leader makes is chained to a **Quorum Certificate (QC)** - a proof that `n-f` replicas voted for it - and the protocol requires *three* successive QC-carrying phases (`prepare` -> `pre-commit` -> `commit`, Section 4.1) before a value is safe to execute, instead of two. The extra phase means a replica can always safely change its vote in favor of *any* higher QC it later learns about, without anyone needing to construct a leader-change proof at all: a new leader just picks the single highest QC it has heard of and proposes from there (Section 4.1, `highQC`). That is the whole trick. Because leader replacement now costs exactly what a normal round costs (`O(n)` authenticators), leaders can be rotated on every single decision for free, which is exactly what the paper does in "Chained HotStuff" (Section 5) to turn the three sequential phases into a pipeline where every view does one phase's worth of work for three overlapping proposals simultaneously. The payoff, demonstrated in Section 8 against BFT-SMaRt at over 100 replicas, is comparable throughput/latency on the happy path and *no* blowup in leader-change cost as `n` grows, where PBFT-style systems degrade cubically.

For an implementer, the paper is really three protocols layered on top of each other: **Basic HotStuff** (Algorithm 2, a clean but non-pipelined view-per-phase protocol - the one to prove correctness against), **Chained HotStuff** (Algorithm 3, the pipelined version), and **Event-driven HotStuff** (Algorithm 4 + a Pacemaker, Section 6) - the one the authors say is "almost like the code skeleton for a prototype implementation" (line 605) and the one their own C++ prototype is built from.

---

## 2. Prerequisites

- **BFT quorum arithmetic.** With `n = 3f+1` and a quorum size of `n-f = 2f+1`, any two quorums intersect in at least `(n-f)+(n-f)-n = f+1` replicas - enough that at least one intersecting replica must be correct. This single fact is what every safety proof in the paper (Lemma 1, Theorem 2) reduces to. If you can't immediately see why two same-view, conflicting QCs are impossible from this arithmetic, re-derive it before writing any code - it's the load-bearing fact for the entire system.
- **Partial synchrony (Dwork-Lynch-Stockmeyer, 1988, cited as [25]).** There is a message delay bound `Delta` and an unknown Global Stabilization Time (GST); before GST, delays are unbounded and arbitrary; after GST, every message between correct replicas arrives within `Delta`. Safety must hold unconditionally; liveness (making progress) is only guaranteed to eventually resume after GST (Section 3). This is why the paper's pseudocode has no explicit timing logic in the safety-critical path - timing only shows up in the separate `nextView`/Pacemaker mechanism.
- **Threshold signatures, `(k,n)` scheme.** Each of the `n` replicas holds a distinct private key share; `tsign_i(m)` produces a partial signature; `tcombine` merges any `k` partial signatures for the same `m` into one compact signature `sigma`; `tverify(m, sigma)` checks it. The paper fixes `k = 2f+1` (Section 3). Look up Shoup, "Practical Threshold Signatures" (Eurocrypt 2000) [48] or Boneh-Lynn-Shacham BLS signatures [14] if you want the real primitive, but see Section 6 of this guide below, because the paper's own prototype doesn't actually use one.
- **Collision-resistant hashing** as an identity function: `h(m)` stands in for "the unique name of `m`" throughout (Section 3) - this is how nodes reference their parent and how a QC references the node it certifies.
- **Familiarity with PBFT's two-phase structure** (Castro & Liskov, [20]) is not strictly required but makes Section 7 of the paper (and this guide's comparison to PBFT/Tendermint/Casper/DLS) much easier to follow, since HotStuff is presented explicitly as "PBFT plus one phase."
- **The separation of safety and liveness as a design pattern.** The paper deliberately splits the system into a safety-only core (Algorithm 4) and a pluggable liveness module called the Pacemaker (Algorithm 5, Section 6): "even if a bad Pacemaker invokes `onPropose` arbitrarily... safety is always guaranteed" (Section 6). Plan your class boundaries around this split from day one; retrofitting it later is painful.

---

## 3. Core data structures

### `Node` (the paper calls it a tree node / block; Section 4.2 "Tree and branches", `createLeaf` in Algorithm 1/3/4)
**Represents:** one proposed command (or batch), linked to its parent to form a tree of pending branches.

| Field | Type (Java) | Paper origin |
|---|---|---|
| `id` | `byte[]` (hash digest) | Section 3: "`h(m)` can serve as an identifier for a unique input `m`" |
| `parentId` | `byte[]` (hash of parent) | Section 4.2: "a parent link which could be a hash digest of the parent node" |
| `cmd` | `List<Command>` (a batch) | Section 4: "a proposed command (or a batch of them)" |
| `justify` | `QuorumCertificate` | Algorithm 4 line 1-3: `b.justify <- qc` |
| `height` | `long` | Algorithm 4 line 1: `b.height <- height`; Section 5: "view-numbers are equated with node heights" |

**Invariants:**
- The *branch* led by a node is the path to the root via `parentId` links (Section 4.2). Two branches conflict if neither extends the other; two nodes conflict if their branches conflict.
- `height` must strictly increase from parent to child - this is asserted implicitly everywhere but is the exact subject of Appendix B.1's "Why direct parent?" remark, which shows a concrete safety violation if you let the ancestry check degrade from "direct/immediate parent" to "any transitive ancestor" without also preserving the height ordering.
- `b.justify.node` must be an ancestor of `b` - stated explicitly in Section 6 ("we require the QC in a valid node always refers to its ancestor") and used directly in the safety proof of Appendix B (Lemma 7). Note this does **not** mean `b.justify.node == b.parent` in Chained/event-driven HotStuff - see "Dummy nodes" in Section 5 of the paper, covered in Section 6 of this guide.
- The genesis node `b0` is a special case: it has a **hardcoded, self-referential QC** (Section 6) - it is its own justification, and does not pass through normal `tverify`.

### `QuorumCertificate` (QC) (Section 4.2 "Quorum certificates", `QC(V)` in Algorithm 1 lines 15-20)
**Represents:** cryptographic proof that `n-f` replicas voted for the same `<type, viewNumber, node>` triple.

| Field | Type (Java) | Paper origin |
|---|---|---|
| `type` | `enum {NEW_VIEW, PREPARE, PRE_COMMIT, COMMIT, GENERIC}` | Section 4.2: `m.type` |
| `viewNumber` | `long` | Algorithm 1 line 17 |
| `nodeId` | `byte[]` | Algorithm 1 line 18 |
| `signature` | `byte[]` (combined/threshold sig, or in practice a `List<PartialSignature>`, see Section 6) | Algorithm 1 line 19: `tcombine(...)` |

**Invariants:**
- Valid iff `tverify(<qc.type, qc.viewNumber, qc.node>, qc.sig)` returns true (Section 4.4, opening definition of "valid").
- **Lemma 1** (Section 4.4): two valid QCs of the same `type` and the same `viewNumber` cannot certify conflicting nodes - this depends entirely on the invariant that *a correct replica casts at most one vote per (type, view)*. That invariant is stated in prose ("the pseudocode allows voting only once for each phase in each view") but is **not visibly enforced by any explicit check in Algorithm 2's pseudocode**, see Section 6 of this guide, "double-voting is assumed, not enforced."
- A QC's `node` must already be present in the local tree before the QC is used to justify anything further (Section 4.2: "a replica delivers a message only after the branch led by the node is already in its local tree").

### `Message` (Section 4.2 "Messages", `Msg`/`voteMsg` in Algorithm 1 lines 1-10)
**Represents:** either a leader's proposal or a replica's vote - the paper deliberately gives these one shared shape.

| Field | Type (Java) | Paper origin |
|---|---|---|
| `type` | same enum as QC, plus `NEW_VIEW` | Algorithm 1 line 2 |
| `viewNumber` | `long`, auto-stamped with sender's `curView` | Algorithm 1 line 3 |
| `node` | `Node` (or `null` for pre-commit/commit/decide in Basic HotStuff, which carry the QC instead) | Algorithm 1 line 4 |
| `justify` | `QuorumCertificate` (optional) | Algorithm 1 line 5 |
| `partialSig` | `PartialSignature` (present only on vote messages) | Algorithm 1 line 9: `tsign_r(<m.type, m.viewNumber, m.node>)` |

**Invariants:** a vote's `partialSig` must be over exactly `<type, viewNumber, node>`, not over the whole message, otherwise `tcombine` cannot merge signatures produced over slightly different byte strings by different replicas.

### Replica bookkeeping state (Section 4.2 "Bookkeeping variables" for Basic HotStuff; Section 6 "Data structures" for the event-driven version, these are two overlapping vocabularies for closely related state)

Basic HotStuff (Algorithm 2):
- `viewNumber : long` - starts at 1.
- `lockedQC : QuorumCertificate` - initially bottom; the highest QC this replica voted **commit** for (set at Algorithm 2 line 25).
- `prepareQC : QuorumCertificate` - initially bottom; the highest QC this replica voted **pre-commit** for.

Event-driven HotStuff (Section 6, Algorithm 4/5), this is the vocabulary to actually build against:
- `vheight : long` - height of the last node this replica voted for.
- `block : Node` - the locked node (role played by `lockedQC` above).
- `bexec : Node` - the last executed node.
- `qcHigh : QuorumCertificate` - highest known QC (role played by `prepareQC`/`genericQC`), owned by the Pacemaker.
- `bleaf : Node` - current leaf the Pacemaker is proposing on top of.
- `V : Map<NodeId, Set<Vote>>` - votes collected so far per node (Algorithm 4 line 22-26), used only by whoever is acting as leader/next-leader.
- Genesis constant `b0` known identically by all correct replicas, with `block = bexec = bleaf = b0` and `qcHigh` = the hardcoded genesis QC, at bootstrap (Section 6).

**Invariants:**
- `vheight` must only ever increase (Algorithm 4 line 17-19), this is the exact mechanism Appendix B.1's "Why monotonic vheight?" defends; relaxing it to "vote once per height" (instead of "vote only for strictly increasing heights") is shown by the paper's own worked example to break safety.
- `bexec` moves forward only via the recursive `onCommit` (Algorithm 4 lines 13-15), which must execute every intervening ancestor, skipping ancestors breaks the total-order guarantee (Theorem 8).

---

## 4. Key algorithms

### 4.1 `safeNode` predicate (Algorithm 1, lines 25-27), the single most important 3 lines in the paper
```
safeNode(node, qc):
    return (node extends from lockedQC.node)          // safety rule
        or (qc.viewNumber > lockedQC.viewNumber)       // liveness rule
```
Concrete translation: *"safety rule"* = walk `node`'s parent chain and confirm `lockedQC.node` appears on it. *"liveness rule"* = compare `qc.viewNumber` (the view of the QC justifying this proposal) against `lockedQC.viewNumber` numerically.

**Gotchas:**
- This predicate is only ever called *after* a separate, explicit check in Algorithm 2 line 9: `m.node extends from m.justify.node AND safeNode(m.node, m.justify)`, two conditions ANDed together, not one. It's easy to implement only the `safeNode` call and drop the "extends from its own justification" check, which is a distinct requirement (that `b.justify.node` is an ancestor of `b`, see Section 3 above).
- Both disjuncts read `lockedQC`, which starts as bottom. Decide up front what "extends from bottom" and "`qc.viewNumber` > bottom's viewNumber" evaluate to (the natural choice, always true before any lock is set, is what makes the very first proposal in the system acceptable).

### 4.2 Basic HotStuff, four phases per decision (Algorithm 2)

**prepare phase** (lines 2-10):
- Leader: wait for `n-f` `new-view` messages from the *previous* view; among their `justify` fields, take the one with the highest `viewNumber` as `highQC`; extend `highQC.node` with a new leaf carrying the client command; broadcast it as a `prepare` message carrying `highQC` as justification.
- Replica: on receiving the leader's `prepare` message, check `m.node extends from m.justify.node` **and** `safeNode(m.node, m.justify)`; if both hold, send a `prepare` vote (a partial signature over `<prepare, curView, m.node>`) back to the leader.
- *Gotcha:* "wait for `n-f` new-view messages" is an unbounded blocking wait in the pseudocode, in real code this must be gated by the view timeout (see 4.5 below, "Finally"/`nextView`), or a slow/absent replica hangs the leader forever.

**pre-commit phase** (lines 11-18):
- Leader: on `n-f` matching `prepare` votes, combine them into `prepareQC` via `QC(V)` and broadcast it in a `pre-commit` message.
- Replica: on receiving a `pre-commit` message whose `justify` QC matches type `prepare` and the current view, set local `prepareQC <- m.justify` and send a `pre-commit` vote (signing `<pre-commit, curView, m.justify.node>`).
- *Gotcha:* the vote signs `m.justify.node`, **not** `m.node` (which is bottom in this phase, per Algorithm 1's `Msg` construction), the payload being certified moved from the message body into the QC's embedded node reference. Easy to accidentally sign the wrong field.

**commit phase** (lines 19-26): structurally identical to pre-commit, one phase later. The one semantically critical line is **25**: `lockedQC <- m.justify`, this is *the* moment a replica becomes locked, and is what the entire safety proof (Theorem 2) hinges on. Get the ordering right: the replica must set `lockedQC` **before** sending its commit vote, not after, if a crash happens between sending the vote and persisting the lock, the replica could later be tricked into voting for a conflicting branch.

**decide phase** (lines 27-34): leader combines `n-f` commit votes into `commitQC` and broadcasts a `decide` message; a replica executes the committed branch's commands and responds to clients, then advances `viewNumber`.

**Finally / `nextView`** (lines 35-36): *any* "wait for" step in *any* phase above is abortable by a `nextView` timeout; on abort, the replica still sends a `new-view` message (carrying its current `prepareQC`) to the *next* view's leader before moving on. This is what lets the next leader learn what the stalled replica had certified.

### 4.3 Chained HotStuff (Algorithm 3), pipelining by giving every view one phase

The key idea: instead of 3 sequential phases per decision inside one view, every view runs exactly *one* "generic" phase, and a proposal's position in the chain retroactively plays the role of prepare/pre-commit/commit/decide for progressively older ancestors.

```
createLeaf(parent, cmd, qc):
    b.parent <- extend-with-blanks-from(parent, up-to-height = curView)   // "dummy nodes", see below
    b.cmd <- cmd
    b.justify <- qc
    return b
```
Per view, the leader:
1. Takes `highQC` from the previous view's collected messages; if `highQC.viewNumber > genericQC.viewNumber`, adopts it (`genericQC <- highQC`).
2. `createLeaf(genericQC.node, cmd, genericQC)`, broadcasts it as a `generic` message.

Per view, a replica receiving `b* = m.node`:
1. Walks back three `justify` links: `b00 = b*.justify.node`, `b0 = b00.justify.node`, `b = b0.justify.node`.
2. Votes if `safeNode(b*, b*.justify)`.
3. **One-Chain check** (line 17): if `b*.parent == b00` (i.e., the QC embedded in `b*` refers to its *direct* parent, not a distant ancestor), update `genericQC <- b*.justify`, this is "the prepare phase of `b00` succeeded."
4. **Two-Chain check** (line 19): if *additionally* `b00.parent == b0`, update `lockedQC <- b00.justify`, "the pre-commit phase of `b0` succeeded."
5. **Three-Chain check** (line 21): if *additionally* `b0.parent == b`, execute through `b`, "the commit phase of `b` succeeded," `b` is now committed.

**Dummy/blank nodes:** "The `genericQC` used by a leader in some view may not directly reference the proposal of the preceding view... To simplify the tree structure, `createLeaf` extends `genericQC.node` with blank nodes up to the height of the proposing view" (Section 5). *The paper never specifies what a blank node's fields actually are*, you must decide this yourself; the natural choice is a node with an empty command list and `justify = null` (or a sentinel), still hashed and chained normally so that height bookkeeping and ancestor-walks keep working uniformly. This is exactly what makes the "One/Two/Three-Chain" checks above sometimes fail (`b*.parent != b00`) even with no faults, a `prepare` phase failure or crash in a *prior* view shows up here as a broken chain, not as an exception.

### 4.4 Event-driven HotStuff (Algorithm 4 + Pacemaker, Algorithm 5), build against this one

```
onReceiveProposal(m = Msgv(generic, bnew, bottom)):
    if bnew.height > vheight and
       (bnew extends block or bnew.justify.node.height > block.height):
        vheight <- bnew.height
        send(getLeader(), voteMsgu(generic, bnew, bottom))
    update(bnew)

update(b*):
    b00 <- b*.justify.node; b0 <- b00.justify.node; b <- b0.justify.node
    updateQCHigh(b*.justify)                       // always safe to raise qcHigh
    if b0.height > block.height:
        block <- b0                                  // "commit phase" on b0 (i.e., lock)
    if b00.parent == b0 and b0.parent == b:
        onCommit(b); bexec <- b                      // "decide phase" on b

onCommit(b):
    if bexec.height < b.height:
        onCommit(b.parent); execute(b.cmd)          // recurse to catch up on skipped ancestors

onReceiveVote(m = voteMsgv(generic, b, bottom)):
    if exists a vote already recorded from v for b: return   // de-dup by sender identity
    V[b] <- V[b] union {(v, m.partialSig)}
    if |V[b]| >= n - f:
        qc <- QC({sigma | (_, sigma) in V[b]})
        updateQCHigh(qc)

onPropose(bleaf, cmd, qcHigh):
    bnew <- createLeaf(bleaf, cmd, qcHigh, bleaf.height + 1)
    broadcast(Msgu(generic, bnew, bottom))                // sent to ALL replicas, including self
    return bnew
```

**Notable relaxation vs. Chained HotStuff:** the `update` procedure's rule `if b0.height > block.height then block <- b0` locks using a **height comparison against any ancestor**, not a "direct parent" structural check, the paper explicitly says it "relax[es] the direct parent constraint for maintaining the highest `genericQC` and `lockedQC`, while still preserving the requirement that the QC in a valid node always refers to its ancestor" (Section 6). This is a real, deliberate loosening from Chained HotStuff's exact One/Two/Three-Chain test, it is proven safe separately in Appendix B, not by re-using Appendix A's proof. If you implement Chained HotStuff's exact structural `parent ==` checks inside the event-driven `update`, you have implemented a stricter-but-still-safe variant, not a bug, but know that you've deviated from Algorithm 4 as written.

**Pacemaker (Algorithm 5):**
```
getLeader()                              // UNSPECIFIED - "specified by the application"
updateQCHigh(qc'):
    if qc'.node.height > qcHigh.node.height:
        qcHigh <- qc'; bleaf <- qcHigh.node
onBeat(cmd):
    if self == getLeader(): bleaf <- onPropose(bleaf, cmd, qcHigh)
onNextSyncView():
    send Msg(new-view, bottom, qcHigh) to getLeader()
onReceiveNewView(Msg(new-view, bottom, qc')):
    updateQCHigh(qc')
```
The paper's comment directly above Algorithm 5 says the quiet part out loud: **"We assume Pacemaker in all correct replicas will have synchronized leadership after GST."** That's an assumption the algorithm rests on, not a mechanism it implements, see Section 6 of this guide.

### 4.5 Two-phase variant (Algorithm 6), build this deliberately, as a negative test, not as your target

```
update(b*):
    b0 <- b*.justify.node; b <- b0.justify.node
    updateQCHigh(b*.justify)
    if b0.height > block.height: block <- b0
    if b0.parent == b: onCommit(b); bexec <- b
```
Only `update` changes: a Two-Chain now suffices to commit, and a One-Chain sets the lock. The paper is explicit that this loses Optimistic Responsiveness and needs a synchronous-delay-based Pacemaker (like Tendermint/Casper) to remain live, Section 4.4's "Livelessness with two-phases" walks through exactly why. **This is worth building as a controlled experiment**, not skipping, see Section 7 below.

---

## 5. Implementation order

The paper's own presentation order (Basic -> Chained -> Event-driven) is close to a reasonable build order, but within that you want the deterministic, single-process safety logic working and adversarially tested *before* you introduce real concurrency, sockets, or timing. Suggested sequence for a solo/small-team Java project on a December deadline:

1. **Static data model.** `Node`, `QuorumCertificate`, `Message`, phase/type enums, and a `Crypto` interface (`sign`, `verify`, `combine`) with one concrete implementation backed by ordinary `java.security` keypairs per replica (see Section 6, do *not* try to build real threshold signatures first).
2. **In-memory single-JVM network.** A registry of `Replica` objects with an in-process message-dispatch queue (no sockets, no threads yet). This lets you drive the whole protocol deterministically from a test, one message at a time.
3. **`safeNode` + one manually-driven view of Basic HotStuff**, 4 replicas (`f=1`), no faults: leader collects new-view, proposes, all four phases execute, one command commits identically at all replicas. This is your first real milestone, "it works on the happy path."
4. **Multi-view loop, still all-honest, still single-process.** Drive several sequential commands through and check every replica's committed log matches.
5. **Byzantine test harness.** Add the ability to make a `Replica` object equivocate (send different proposals to different replica subsets in the same view), stay silent, or send garbage signatures. Verify Theorem 2 (no two conflicting nodes both commit) under every `f`-bounded fault pattern you can construct, this is where most of the real bugs live, and it's far easier to find them before you add networking and timing on top.
6. **`nextView`/timeout + minimal Pacemaker** (rotate leader deterministically by view number, exponential backoff, Section 4.4's suggested scheme), still single-process, but now simulate a "stuck" leader by having its `Replica` object stop responding, and confirm the system recovers.
7. **Chained HotStuff (Algorithm 3).** Pipeline the phases; implement dummy/blank-node extension and the One/Two/Three-Chain checks. Verify pipelined throughput improves over Basic HotStuff on the same in-memory network.
8. **Event-driven refactor (Algorithm 4/5).** This is the natural point to introduce real concurrency, one thread (or async event loop) per replica reacting to inbound messages, since the callback style (`onReceiveProposal`, `onReceiveVote`, etc.) assumes asynchronous arrival, not a single-threaded round-robin loop.
9. **Real network transport.** Replace the in-memory dispatcher with actual sockets (Java NIO, or a small RPC layer) with authenticated point-to-point channels (Section 3's network assumption), e.g., TLS mutual auth per link, or application-level message signing over plain TCP.
10. **Client protocol.** Request numbering and de-duplication (explicitly out of scope in the paper, Section 4), batching multiple commands into one `Node`, and delivering responses once `f+1` matching replies exist.
11. **Persistence.** Durably write `block`/`lockedQC`, `vheight`, and the execution log before they're used to justify a vote or an execution, so a crash-and-restart can't cause equivocation (not discussed anywhere in the paper, pure implementation necessity).
12. **Catch-up/sync protocol.** A replica that's missing an ancestor node fetches it from peers before delivering any message that depends on it (Section 4.2, explicitly deferred by the paper).
13. **Stretch goals, if time remains:** a real threshold signature scheme (BLS) in place of a signature list; reproduce the Section 8.4 view-change authenticator-count measurement across increasing `n`; benchmark throughput/latency the way Section 8.2-8.3 does.

---

## 6. What the paper glosses over

- **The prototype itself doesn't use real threshold signatures, despite Section 3 presenting them as core to the design.** Section 8.1 states plainly: "Our prototype implementation of HotStuff uses secp256k1 for all digital signatures in both votes and quorum certificates." That means their QC is, in practice, a *list* of `n-f` individually verifiable ECDSA signatures, not a single combined threshold signature. Section 8.3 admits the gap directly: "This is because we currently still use a list of secp256k1 signatures for a QC. In the future, we plan to reduce the cryptographic computation overhead... by using a fast threshold signature scheme." **For your project, follow the authors' own actual choice, not their Section 3 abstraction**: make `QuorumCertificate.signature` a list of `(replicaId, signature)` pairs verified individually. This does not change any of the `O(n)` vs `O(n^2)`/`O(n^3)` *authenticator-count* comparisons in Table 1 (those count authenticators, not bytes), but it does mean your QC is `O(n)` in size, not constant-size, don't claim the constant-size benefit unless you actually implement a real threshold scheme.
- **Double-voting prevention is asserted, not enforced, in the Basic HotStuff pseudocode.** Lemma 1's entire proof rests on "the pseudocode allows voting only once for each phase in each view," but Algorithm 2 never shows an explicit guard for this; it's an artifact of the sequential, single-pass-per-view control flow as written. The moment you implement this with concurrent message handlers (which you will, by the event-driven stage), you must add an explicit check yourself. The event-driven version (Algorithm 4) *does* show one concrete mechanism for this, the `vheight` monotonicity check, make sure it, or something equivalent, is present before you trust any vote-collection logic.
- **Vote de-duplication by the collector is shown, but only against exact duplicate messages, not against a malicious replica submitting distinct valid-looking signatures.** Algorithm 4's `onReceiveVote` de-dupes by sender identity, which is correct, but make sure your vote map is keyed by replica identity, not by signature bytes; a Byzantine replica could otherwise resubmit slightly different (but both valid) partial signatures to try to be counted twice.
- **Message/vote verification is explicitly waved away.** Section 3: "we will typically leave invocations of `tverify` implicit in our protocol descriptions." Every single "wait for message" step in Algorithms 2-4 silently assumes you have already checked the signature before counting the message. There is no explicit `verify()` call anywhere in the pseudocode listings, you must add one at every ingestion point (both partial-signature votes and combined-QC justifications), or a single Byzantine replica can inject fabricated messages that get silently counted.
- **The catch-up/sync protocol is explicitly deferred, not designed.** Section 4.2: "a replica delivers a message only after the branch led by the node is already in its local tree. In practice, a recipient who falls behind can catch up by fetching missing nodes from other replicas. For brevity, these details are also omitted from the pseudocode." This is a real subsystem (request-missing-ancestor, verify-on-receipt, insert-into-tree) that you have to design from scratch; the paper gives you nothing beyond "it's possible."
- **View synchronization ("Pacemaker synchronization") is assumed, not solved.** The comment directly above Algorithm 5 states: "We assume Pacemaker in all correct replicas will have synchronized leadership after GST." Section 6's own text hedges further: the usual mechanism is "for replicas to increase the count of Delta's they spend at larger heights, until progress is being made," vague by the paper's own admission. Section 4.4 offers exponential backoff as "a possible solution," with **no concrete initial timeout, growth factor, or cap**, you must choose these, and the choice materially affects both recovery time after a real leader crash and how often you get needless view-changes under normal jitter.
- **`getLeader()` is completely unspecified** (Algorithm 5: "specified by the application"). Section 4.4's only suggestion is "some deterministic mapping from view number to a replica, eventually rotating through all replicas," implicit in this is that *all correct replicas must compute the same mapping*, which is never stated as a requirement to check, only assumed.
- **Client request semantics are explicitly out of scope.** Section 4's opening paragraph: "For the most part, we omit the client from the discussion, and defer to the standard literature for issues regarding numbering and de-duplication of client requests." You need your own design here (a common approach, borrowed from PBFT: client signs `(command, sequence#)`, replica caches the last executed sequence number and response per client to answer retransmissions without re-executing).
- **Batching is an experimental parameter, not part of the algorithm.** The only algorithmic mention is "a proposed command (or a batch of them)" (Section 4). Batch-size/batch-timeout tradeoffs are discussed only in Section 8.2 as an empirical evaluation knob, how and when to close a batch is left entirely to you.
- **Blank/dummy node contents are undefined.** Section 5 introduces them ("blank nodes up to the height of the proposing view") to keep height equal to view number, but never specifies their fields. Decide a concrete representation (e.g., empty command list, `justify = null`) and make sure it still hashes and chains like a normal node, since your One/Two/Three-Chain checks depend on ordinary parent-pointer walks working uniformly across real and blank nodes.
- **Persistence/crash-recovery is never mentioned anywhere in the paper.** `lockedQC`/`block`, `vheight`, and `bexec` are safety-critical, if a replica crashes and restarts having forgotten its lock, it can be tricked into voting for a branch conflicting with one it already helped commit. This is a standard BFT engineering concern the paper simply doesn't address; you must persist these fields before acting on them, not after.
- **Genesis bootstrap is a special case that breaks the normal verification path.** Section 6: `b0` "contains a hard-coded QC for itself," this QC cannot be checked with the normal `tverify` (there is no real quorum that ever signed it), so your verification logic needs an explicit, well-known exception for the genesis QC.
- **Two-phase HotStuff's livelock (Section 4.4, "Livelessness with two-phases") is handed to you as a fully worked adversarial scenario**, the paper spells out an infinitely repeating single-view transcript where a lone locked replica blocks progress forever. This is not a gap so much as a gift: it is a ready-made regression test (see Section 7).
- **Appendix B.1's two "Remarks" are the paper explaining its own near-misses.** "Why monotonic `vheight`?" and "Why direct parent?" each construct a specific interleaving that breaks safety if you relax the stated rule even slightly (e.g., "vote once per height" instead of "vote for strictly increasing heights"; or transitive-ancestor instead of direct-parent checks in the *Chained* HotStuff commit rule specifically, note Algorithm 4's event-driven `update` deliberately *does* relax the direct-parent check in a different, separately-proven-safe way, so don't confuse the two). These are the two most concrete gotchas the paper hands you, and both make excellent unit tests (see below).

---

## 7. Verification strategy

**Structural/invariant tests (Section 3 above):**
- Construct a scenario where `n-f` votes exist for two different nodes in the same `(type, viewNumber)`, confirm your vote-collection code (keyed by replica identity) never allows this to happen in the first place, because no single replica contributes to both; this directly exercises Lemma 1's quorum-intersection argument.
- Feed your `QC()`/combine function a vote set containing two entries from the same replica identity (simulating a Byzantine replica trying to be double-counted) and confirm the combiner treats it as one vote, not two.
- Attempt to build a QC from `n-f-1` votes and confirm it's rejected, off-by-one here silently breaks the entire quorum-intersection safety argument.

**Byzantine-fault fault-injection (Section 6 above, first three bullets):**
- **Equivocating leader:** have the leader send node `A` to replica subset `{1,2,3,...}` and conflicting node `B` to a different subset in the same view; confirm neither ever accumulates `n-f` votes and neither commits (this is Theorem 2's core claim, exercised directly rather than just proven on paper).
- **Multi-view interleaving matching Theorem 2's own proof construction:** build the "first switching point" scenario the proof walks through, a replica locked on `w` at view `v1`, a conflicting `b` proposed and QC'd later at `v2 > v1`, and confirm the locked replica's `safeNode` check correctly refuses to vote unless the liveness disjunct (a strictly higher-view QC) legitimately overrides the stale lock.
- **The paper's own two-phase livelock (Section 4.4):** implement the exact adversarial schedule described there against your deliberately-built two-phase variant (Algorithm 6) and confirm it never commits within any bounded number of views; then run the identical schedule against your real three-phase implementation and confirm it does commit. This side-by-side is one of the strongest correctness demonstrations available to you, and the paper wrote the test case for you.
- **Appendix B.1's two remarks, as literal unit tests:** reproduce the "vote for `b`, then learn of `w0`/`w00` and vote for `w00` without having seen `b0`/`b00` yet, then later learn of `b00`" interleaving, and confirm your monotonic-`vheight` check rejects the second vote that the relaxed rule would have allowed. Separately, reproduce the height ordering `w0.height < b.height < b0.height < w00.height < b00.height` from the "why direct parent" remark and confirm your commit-path ancestor check rejects the unsafe commit.
- **Message duplication/replay:** deliver the same vote twice (simulating network duplication) and confirm the collector doesn't double-count it; deliver a vote signed over a mismatched `(type, view, node)` triple and confirm verification rejects it outright.
- **Genesis edge case:** cold-start a cluster and confirm the hardcoded genesis QC is accepted via the special-cased bootstrap path (not general `tverify`), and that the very first real proposal correctly chains onto it.
- **Crash/restart:** kill a replica mid-phase (after it has sent a commit vote but before the decide message arrives) and restart it from persisted state; confirm it neither re-votes inconsistently nor loses its lock.

**Liveness/progress tests (Theorem 4):**
- Under simulated post-GST conditions (bounded, small message delay, no further faults) with a correct current leader, confirm a decision commits within a bounded number of views, this is checking the theorem's actual claim, not merely "the demo doesn't hang."
- Kill or silence the current leader and confirm the system progresses to a new, correct leader and resumes committing within a bounded number of `nextView` timeouts, tune your backoff parameters against this test, since the paper gives you no default.

**Scale/complexity verification (mirrors Section 8.4's methodology directly):**
- Instrument authenticator counts (partial signatures + combined signatures actually received per replica) across a leader replacement, and confirm the count stays `O(n)` and constant regardless of whether the leader changed, this is literally re-running the paper's own measurement methodology (Figure 9/10) and is the most direct empirical check of the paper's central claim (Table 1) that your implementation embodies it, rather than merely "being a working consensus protocol" in some other, non-linear way.
````

</details>
