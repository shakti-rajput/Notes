# Project Interview Preparation Guide
**Shakti Rajput — 7 Projects, Detailed Explanations & Q&A**

---

## How to Use This Guide

Each project has:
1. **The 30-second version** — your opening answer
2. **The deep explanation** — with worked examples where the algorithm allows
3. **Likely questions & answers** — what interviewers actually probe

A general rule: open with the *problem*, then the *approach*, then the *result*. Don't lead with technology names.

---

# 1. Graph Query Algorithm — 163× Faster than PostgreSQL
*M.Sc. Research, University of Freiburg · C++/Python · Sep 2023*

> ⚠️ **FILL-IN REQUIRED**: I have your results (6.7s vs PostgreSQL 1,095s vs SPARQL 12s on 10M records) but not the internal mechanism you designed. The framework below is correct; **you must fill in the actual data structure and traversal strategy from your thesis** before using this in an interview. I've marked those spots clearly.

## The 30-Second Version

"This came out of my M.Sc. thesis. The problem: graph-shaped queries — things like 'find all papers cited by papers that cite X' — are genuinely slow in relational databases, because every hop in the graph becomes another JOIN, and the query planner doesn't know it's traversing a graph. I designed a custom algorithm and data structure specifically for this traversal pattern, benchmarked it on 10 million records, and got it down from PostgreSQL's 1,095 seconds to 6.7 seconds — about 163 times faster. SPARQL, which is purpose-built for graph queries, came in at 12 seconds, so I also beat the specialized tool by roughly 2×."

## Why Relational Databases Are Slow Here — The Core Insight

This is the part you should be able to explain cleanly, because it's the *motivation* for everything else.

Suppose you have a citations table:

| citing_paper | cited_paper |
|---|---|
| A | B |
| B | C |
| C | D |

A 3-hop query ("what does A reach in 3 steps?") in SQL looks like:

```sql
SELECT c3.cited_paper
FROM citations c1
JOIN citations c2 ON c1.cited_paper = c2.citing_paper
JOIN citations c3 ON c2.cited_paper = c3.citing_paper
WHERE c1.citing_paper = 'A';
```

**Why this degrades badly:**
- Each hop = one more self-join. A 5-hop query = 5 joins on a 10M-row table.
- The query planner estimates intermediate result sizes, and on graph data those estimates are usually *wrong* — real graphs have hub nodes (a paper cited 50,000 times), so the planner expects 100 intermediate rows and gets 500,000.
- Bad estimates → wrong join strategy (hash join vs nested loop) → catastrophic slowdown.
- Each join materializes an intermediate result set, so you pay memory and I/O at every hop.

**The key sentence to say out loud:** *"Relational engines are optimized for set operations on tables, not for pointer-chasing through a graph. Every hop costs you a join, and the optimizer's cardinality estimates break down on graph-shaped data because of degree skew."*

## YOUR ALGORITHM — Fill This In

Be ready to answer these three questions about your actual implementation:

**(a) What data structure did you build?**
Common approaches (identify which was yours):
- *Adjacency list in memory* — each node maps to a vector of neighbor IDs; traversal is pointer-following, not joins
- *CSR (Compressed Sparse Row)* — two flat arrays: an offsets array and a neighbors array. Extremely cache-friendly, which matters enormously at 10M scale
- *Index over a specific traversal pattern* — precomputed reachability or a path index
- *Bitmap/bitset representation* — set operations become bitwise AND/OR

**(b) What made it fast?** (the mechanism, not the result)
Likely candidates:
- No join overhead — traversal is direct memory access
- Cache locality — contiguous arrays vs. scattered B-tree pages
- No intermediate materialization — stream results instead of building temp tables
- Avoided the query planner entirely — you knew the access pattern, so no estimation needed

**(c) What's the complexity?**
- BFS/DFS traversal: **O(V + E)** for a full traversal, or **O(visited nodes + their edges)** for a bounded query
- PostgreSQL's join approach: effectively **O(n^k)** in the worst case for k hops without good indexes

## Likely Questions & Answers

**Q: "163× sounds too good. What's the catch?"**
> "Fair question — it's not free. The main trade-off is that I'm building a purpose-specific structure, so I lose everything a general-purpose database gives you: ACID transactions, arbitrary ad-hoc queries, concurrent writes, durability. PostgreSQL is solving a much harder, more general problem. My algorithm is fast *because* it's specialized — it only does this one traversal pattern well. Also, my structure needs to be built up front, which costs time and memory that the benchmark's query time doesn't reflect."

**Q: "Did you tune PostgreSQL, or is this an unfair comparison?"**
> [BE HONEST HERE] "I [added indexes on the join columns / used the default configuration]. It's a fair point that a heavily tuned Postgres with the right indexes and work_mem settings would close some of the gap. That's part of why I included SPARQL in the benchmark — it's a purpose-built graph engine, so it's the more meaningful comparison, and I was about 2× faster there."

