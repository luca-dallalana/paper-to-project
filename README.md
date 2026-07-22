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
