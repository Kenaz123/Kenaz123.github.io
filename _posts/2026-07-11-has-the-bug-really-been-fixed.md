---
layout: post
title: "Paper Notes: Has the Bug Really Been Fixed?"
date: 2026-07-11 09:00:00-0500
description: Reading Gu, Barr, Hamilton, and Su's ICSE'10 paper on detecting bad bug fixes with distance-bounded weakest preconditions.
tags: [Software Engineering]
related_posts: false
---

Every developer has done this: fix a bug, rerun the failing test, watch it go green, and move on to the next ticket. [_Has the Bug Really Been Fixed?_](https://people.inf.ethz.ch/suz/publications/icse10-badfix.pdf) (Gu, Barr, Hamilton, and Su, ICSE 2010) is about how often that green checkmark is lying to you, and it backs the claim with a fairly clean piece of formalism.

### The setup

Mining the Bugzilla histories of Ant, AspectJ, and Rhino, the paper finds that bad fixes account for 66-80% of reopened bugs across the three projects, and roughly 9% of _all_ bugs in the Apache-project sample turn out to be bad fixes once you trace the history. Developers even confess to it in the comments — the paper quotes one saying "Oops, missed one code path."

### Formalizing "bad"

Model a program as a function $$P : I \to O$$ from inputs to outputs, and model a bug $$b$$ as the violation of an assertion $$\varphi$$. Given a known bug-triggering input $$i_b$$, the buggy program satisfies

$$P_b(i_b) \not\models \varphi.$$

Of course $$i_b$$ is rarely the only input that triggers $$b$$. The full **bug-triggering input domain** is

$$\tilde{i}_b = \{\, i \in I : P_b(i) \models \neg\varphi \,\}.$$

A fix $$f$$ produces a new program $$P_f$$. At minimum $$P_f(i_b) \models \varphi$$ — the fix silences the one input you tested — but $$P_f$$ may not silence all of $$\tilde{i}_b$$. The subset it actually handles is

$$\hat{i}_b = \{\, i \in \tilde{i}_b : P_f(i) \models \varphi \,\},$$

and **coverage** is just how close $$\hat{i}_b$$ gets to $$\tilde{i}_b$$. On the other side, letting $$P^o$$ denote the (unknown, ideal) correct oracle, **disruption** is the set of _new_ deviations the fix introduces outside the bug's own input domain:

$$B_f = \{\, i \in I \setminus \tilde{i}_b : P_f(i) \neq P^o(i) \,\}.$$

An ideal fix satisfies both at once:

$$\hat{i}_b = \tilde{i}_b \;\wedge\; B_f = \emptyset,$$

and the two sets give you a genuine partial order over competing fixes for the same bug — $$f_a$$ is strictly better than $$f_b$$ iff

$$cov(f_b) \subseteq cov(f_a) \;\wedge\; B_{f_a} \subseteq B_{f_b}.$$

I like this framing because it turns "did we fix it?" from a vibe into a question with a falsifiable answer: find one $$i \in \tilde{i}_b \setminus \hat{i}_b$$ and you've disproven the fix.

### Why you can't just compute the weakest precondition

Deciding coverage looks like a job for Dijkstra's weakest precondition (WP): symbolically walk $$P_f$$ backward from $$\varphi$$ and check whether the resulting precondition still overlaps $$\tilde{i}_b$$. Two things make full WP impractical here — it needs loop invariants, which are hard to infer automatically, and the number of paths it has to consider grows exponentially in the number of branches.

The paper's answer is **distance-bounded weakest precondition**, $$WP_d$$:

$$WP_d : \text{Programs} \times \text{Predicates} \times \text{Paths} \times \mathbb{N}_0 \to \text{Predicates}.$$

Instead of computing WP over every path, restrict it to paths near one distinguished path $$\Pi$$ — concretely, the path the known bug-triggering input $$i_b$$ actually took. Every path is encoded as a string of control-flow-graph edge labels, and path-similarity is Levenshtein edit distance $$\Delta$$ between those strings. Fix a budget $$d$$ and only keep paths within it:

$$C = \{\, s \in \text{Paths} : \Delta(s, \Pi) \le d \,\}.$$

$$WP_d$$ is then just the disjunction of the ordinary weakest precondition over that candidate set:

$$WP_d(P, \varphi, \Pi, d) = \bigvee_{c \in C} WP(c, \varphi).$$

Two nice properties fall out immediately. First, $$WP_d$$ never needs a loop invariant — loops are unrolled into an infinite CFG and truncated by the same distance budget, so "how many times does this loop run" is answered by edit distance, not by inference. Second, it's a strict generalization of WP: at $$d = 0$$ you get the WP of the single concrete path, and

$$\lim_{d \to \infty} WP_d(P, \varphi, \Pi, d) = WP(P, \varphi).$$

The bet underneath all of this is empirical, not just computational convenience: Kim et al. showed that bugs cluster spatially in a codebase (temporal/lexical locality), so paths that are edit-distance-close to a known buggy path are disproportionately likely to matter for the _same_ bug. Small $$d$$ should already buy you most of the useful predicate before path counts blow up.

### Putting it together

Checking coverage of a fix $$f$$ reduces to a three-step pipeline, which the paper compresses into one formula:

$$\exists x_1 \cdots x_n \Big[\, SE\big(P_f,\; \underbrace{WP_d(P_b, \neg\varphi, \Pi_{i_b}, d)}_{\alpha} \,\big) \wedge \varphi \,\Big]$$

1. Extract the concrete path $$\Pi_{i_b}$$ that the known buggy input induces.
2. Compute $$\alpha = WP_d(P_b, \neg\varphi, \Pi_{i_b}, d)$$ — an _under-approximation_ of $$\tilde{i}_b$$.
3. Symbolically execute $$P_f$$ from precondition $$\alpha$$ to get a postcondition $$\psi$$, existentially quantify away the non-input variables, and check whether $$\psi \to \varphi$$ is valid.

Any counterexample to that implication is, by construction, a member of $$\alpha$$'s underlying input set — and since $$\{i \in I : \alpha\} \subseteq \tilde{i}_b$$ by construction, every counterexample $$WP_d$$ produces is a genuine bug-triggering input, not a false positive. That's the soundness guarantee that makes the whole approach trustworthy as a bug-fix critic rather than just a heuristic: FIXATION can miss bad fixes (if $$d$$ is too small to reach the relevant path), but it never cries wolf.