**Q: "Why was SPARQL 12s and yours 6.7s? You beat a specialized tool — how?"**
> [FILL IN — likely answers: "SPARQL engines are general-purpose over RDF triples and pay overhead for that generality; I could specialize to my exact query shape and data layout."]

**Q: "How would you make this production-ready?"**
> "Right now it's a research artifact — single-threaded, read-only, builds the whole structure in memory. For production you'd need incremental updates rather than full rebuilds, persistence so you don't rebuild on restart, and concurrency control. At that point you'd honestly be building a graph database, and you'd want to evaluate Neo4j or similar before building it yourself."

**Q: "Walk me through the algorithm on a small example."**
> [Prepare a 5-node example graph and trace your algorithm step by step. This WILL be asked — practice drawing it.]

---

# 2. Research Paper Recommendation System (PySpark)
*Personal Project · Python/PySpark · 2023*

## The 30-Second Version

"It's a recommendation system for research papers, built on PySpark so it processes data at scale rather than in memory. I implemented two complementary approaches — collaborative filtering, which recommends based on what similar users read, and content-based filtering, which recommends based on the actual text of the papers — then evaluated both offline with MRR, precision, and recall."

## Part A: Collaborative Filtering with ALS — Worked Example

**The setup.** You have a user-item matrix of who has which papers in their library:

|  | Paper1 | Paper2 | Paper3 | Paper4 |
|---|---|---|---|---|
| **Alice** | 1 | 1 | ? | ? |
| **Bob** | 1 | ? | 1 | ? |
| **Carol** | ? | 1 | 1 | 1 |

The `?` cells are what you want to predict. **The matrix is ~99% empty** — this is the sparsity problem, and it's the first thing I analyzed.

**The idea behind matrix factorization.** Instead of storing the full (huge, mostly empty) matrix, approximate it as the product of two much smaller "factor" matrices:

```
R  ≈  U × Pᵀ

R = users × papers        (e.g. 10,000 × 50,000 — huge, sparse)
U = users × k factors     (e.g. 10,000 × 10 — small, dense)
P = papers × k factors    (e.g. 50,000 × 10 — small, dense)
```

Each user becomes a vector of *k* latent factors (say k=10). Each paper becomes a vector in the same space. A prediction is just a dot product.

**Concretely:** suppose after training, with k=2, the learned factors are:

```
Alice   = [0.9, 0.1]     (loves ML, not interested in databases)
Bob     = [0.8, 0.3]
Carol   = [0.2, 0.9]     (loves databases, not ML)

Paper1  = [0.9, 0.1]     (an ML paper)
Paper4  = [0.1, 0.9]     (a databases paper)
```

Predicting whether Alice would want Paper4:
```
score(Alice, Paper4) = 0.9×0.1 + 0.1×0.9 = 0.09 + 0.09 = 0.18   → low, don't recommend
score(Carol, Paper4) = 0.2×0.1 + 0.9×0.9 = 0.02 + 0.81 = 0.83   → high, recommend
```

Nobody told the model what "ML" or "databases" means — the factors emerge from the co-occurrence patterns in the data. That's the elegant part.

**Why "Alternating" Least Squares.** Solving for both U and P at once is a non-convex problem — hard. But notice: **if you fix P, solving for U is just ordinary least squares** (a convex problem with a closed-form solution). And vice versa. So ALS alternates:

```
1. Fix P, solve for U        (least squares — exact solution)
2. Fix U, solve for P        (least squares — exact solution)
3. Repeat until it converges
```

**Why this matters for Spark specifically:** when P is fixed, every user's factor vector can be computed completely independently of every other user's. That's embarrassingly parallel — perfect for distributing across a cluster. This is exactly why ALS is the algorithm Spark's MLlib ships with, rather than, say, SGD-based factorization.

## Part B: Content-Based Filtering — Worked Example

Collaborative filtering has a fatal flaw: **a brand-new paper that nobody has read yet can never be recommended** (the cold-start problem). Content-based filtering fixes this by looking at the paper's actual text.

**Step 1 — TF-IDF.** Turn each paper's text into a vector of weighted terms.

- **TF (Term Frequency):** how often does this word appear in *this* document?
- **IDF (Inverse Document Frequency):** how rare is this word across *all* documents? `log(total_docs / docs_containing_word)`

Multiply them. The effect: words that are frequent *here* but rare *overall* get high weight.

*Example:* the word "the" appears constantly in every paper → IDF ≈ log(10000/10000) = 0 → weight zero, correctly ignored. The word "transformer" appears 15 times in this paper but only in 200 of 10,000 papers → IDF = log(10000/200) = 3.9 → high weight, a genuinely distinguishing term.

**Step 2 — LDA for topic modelling.** TF-IDF gives you thousands of dimensions (one per vocabulary word). LDA compresses that into a handful of interpretable *topics*. Each paper becomes a distribution over topics:

```
Paper_A = {Topic1: 0.7, Topic2: 0.2, Topic3: 0.1}
```
where Topic1 might turn out to be ~"neural networks, training, gradient" and Topic3 ~"query, index, relational".

**Step 3 — Build a user profile.** Average the topic vectors of everything in the user's library:

```
Alice's library = [Paper_A, Paper_B, Paper_C]
Alice's profile = mean(topics of A, B, C) = {Topic1: 0.8, Topic2: 0.15, Topic3: 0.05}
```

**Step 4 — Cosine similarity.** Score candidate papers against the profile:

```
cos(θ) = (A · B) / (||A|| × ||B||)
```

*Worked example:*
```
Alice profile   = [0.8, 0.15, 0.05]
Candidate paper = [0.75, 0.2, 0.05]

Dot product = (0.8×0.75) + (0.15×0.2) + (0.05×0.05) = 0.6 + 0.03 + 0.0025 = 0.6325
||Alice||   = √(0.64 + 0.0225 + 0.0025) = √0.665 = 0.8155
||Paper||   = √(0.5625 + 0.04 + 0.0025) = √0.605 = 0.7778

cosine = 0.6325 / (0.8155 × 0.7778) = 0.6325 / 0.6343 = 0.997   → near-perfect match
```

**Why cosine and not Euclidean distance?** Cosine measures the *angle* between vectors, ignoring magnitude. A user with 200 papers and a user with 5 papers can have the same interests — cosine treats them as similar; Euclidean distance would say they're far apart just because one vector is longer.

## Part C: Evaluation

- **MRR (Mean Reciprocal Rank):** if the first genuinely relevant item appears at position 3, you score 1/3. Averaged across users. Rewards putting the right answer *near the top*, not just somewhere in the list.
- **Precision:** of the 10 things I recommended, how many were actually relevant?
- **Recall:** of all the things that were relevant, how many did I surface?

The precision/recall trade-off: recommend more items → recall goes up, precision goes down.

## Likely Questions & Answers

**Q: "Why did you build both approaches instead of just one?"**
> "They fail in opposite situations. Collaborative filtering can't handle a new paper nobody's read yet — it has no interaction data to work with. Content-based filtering handles that fine because it reads the text. But content-based filtering tends to be repetitive; it'll keep recommending things very similar to what you've already read, and it can't surface a genuinely surprising recommendation the way collaborative filtering can, where it discovers 'people like you also read this' across topic boundaries."

**Q: "How did you handle sparsity?"**
> "First I measured it — I did a sparsity and rank-frequency analysis on the user-library data, because how you approach the problem depends on how bad it is. Matrix factorization is itself the main answer to sparsity: instead of trying to fill in a 99%-empty matrix directly, you reduce it to low-dimensional dense factors, where every observed interaction contributes to learning the factors. ALS also uses regularization to prevent overfitting on users with very few papers."

**Q: "What is k, the number of latent factors, and how do you choose it?"**
> "It's the dimensionality of the latent space — how many hidden dimensions you're using to describe users and papers. Too small and the model can't capture real distinctions; too large and it overfits and gets slow. You pick it empirically by evaluating on a held-out test set — I used a train/test split with RMSE."

**Q: "Why PySpark? Couldn't you do this in pandas?"**
> "For the dataset I had, pandas would technically work. I used Spark because the point was to build it in a way that *scales* — RDD and DataFrame operations distribute across a cluster, and ALS in particular parallelizes naturally, since each user's factors can be solved independently when the item factors are fixed. It's the difference between a solution that works on my laptop and one that works on a real corpus."

**Q: "What's the difference between an RDD and a DataFrame?"**
> "RDDs are the lower-level API — distributed collections of objects, where you control the transformations explicitly and Spark doesn't know anything about the structure of your data. DataFrames add a schema, which lets Catalyst, Spark's query optimizer, actually optimize your execution plan — things like predicate pushdown and column pruning. In practice you use DataFrames unless you need fine-grained control, which is why I used both depending on the stage."

---

# 3. Movie Search Engine
*Personal Project · Python · Feb 2022*

## The 30-Second Version

"It's a fuzzy search engine for movie titles — built from scratch rather than using an existing search library, because the point was to understand how the algorithms actually work. The core problem: if someone types 'Shawshenk Redemtion', you still need to return 'The Shawshank Redemption'. I used Q-Gram indexing to narrow down candidates quickly, then Prefix Edit Distance to rank them."

## Part A: Q-Gram Indexing — Worked Example

**The problem it solves.** You have 100,000 movie titles. A naive fuzzy search compares the query against every single one — 100,000 expensive edit-distance calculations per keystroke. Way too slow for a search-as-you-type interface.

**The idea.** Break every string into overlapping substrings of length *q* (say q=3, "trigrams"), and build an inverted index from trigram → titles containing it.

**Indexing "SHREK":**

First, pad the string so that prefixes and suffixes are represented properly:
```
$$SHREK$$
```
Then slide a 3-character window:
```
$$S, $SH, SHR, HRE, REK, EK$, K$$
```

Build the index across all titles:
```
"SHR"  →  [Shrek, Shrek 2, Shredder, ...]
"HRE"  →  [Shrek, Shrek 2, Shredder, ...]
"REK"  →  [Shrek, Shrek 2, ...]
"EK$"  →  [Shrek, ...]
```

**Querying "SHRECK" (misspelled):**
```
Trigrams: $$S, $SH, SHR, HRE, REC, ECK, CK$, K$$
```

Look each one up and count how many trigrams each candidate shares:
```
Shrek     → matches $$S, $SH, SHR, HRE       = 4 shared trigrams
Shrek 2   → matches $$S, $SH, SHR, HRE       = 4 shared trigrams
Shredder  → matches $$S, $SH, SHR, HRE       = 4 shared trigrams
Titanic   → matches nothing                  = 0 shared trigrams
```

**The payoff:** instead of 100,000 edit-distance computations, you now have maybe 20 candidates, and you only run the expensive comparison on those. Titanic was never even considered.

**The key insight to articulate:** *"Q-gram indexing is a cheap filter, not the final answer. It's designed to be fast and over-inclusive — it produces a small candidate set that definitely contains the right answer, and then you pay for accuracy only on that small set."*

**Why the padding matters:** without `$$`, a query for "SHREK" and a title "ASHREKB" would look similar in the middle. Padding makes the beginning and end of the string explicit, so prefix matches get properly weighted — which matters a lot for search-as-you-type.

## Part B: Prefix Edit Distance — Worked Example

**Standard edit distance (Levenshtein)** = minimum number of insertions, deletions, or substitutions to turn string A into string B.

Classic DP table for "KITTEN" → "SITTING":

```
        ""  S   I   T   T   I   N   G
    ""   0  1   2   3   4   5   6   7
    K    1  1   2   3   4   5   6   7
    I    2  2   1   2   3   4   5   6
    T    3  3   2   1   2   3   4   5
    T    4  4   3   2   1   2   3   4
    E    5  5   4   3   2   2   3   4
    N    6  6   5   4   3   3   2   3
```
Answer: **3** (K→S, E→I, insert G).

The recurrence:
```
d[i][j] = min(
    d[i-1][j]   + 1,                        # deletion
    d[i][j-1]   + 1,                        # insertion
    d[i-1][j-1] + (0 if A[i]==B[j] else 1)  # match or substitution
)
```

**Why plain edit distance is WRONG for search-as-you-type.** The user has typed "SHREK" and the real title is "SHREK FOREVER AFTER". Standard edit distance says the cost is 14 — you'd have to insert 14 characters. It would rank that title as a terrible match, when actually it's exactly what the user is looking for.

**Prefix Edit Distance fixes this with one change:** instead of taking the value in the bottom-right corner of the table, **take the minimum value across the entire last row.**

```
Query: "SHREK"        Title: "SHREK FOREVER AFTER"

                S  H  R  E  K  ␣  F  O  R  E  V  E  R ...
         ""  0  1  2  3  4  5  6  7  8  9 10 11 12 13
         S   1  0  1  2  3  4  5  6  7  8  9 10 11 12
         H   2  1  0  1  2  3  4  5  6  7  8  9 10 11
         R   3  2  1  0  1  2  3  4  5  6  7  8  9 10
         E   4  3  2  1  0  1  2  3  4  5  6  7  8  9
         K   5  4  3  2  1  0  1  2  3  4  5  6  7  8
                              ↑
                        minimum of last row = 0
```

**Prefix edit distance = 0.** The query is a perfect prefix of the title. Exactly the ranking behavior you want.

**The one-sentence explanation:** *"Standard edit distance asks 'how different are these two strings?' Prefix edit distance asks 'how well does this query match the beginning of this string?' — which is the actual question in a search box."*

## Likely Questions & Answers

**Q: "What's the complexity of edit distance, and is that a problem?"**
> "It's O(m×n) in both time and space for strings of length m and n — you fill in the whole DP table. For short strings like movie titles that's fine. It's exactly *because* it's relatively expensive that you use q-gram indexing first: you want to run this on 20 candidates, not 100,000."

**Q: "How did you pick q=3?"**
> "It's a trade-off. Small q — say q=2 — means each trigram is common, so your candidate lists are huge and the filter barely filters. Large q — say q=5 — means a single typo destroys many overlapping grams at once, so you start missing genuine matches. q=3 is the standard sweet spot for word-length strings, and it's what I validated against my data."

**Q: "How do you rank the final results?"**
> "Primarily by prefix edit distance — lower is better. You can layer in other signals on top, like popularity or number of shared q-grams as a tiebreaker, so that between two equally-good string matches you show the more famous movie first."

**Q: "Why not just use Elasticsearch or a database LIKE query?"**
> "For production you probably would use Elasticsearch — no argument. The point of building it myself was to understand what's happening underneath. As for `LIKE '%shrek%'`, that doesn't handle typos at all, and it can't use an index for leading wildcards, so it degrades to a full table scan."

**Q: "How would you scale this to millions of entries?"**
> "The q-gram index is the part that grows — you'd want it distributed or backed by something like Redis rather than held in memory. You'd also prune very common q-grams, the same way search engines drop stopwords, because a q-gram that matches half your corpus isn't doing any filtering work and just costs you lookup time."

---

# 4. Deep Reinforcement Learning — Supply Chain Optimization
*Business Analytics Seminar, University of Freiburg · Python · Summer 2022*

## The 30-Second Version

"I formulated an inventory management problem as a reinforcement learning environment from scratch — defining the state space, the action space, and the reward function — then implemented and compared two algorithms, PPO and DQN, under stochastic demand. PPO reached about 79% of the theoretical optimal reward, DQN about 72%."

## The Problem Formulation — The Part That Matters Most

Interviewers care much more about *how you framed the problem* than about which library you called. Building the MDP yourself is the substantive work here.

**Why this is an RL problem at all:** you make a decision today (how much to order), the consequence arrives later (stock arrives after a lead time, demand materializes over days), and today's decision constrains tomorrow's options. That sequential, delayed-consequence structure is exactly what RL is for. A simple supervised model can't capture it because there's no labelled "correct order quantity" — you only find out later whether it worked.

**State space** — what the agent observes:
```
s = [current_inventory_level, recent_demand_history, outstanding_orders_in_transit]
```

**Action space** — what the agent can do:
```
a = how many units to reorder this period  (discrete: 0, 10, 20, ... N)
```

**Reward function** — this is the design decision that drives all behavior:
```
reward = revenue_from_sales
       − holding_cost × units_in_stock        (penalty for overstocking)
       − stockout_penalty × unmet_demand      (penalty for running out)
       − ordering_cost
```

**Why the reward design is the hard part:** it has to encode a genuine business trade-off. Penalize stockouts too heavily and the agent hoards inventory — never runs out, but holding costs destroy profit. Penalize holding too heavily and it runs lean and constantly disappoints customers. The interesting behavior only emerges if the two penalties are balanced realistically.

**Stochastic demand:** demand was sampled from both uniform and Poisson distributions. Poisson is the more realistic model for arrival-type events like customer orders. The randomness is what makes it hard — the agent can't memorize a fixed sequence, it has to learn a *policy* that's robust across many possible demand realizations.

## DQN vs PPO — Explain the Difference Clearly

**DQN (Deep Q-Network) — value-based.**
Learns a function Q(s, a) = "expected total future reward if I take action *a* in state *s*, then act optimally afterward." The policy is implicit: always take the action with the highest Q-value.

```
Q(inventory=50, order=0)   = 120
Q(inventory=50, order=20)  = 145   ← pick this one
Q(inventory=50, order=40)  = 110
```

**PPO (Proximal Policy Optimization) — policy-based.**
Learns the policy π(a|s) directly, as a probability distribution over actions. No Q-function in the middle.

```
π(a | inventory=50) = {order 0: 0.1, order 20: 0.7, order 40: 0.2}
```

**Why PPO won — the explanation to give:**
> "PPO's core mechanism is that it constrains how far the policy can move in a single update — it clips the update if the new policy diverges too much from the old one. That matters enormously with stochastic demand, because a run of unusually high demand can look like evidence that hoarding is correct, when it was just noise. DQN has no such constraint, so it's more prone to overreacting to a lucky or unlucky batch of experience and destabilizing. The gap I measured — 79% versus 72% of ideal — is consistent with that: PPO converged to a more stable policy."

**Also worth saying:** "DQN is designed for discrete action spaces, which mine was, so it was a legitimate choice — it's not that DQN was inappropriate, it's that PPO handled the noise better."

## Likely Questions & Answers

**Q: "What is the '79% of ideal reward' measured against?"**
> "A theoretical optimum computed with perfect hindsight — if you knew the exact demand sequence in advance, what's the best possible ordering policy? That's not achievable by any real agent, since it requires knowing the future, but it gives you a meaningful ceiling to measure against rather than just comparing two algorithms to each other."

**Q: "Why not just use a classical inventory model, like Economic Order Quantity or (s,S) policy?"**
> "Honestly, for a simple version of this problem, classical operations research methods are very strong and often better — they're analytically optimal under their assumptions, and they're interpretable. RL becomes interesting when the assumptions break: non-stationary demand, complex multi-echelon supply chains, constraints that don't fit the closed-form models. The seminar context was specifically about evaluating whether RL could handle the problem, so establishing that comparison honestly was part of the work."

**Q: "What was hardest?"**
> "Reward shaping. The first versions produced degenerate policies — an agent that just never orders anything, because ordering has an immediate cost and the stockout penalty wasn't weighted enough to counteract it. Getting the relative magnitudes of holding cost, stockout penalty, and revenue right so that sensible behavior emerged took real iteration."

**Q: "What's stable-baselines3 and did you implement the algorithms yourself?"**
> [BE HONEST] "stable-baselines3 is a library of well-tested RL algorithm implementations. I used it for PPO and DQN rather than reimplementing them — what I built from scratch was the environment itself: the state representation, the action space, the reward function, and the demand simulation, following the OpenAI Gym interface so the algorithms could plug into it."

**Q: "How do you know it actually learned rather than memorized?"**
> "The demand was stochastic and resampled each episode, so there's no fixed sequence to memorize. And I evaluated on episodes the agent hadn't trained on. A memorizing agent would fall apart under a new demand realization."

---

# 5. Event-Driven E-Commerce Platform
*Personal Project · Java/Kafka/Kubernetes · 2025*

## The 30-Second Version

"I built a six-service distributed e-commerce backend to go deep on event-driven architecture — services communicating through Kafka rather than direct synchronous calls. The two hard problems I focused on were exactly-once delivery semantics and distributed transactions using the Saga pattern. I deployed it on Kubernetes with Prometheus and Grafana for observability."

## Why Event-Driven At All

**Synchronous approach:**
```
Order Service  →  calls Inventory Service  →  calls Payment Service  →  calls Shipping Service
```
Problems: if Payment is down, the whole chain fails. Every service must know about the next. Latency accumulates. Tight coupling.

**Event-driven approach:**
```
Order Service  →  publishes "OrderCreated" to Kafka
                        ↓
        Inventory, Payment, Shipping all consume independently
```
Benefits: Order Service doesn't know who's listening. Services can be down and catch up later — the event is durably stored in the log. Adding a seventh service means subscribing to a topic, changing nothing upstream.

**Trade-off to acknowledge:** you gain decoupling and resilience, but you lose the simplicity of a single database transaction and immediate consistency. You now have to reason about eventual consistency and out-of-order or duplicate delivery. That's what the next two sections solve.

## Exactly-Once Semantics — The Real Explanation

**The three delivery guarantees:**

| Guarantee | Meaning | Failure mode |
|---|---|---|
| At-most-once | Fire and forget | Messages can be *lost* |
| At-least-once | Retry until acked | Messages can be *duplicated* |
| Exactly-once | Neither lost nor duplicated | Hardest; costs throughput |

**Why duplicates happen** (the scenario to describe):
```
1. Producer sends "ChargeCustomer €50" to Kafka
2. Kafka writes it successfully
3. The acknowledgement back to the producer is lost (network blip)
4. Producer times out, assumes failure, retries
5. Kafka now has the message TWICE
   → customer charged €100
```

**How Kafka actually achieves exactly-once — two mechanisms:**

**(a) Idempotent producer.** Each producer gets a Producer ID, and each message carries a monotonically increasing sequence number. The broker tracks the last sequence number it saw per producer per partition. If it receives a sequence number it has already written, it acknowledges but discards the duplicate.

```
Producer sends:  [PID=7, seq=42] "ChargeCustomer €50"     → broker writes it
Retry sends:     [PID=7, seq=42] "ChargeCustomer €50"     → broker sees seq 42 already written, drops it
```

**(b) Transactional writes.** The consume-process-produce cycle is wrapped in a transaction, so the offset commit and the output message are committed atomically:

```java
producer.beginTransaction();
// ... process the record, produce output ...
producer.sendOffsetsToTransaction(offsets, consumerGroupId);
producer.commitTransaction();
```

Without this, you can process a message, produce the result, then crash before committing the offset — so on restart you reprocess and produce the output twice. The transaction makes "I consumed this" and "I produced that" a single atomic fact.

**The honest caveat to mention:** "Exactly-once in Kafka means exactly-once *within Kafka's boundaries*. If your consumer writes to an external database, you still need idempotency on that write — typically an idempotency key — because Kafka's transaction can't span your database."

## The Saga Pattern — Worked Example

**The problem.** An order spans four services, each with its own database. You can't use a single ACID transaction across four databases.

**The Saga answer:** break it into a sequence of local transactions, each with a defined *compensating action* that undoes it.

**Happy path:**
```
1. Order Service      → create order (PENDING)
2. Inventory Service  → reserve 2 units
3. Payment Service    → charge €50
4. Shipping Service   → schedule delivery
5. Order Service      → mark order CONFIRMED
```

**Failure path — payment declines at step 3:**
```
1. Order created (PENDING)          ✓
2. Inventory reserved (2 units)     ✓
3. Payment charge                   ✗ DECLINED
   ↓ COMPENSATE, in reverse order
2'. Inventory Service → release the 2 reserved units
1'. Order Service     → mark order FAILED
```

**The crucial conceptual point:** *"A Saga doesn't give you rollback — it gives you semantic undo. The inventory reservation genuinely happened and was genuinely visible to other transactions for a moment. You can't un-happen it; you can only issue a compensating action that restores a correct end state. That's why the system is eventually consistent, not immediately consistent."*

**Two coordination styles** (know both):
- **Choreography:** each service listens for events and decides what to do. No central coordinator. Simple for short sagas, but hard to follow as it grows — the flow is implicit across services.
- **Orchestration:** a central coordinator tells each service what to do and tracks the state machine. Easier to reason about and debug, but the orchestrator is a component you have to build and operate.

## Likely Questions & Answers

**Q: "Why six services? Isn't that over-engineering for a personal project?"**
> "For a real product of that size, absolutely — a monolith would be the right call. I split it deliberately because the *point* was to hit the distributed-systems problems: partial failure, eventual consistency, distributed transactions. You don't encounter those in a monolith, and I wanted to solve them hands-on rather than just read about them."

**Q: "What happens if a compensating action itself fails?"**
> "That's the genuinely hard case. Compensating actions need to be retryable and idempotent, so you retry with backoff. If they keep failing, you need a dead-letter queue and, realistically, human intervention — an alert to an operator. There's no purely automatic answer; at some point a human has to reconcile it. Acknowledging that limit honestly is part of designing the system."

**Q: "What did you monitor with Prometheus and Grafana?"**
> "Consumer lag was the key one — how far behind the consumers are relative to the head of the log, which tells you immediately if a service is falling behind. Beyond that: message throughput per topic, processing latency per service, and error rates."

**Q: "How do you handle a message that repeatedly fails processing?"**
> "Retry with exponential backoff, then route it to a dead-letter topic after a threshold. The important thing is that one poison message must not block the partition — since Kafka delivers in order per partition, a message that fails forever would halt everything behind it."

---

# 6. File Upload & Data Retrieval Backend Service
*Personal Project · Python/FastAPI · 2026*

## The 30-Second Version

"A backend service for uploading and processing large CSV files asynchronously. FastAPI for the API layer, Celery with Redis for background job processing, PostgreSQL for storage. The interesting engineering problems were handling files too large to fit in memory, making the upload idempotent so re-uploading doesn't corrupt data, and tuning the job queue so jobs aren't lost when a worker dies."

## Problem 1: Large Files and Memory — Generator-Based Streaming

**The naive approach that breaks:**
```python
rows = pd.read_csv(uploaded_file)        # loads the ENTIRE file into RAM
for row in rows:
    process(row)
```
A 2 GB CSV → 2 GB+ of RAM (often several times more after parsing) → OOM kill.

**The streaming approach:**
```python
def read_rows(file):
    for line in file:              # reads one line at a time
        yield parse(line)          # yields it, doesn't accumulate

for row in read_rows(file):
    process(row)                   # constant memory, regardless of file size
```

**The key sentence:** *"A generator turns the memory cost from O(file size) into O(1) — you hold one row at a time instead of all of them. The file can be 50 GB and the process footprint doesn't change."*

## Problem 2: Idempotency — Upsert with ON CONFLICT

**The scenario:** a user uploads a file, the job fails halfway, they re-upload. Now half the rows already exist. A plain INSERT throws a primary key violation and the whole batch fails.

**The fix — PostgreSQL upsert:**
```sql
INSERT INTO products (sku, name, quantity)
VALUES ('ABC-123', 'Widget', 50)
ON CONFLICT (sku)
DO UPDATE SET
    name = EXCLUDED.name,
    quantity = EXCLUDED.quantity;
```

Now the operation is **idempotent** — running it once or five times produces the same final state. `EXCLUDED` refers to the row that *would have been* inserted.

**Why this matters beyond convenience:** in any system with retries — and every distributed system has retries — idempotency is what makes retrying *safe*. Without it, a retry is a data-corruption risk.

## Problem 3: Celery Reliability — acks_late and prefetch

**`acks_late=True`** — when does the worker acknowledge the task?

```
Default (acks_early):   worker takes task → ACKs immediately → processes
                        ↳ if worker crashes mid-processing, task is LOST

acks_late=True:         worker takes task → processes → ACKs on success
                        ↳ if worker crashes, task is unacked and gets REDELIVERED
```

The trade-off: `acks_late` means a crashed job gets retried (good), but it also means a job might run twice (hence the idempotency work above — they're connected design decisions).

**`prefetch_multiplier`** — how many tasks does a worker grab at once?

```
High prefetch (default 4):  worker grabs 4 tasks, holds 3 in local memory
                            ↳ if that worker dies, all 4 are stuck/delayed
                            ↳ bad for long-running tasks; causes uneven load

prefetch_multiplier=1:      worker takes exactly one task at a time
                            ↳ even distribution, fast failover
                            ↳ slightly more broker round-trips
```

**The explanation to give:** *"For short, fast tasks, high prefetch is more efficient — you amortize the broker round-trip. For long-running tasks like processing a large file, prefetch=1 is right, because a worker sitting on three queued jobs while slowly processing a fourth means those jobs are stuck behind it even if other workers are idle."*

## Problem 4: Data-Quality Audit Pipeline

After ingestion, a validation pass checks for:
- **Orphan rows** — a foreign key referencing something that doesn't exist (e.g. an order line pointing to a deleted product)
- **Negative quantities** — values that are structurally valid but semantically impossible

**Why this belongs in the system:** *"Bad data doesn't announce itself. If you don't check, you find out weeks later when a report is wrong and nobody can explain why. A cheap audit pass at ingestion time turns a silent corruption into an immediate, actionable error."*

## Likely Questions & Answers

**Q: "Why Celery instead of just processing in the request?"**
> "HTTP requests time out — typically 30 to 60 seconds at the gateway. Processing a large CSV takes minutes. So the request handler's job is to accept the file, enqueue a task, and immediately return a job ID. The client polls for status. That also means the work survives an API restart, since the task is in the queue, not in the request thread."

**Q: "Why Redis as the broker rather than RabbitMQ?"**
> "Redis is simpler to operate and fast, and it was sufficient for this scale. RabbitMQ gives you stronger delivery guarantees and richer routing, which matters more for critical workloads. For a project focused on the async processing patterns rather than broker semantics, Redis was the pragmatic choice."

**Q: "How does the client know when the job is done?"**
> "Polling a status endpoint with the job ID. The alternative is a webhook or WebSocket push, which avoids the polling overhead but adds complexity — you need to handle the client not being connected."

**Q: "What happens if two users upload the same file simultaneously?"**
> "The upsert handles the data-level race — last write wins, and the end state is consistent. If you needed to prevent duplicate *processing*, you'd add a lock keyed on a content hash of the file."

---

# 7. WatchDog — IoT Face Recognition Access Control
*Personal Project · Python/Raspberry Pi · Nov 2021*

## The 30-Second Version

"An access control system running end-to-end on a Raspberry Pi — a camera captures faces, an OpenCV recognition model decides whether the person is authorized, and on a positive match it drives a GPIO pin to trigger a physical lock. It's small, but it's genuinely full-stack in the hardware sense: sensor input through model inference to physical actuation."

## The Pipeline

```
Camera  →  Frame capture  →  Face detection  →  Face recognition  →  Decision  →  GPIO signal  →  Lock
```

**Step 1 — Detection vs. Recognition (know the difference).**
- *Detection* answers "is there a face in this frame, and where?" — typically Haar cascades or HOG
- *Recognition* answers "whose face is it?" — encode the face as a vector, compare against known encodings

They're separate stages; you detect first, then crop and recognize.

**Step 2 — Face encoding.** A face is converted into a fixed-length numeric vector (an embedding) that represents its features. Two photos of the same person produce vectors that are close together in that space; different people produce distant vectors.

**Step 3 — Matching.** Compare the live encoding against stored authorized encodings using a distance metric, with a threshold:
```
distance(live_face, authorized_face) < threshold  →  ACCESS GRANTED
```

**The threshold is the critical design decision:**
- Threshold too loose → false positives → **unauthorized people get in** (a security failure)
- Threshold too tight → false negatives → **authorized people get locked out** (a usability failure)

For a security application, you bias toward false negatives — it's better to occasionally reject a legitimate user than to occasionally admit an intruder.

**Step 4 — Hardware actuation.** A GPIO pin goes high, driving a relay that powers the lock mechanism. This is the part that makes it a hardware project rather than a software demo.

## Likely Questions & Answers

**Q: "What were the constraints of running this on a Raspberry Pi?"**
> "Compute, mainly. A Pi doesn't have a GPU, so inference is slow compared to a desktop. That shapes the design — you can't run recognition on every frame at 30fps. You detect on a downscaled frame, and only run the more expensive recognition step when a face is actually detected, not continuously."

**Q: "Is this secure enough for a real door?"**
> "Honestly, no — and it's worth being clear about that. A basic facial recognition system like this can be defeated with a photograph. A production system needs liveness detection — checking for depth, blinking, or using infrared — plus a secure enrollment process and a fallback authentication method. It's a solid demonstration of the pipeline, but I wouldn't put it on a real secure door as-is."

**Q: "What would you improve?"**
> "Liveness detection first, for the reason above. Then moving from classical OpenCV recognition to a modern deep-learning embedding model, which is substantially more accurate. And logging — every access attempt, successful or not, should be recorded, because for a security system the audit trail matters as much as the decision."

**Q: "How did you handle poor lighting?"**
> [ANSWER FROM YOUR EXPERIENCE — likely histogram equalization, or acknowledge it as a real limitation you encountered.]

---

# General Interview Advice for the Projects Section

**Structure every answer:** problem → approach → result → trade-off. The trade-off is what separates a senior answer from a junior one.

**Be honest about scope.** Several of these are lab exercises or personal projects, not production systems. Saying so directly builds credibility — and interviewers can usually tell anyway. "This was a seminar project, so the scale was modest, but the problem formulation was genuinely mine" is a strong answer.

**Know what you'd do differently.** For every project, have one clear answer to "what would you change?" It demonstrates that you've reflected rather than just shipped.

**Don't oversell.** If asked whether you implemented PPO yourself, say you used stable-baselines3 and built the environment. Getting caught inflating something small damages everything else you've said.

**Redirect to your strongest ground.** If a project question is going badly, it's legitimate to say: "That's the limit of what I did there — but a related problem I went much deeper on was [X]," and pivot to the graph algorithm or the Flybionic work.
